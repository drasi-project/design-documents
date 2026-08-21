# Tracing / Logging / Metrics Integration for Drasi Server

* Project Drasi - Ruokun Niu (@ruokun-niu)
* Last edited on August 19th, 2026

## Overview

This design makes Drasi Server the embedding application that collects them: it wires up the tracing subscriber and metrics recorder so drasi-lib's telemetry flows to external backends (OTLP, Prometheus, stdout).

Drasi Server does not manage or run telemetry backends for the user. It exports traces via OTLP (OpenTelemetry Protocol), which is accepted by most observability tools — Jaeger, Grafana Tempo, Datadog, Honeycomb, New Relic, AWS X-Ray, Azure Monitor, and others. The user points Drasi Server at any OTLP-compatible endpoint and runs their own backend.


## Objectives

### User Scenarios

1. **Developer running Drasi Server locally**: A developer starts Drasi Server with default config. Structured logs appear on stdout with span context. No extra setup needed.

2. **Operator exporting traces to Jaeger**: An operator runs Jaeger themselves and points Drasi Server at it via `telemetry.tracing.endpoint: "http://jaeger:4317"`. Drasi Server exports drasi-lib's end-to-end spans to Jaeger via OTLP.

3. **Operator scraping metrics with Prometheus**: An operator runs Prometheus themselves and enables the scrape endpoint in Drasi Server config. Drasi Server serves `/metrics` from the existing management API server, so no additional port has to be opened.

4. **Operator pushing metrics via OTLP**: An operator runs an OTLP collector and configures Drasi Server to push metrics to it.

5. **Operator monitoring process resource usage**: An operator wants to watch Drasi Server's own memory and CPU consumption to spot leaks or size the container. They enable process metrics and see `process_resident_memory_bytes`, `process_cpu_seconds_total`, `process_open_fds`, and `process_threads` alongside the pipeline metrics on the same Prometheus scrape endpoint (or OTLP stream).

### Goals

- Install a `tracing::Subscriber` that sends drasi-lib's spans to stdout and optionally to a user-provided OTLP endpoint
- Install a `metrics::Recorder` that exports drasi-lib's metrics via a Prometheus scrape endpoint or OTLP push to a user-provided endpoint
- Emit standard process resource metrics (resident/virtual memory, CPU time, open file descriptors, thread count) for the Drasi Server process through the same recorder
- Make telemetry configuration available via YAML config and environment variable overrides
- Preserve the existing `ComponentLogLayer` and per-component log streaming API endpoints
- Preserve the existing `logLevel` configuration


## Design

### High-Level Design

Drasi Server currently initializes logging by setting `RUST_LOG` from the YAML config's `logLevel` field and calling `get_or_init_global_registry()` from drasi-lib. This design adds an optional `telemetry` config section. When present, Drasi Server builds the appropriate OTLP tracing layer and/or metrics recorder at startup and connects to the user-provided endpoints. The user is responsible for running their own backends (Jaeger, Prometheus, OTLP collector).

```
┌─────────────────────────────────────────────────────────────────────┐
│  Drasi Server (main.rs)                                             │
│                                                                     │
│  Startup:                                                           │
│                                                                     │
│  1. Load config                                                     │
│  2. Build tracing subscriber:                                       │
│     ┌─ Registry ────────────────────────────────────────────────┐   │
│     │  ┌─────────────────┐  ┌────────────────────────────────┐  │   │
│     │  │ ComponentLogLayer│  │ fmt layer (stdout, built-in)   │  │   │
│     │  │ (from drasi-lib) │  │ (filtered by logLevel)         │  │   │
│     │  └─────────────────┘  └────────────────────────────────┘  │   │
│     │  ┌────────────────────────────────────────────────────┐   │   │
│     │  │ OTLP layer (if telemetry.tracing.endpoint set)     │   │   │
│     │  │ → connects to user-provided Jaeger/OTLP collector  │   │   │
│     │  └────────────────────────────────────────────────────┘   │   │
│     └───────────────────────────────────────────────────────────┘   │
│                                                                     │
│  3. Install metrics recorder (if telemetry.metrics configured):     │
│     FilterLayer → selected backend recorder                         │
│     → PrometheusRecorder + Handle.render() at /metrics              │
│     → OpenTelemetryRecorder + PeriodicReader → OTLP                 │
│                                                                     │
│  4. Start DrasiLib instances (unchanged)                            │
│     → drasi-lib emits spans + metrics through facades               │
│     → data flows to installed subscriber + recorder                 │
│                                                                     │
│  5. On shutdown (SIGTERM/SIGINT): stop DrasiLib, THEN flush         │
│     telemetry, THEN tear down the runtime — see §6 (NEW WORK)       │
└─────────────────────────────────────────────────────────────────────┘
```

### Detailed Design

#### 1. Configuration

The `DrasiServerConfig` is extended with an optional `telemetry` section:

```yaml
# config/server.yaml
apiVersion: drasi.io/v1
id: my-server
host: 0.0.0.0
port: 8080
logLevel: info

# NEW: optional telemetry section
telemetry:
  profile: basic                          # off | basic | debug | persistence
  filters:                                # optional additional exclusions
    exclude: ["drasi.plugin."]            # suppress matching metric families
  tracing:
    endpoint: "http://jaeger:4317"       # gRPC OTLP endpoint; omit to disable
    serviceName: "drasi-server"           # OTel service.name resource attribute
  metrics:
    backend: prometheus                   # "prometheus", "otlp", or "none" (or omit `metrics` to disable)
    processMetrics: true                  # emit process resource metrics (memory/CPU/fds/threads); default false
    prometheus:
      port: 9090                          # OPTIONAL — omit to serve /metrics from the API server (default)
      path: "/metrics"                    # Scrape path
    otlp:
      endpoint: "http://otel-collector:4317"
      exportInterval: 30                  # Export interval in seconds
```

##### Profiles

`profile` selects a preset instead of requiring every signal to be configured individually. The
presets, the ladder they form, and the reasoning behind them are defined once in
[LIB — Enablement, Filtering and Observability Profiles](../../drasi-lib/tracing-logging/00-observability-overview.md#enablement-filtering-and-observability-profiles);
Drasi Server does not define its own vocabulary. Defaults to `basic` when the `telemetry` section is
present, and `off` when it is absent.

**A profile is not only a filter list, and Drasi Server is the component that makes that true.**
Resolving a profile yields three outputs, and the server must apply all three or the profile is
silently half-honoured:

| Output | Where Drasi Server applies it |
|---|---|
| `trace_directives` | The `EnvFilter` composed in §2 |
| `metric_filters` | A `FilterLayer` at the top of the recorder stack in §3 |
| `collection` | Passed to `DrasiLibBuilder`, **and** to the index-provider constructors in §5 |

The last one is the one that is easy to miss. Backend engine statistics — RocksDB `Statistics` is the
worked example — must be enabled in the provider's own constructor, which runs *before* drasi-lib is
handed the provider. Drasi Server is able to honour `profile: persistence` only because it owns both
the profile and that constructor call; a library embedder that builds its own providers does not. See
the ownership rule in LIB for why this is structural rather than an oversight.

##### Filters

`filters.exclude` adds metric-name patterns to the deny list produced by the selected profile:

| Key | Effect |
|---|---|
| `exclude` | Add patterns to the deny list — suppress something the profile admits |

Patterns are **substrings**, not globs — `metrics_util::layers::FilterLayer` matches with an
Aho-Corasick automaton across the whole metric key. Write `drasi.index.`, not `drasi.index.*`; a
trailing `*` matches literally and therefore matches nothing.

```yaml
telemetry:
  profile: persistence
  filters:
    exclude: ["drasi.plugin."] # suppress all plugin metrics
```

Filters can only suppress metrics. To enable metric families or engine statistics that a profile
does not collect, select a profile that includes them.

All fields support environment variable interpolation:

```yaml
telemetry:
  profile: "${DRASI_TELEMETRY_PROFILE:-basic}"
  tracing:
    endpoint: "${OTEL_ENDPOINT:-}"
  metrics:
    backend: "${METRICS_BACKEND:-prometheus}"
    prometheus:
      port: "${METRICS_PORT:-9090}"
```

When the `telemetry` section is omitted, Drasi Server behaves exactly as today — stdout logs at the configured `logLevel`, no external connections.

#### 2. Tracing Subscriber Setup

On startup, Drasi Server composes a multi-layer tracing subscriber with three layers:

1. **ComponentLogLayer** (from drasi-lib) — routes log events to per-component streams, unchanged
2. **fmt layer** — writes structured logs to stdout, filtered by `logLevel`
3. **OTLP layer** (optional) — exports spans to the user-provided OTLP endpoint via `tracing-opentelemetry`. Only created when `telemetry.tracing.endpoint` is configured.

If no OTLP endpoint is configured, only layers 1 and 2 are active — identical to current behavior.

#### 3. Metrics Recorder Setup

On startup, Drasi Server installs a metrics recorder based on the `telemetry.metrics.backend` config:

- **`prometheus`** — build a `metrics_exporter_prometheus::PrometheusRecorder`, retain its
  `PrometheusHandle`, and wrap the recorder in the profile's `FilterLayer` before installing it
  globally. The `/metrics` handler calls `PrometheusHandle::render()`. No separate
  `metrics_util::Registry` is required. `telemetry.metrics.prometheus.port` remains optional for
  deployments that deliberately isolate the scrape surface.
- **`otlp`** — configure an OpenTelemetry `PeriodicReader` with an
  `opentelemetry_otlp::MetricExporter`, add it to an `SdkMeterProvider`, and pass the provider's
  `Meter` to `metrics_exporter_otel::OpenTelemetryRecorder`. Wrap that recorder in the profile's
  `FilterLayer` and install it globally. Retain the `SdkMeterProvider` for bounded flush and
  shutdown. `metrics-exporter-otel` is the concrete adapter from the `metrics` facade to the
  OpenTelemetry metrics API; `opentelemetry_sdk` alone does not implement `metrics::Recorder`.
- **`none`** (or the `metrics` section omitted) — no recorder installed, `metrics` facade calls are no-ops. Tracing can still be enabled independently via `telemetry.tracing`.

#### 4. Process Resource Metrics


When `telemetry.metrics.processMetrics: true`, Drasi Server installs a [`metrics-process`](https://crates.io/crates/metrics-process) `Collector`. This crate is the established, cross-platform (Linux, macOS, Windows, FreeBSD) way to emit Prometheus-standard process metrics through the `metrics` facade. Because it records through the same global `metrics::Recorder` already installed in §3, the process metrics flow to **whatever backend is configured** — Prometheus scrape or OTLP push — with no separate exporter.

The collector emits the standard `process_*` metric family:

| Metric | Type | Meaning |
|--------|------|---------|
| `process_resident_memory_bytes` | gauge | Resident set size (physical memory) |
| `process_virtual_memory_bytes` | gauge | Virtual memory size |
| `process_cpu_seconds_total` | counter | Total user + system CPU time |
| `process_open_fds` | gauge | Open file descriptors |
| `process_max_fds` | gauge | File descriptor limit |
| `process_threads` | gauge | OS thread count |
| `process_start_time_seconds` | gauge | Process start time (Unix epoch) |

Availability of individual metrics varies by platform (e.g., `process_open_fds` is not available on Windows); `metrics-process` handles this per-OS and simply omits unsupported metrics.


**Collection model**: `metrics-process` requires a periodic `collect()` call to refresh values. The collection point depends on the backend:
- **Prometheus** — call `collector.collect()` inside the `/metrics` scrape handler, so values are refreshed on-demand only when scraped (no idle cost).
- **OTLP** — spawn a lightweight background task that calls `collector.collect()` once per `exportInterval` before each push.

`processMetrics` requires a metrics backend (`prometheus` or `otlp`) to be configured; with `backend: none` there is no recorder to receive the values and the setting is a no-op. It defaults to `false`, so existing deployments are unaffected.


#### 5. Startup Order

```
1. Load config (existing)
2. Resolve the telemetry profile once — TelemetryProfile::resolve() yields         (NEW)
   trace_directives + metric_filters + collection; all three must be consumed
3. init_component_log_layer() — create the ComponentLogRegistry + `ComponentLogLayer`
   (CHANGED: get_or_init_global_registry() is removed; this installs nothing)
4. Build and install tracing subscriber — EnvFilter(trace_directives) +            (NEW)
   `ComponentLogLayer` + fmt + optional OTLP, composed into one Registry
   (drasi-lib's log worker thread starts here, on Dispatch construction)
5. Build the selected metrics backend, wrap its recorder in                     (NEW)
  FilterLayer(metric_filters), and install it globally. Retain either the
  PrometheusHandle or the OTLP SdkMeterProvider.
6. Install process metrics collector — describe() + wire collect() into scrape/push (NEW)
7. Construct index providers, passing the storage flags from `collection`          (NEW)
8. Build and start DrasiLib instances, passing `collection` to the builder       (CHANGED)
9. Start Axum API server, mounting the /metrics route when the Prometheus         (CHANGED)
   backend is selected
```

#### 6. Shutdown

1. **`SIGTERM` is not handled.** The shutdown path is `tokio::signal::ctrl_c()` (`src/server.rs:825`),
   which is **`SIGINT` only** — there are no occurrences of `SignalKind`, `signal::unix` or
   `terminate` anywhere in the server. Kubernetes terminates pods with `SIGTERM`, so **the deployment
   target that matters has no graceful shutdown path at all**; the process dies on the default
   disposition, running no destructors.
2. **Nothing flushes telemetry even on the `SIGINT` path.** `DrasiLib::shutdown()` stops components,
   aborts the graph-update task and releases index handles. It drains no queue and flushes no
   exporter.

The required sequence, and the ordering is load-bearing:

| Step | Why this order |
|---|---|
| 1. Await **either** `SIGINT` or `SIGTERM` | `ctrl_c()` alone is a developer-laptop path |
| 2. `DrasiLib::shutdown()` | Component-stop logs and final metric values must be *recorded* first |
| 3. Flush telemetry — tracer provider `shutdown()`/`force_flush()`, metrics provider on the OTLP branch | Flushing before step 2 discards exactly the shutdown diagnostics you wanted |
| 4. Tear down the tokio runtime | Dropping it before the flush completes loses what the flush was for |
| 5. Bound step 3 with the export timeout | A dead collector must not hang termination past the orchestrator's grace period — after which `SIGKILL` makes it moot anyway |

Under the Prometheus **scrape** branch there is no metrics flush to perform at all: the scrape reads
current values, so metrics have no unexported window. Steps 3–5 concern traces, and logs if an OTLP
log layer is composed.

### API Design

#### Builder API

`DrasiServerBuilder` is extended with telemetry configuration methods:

```rust
let server = DrasiServerBuilder::new()
    .with_id("my-server")
    .with_host_port("0.0.0.0", 8080)
    // NEW: telemetry configuration
    .with_telemetry_profile(TelemetryProfile::Basic)   // off | basic | debug | persistence
    .with_tracing_endpoint("http://jaeger:4317")
    .with_tracing_service_name("drasi-server")
    .with_metrics_prometheus()            // /metrics on the API server; add (port, path) to isolate
    // or: .with_metrics_otlp("http://otel-collector:4317", 30)
    .with_process_metrics(true)           // NEW: emit process_* resource metrics
    .with_source(my_source)
    .add_query(query)
    .with_reaction(my_reaction)
    .build()
    .await?;
```

These are convenience methods that populate the `TelemetryConfig` struct. All are optional — omitting them gives current behavior (stdout logs, no external telemetry).

#### `init` CLI Command

The `drasi-server init` command generates a YAML config interactively. It is extended with telemetry prompts:

```
$ drasi-server init --output config/server.yaml

  Server ID [auto]: my-server
  Host [0.0.0.0]:
  Port [8080]:
  Log level [info]:

  Enable telemetry? [y/N]: y
    Observability profile (off/basic/debug/persistence) [basic]:
    OTLP tracing endpoint (blank to skip): http://jaeger:4317
    Service name [drasi-server]:
    Metrics backend (prometheus/otlp/none) [none]: prometheus
    Serve /metrics from the API server? [Y/n]: y
    Collect process resource metrics (memory/CPU)? [y/N]: y

  ...
```

When telemetry is skipped, the `telemetry` section is omitted from the generated YAML — identical to current behavior.

No changes to Drasi Server's existing REST API endpoints. The Prometheus scrape endpoint is served as an additional route on the **existing management API server** rather than a separate listener — see §3 and [Open Issue 1](#open-issues).

### Alternatives Considered

#### 1. Telemetry Backends as Dynamic Plugins

Load telemetry backends as dynamically loaded shared libraries (`libdrasi_telemetry_*`), following the same OCI-based plugin model used for sources and reactions.

**Rejected because**: The `tracing_subscriber::Layer` trait uses generics and is not FFI-safe across shared library boundaries — this is a significant technical challenge with no clean solution. The binary size savings are minimal (OTLP + Prometheus add a few MB). The plugin approach adds complexity (new trait, C ABI bridge, OCI packaging, plugin loader changes) for limited benefit. Statically compiling the supported backends is simpler and sufficient.

#### 2. Let Users Configure Telemetry Entirely Outside Drasi Server

Rely on `RUST_LOG` and OpenTelemetry's auto-instrumentation environment variables without any Drasi Server config.

**Rejected because**: OpenTelemetry's env var auto-configuration requires the application to opt into it with code. There's no way to auto-install a `tracing-opentelemetry` layer or a `metrics::Recorder` just from env vars — the setup code must exist in the binary. The YAML config makes telemetry a first-class, documented feature.

#### 3. Hard-Code OTLP as the Only Export Backend

Always export to OTLP, require users to run an OpenTelemetry Collector to fan out to Prometheus/Jaeger/etc.

**Rejected because**: For simple deployments (single Docker container), requiring an OTel Collector just to get Prometheus metrics is heavy. Supporting both Prometheus scrape and OTLP push directly lets users choose the simpler option.

## Security

- **OTLP endpoint**: The OTLP exporter connects to a user-configured endpoint. If the endpoint is remote, users should use TLS. Drasi Server does not enforce TLS — this matches the pattern used by drasi-platform's query-host.
- **Prometheus endpoint — changed by the port decision.** Serving `/metrics` from the management API server means the endpoint **inherits the API server's exposure and access control** rather than having its own. That is an improvement where the API is already protected, but it has a corollary worth stating: metrics become reachable everywhere the management API is, and a scraper must be able to satisfy whatever authentication the API requires. Operators who need the two surfaces separated should set `telemetry.metrics.prometheus.port`, which is exactly why that field is retained.
- **Metrics content**: The scrape output exposes operational data only — component IDs, event counts, latency. No user data, no credentials.
- ⚠️ **Backend identity labels must be sanitised.** The `backend_instance` label proposed for `drasi.index.*` metrics derives from storage connection configuration, which can embed credentials. It must be reduced to host:port (or a path, or `inline`) before it becomes a label, and that reduction needs an explicit test — a metric label is exported verbatim and is not redacted anywhere downstream.
- ⚠️ **Deferred to Phase 2, recorded now: API-layer spans carry a secret-leak risk.** Management payloads contain connection strings, credentials and query text, and span attributes export verbatim. When server API spans are added they need an explicit **allow-list** (`query_id`, `component_type`, status, duration) — never the request body. This risk does not arise in drasi-lib, whose control-plane operations take typed config rather than raw payloads.
- **Process metrics**: The `process_*` metrics expose only OS-level resource counters (memory, CPU, fd/thread counts) for the Drasi Server process. They contain no user data and are gated behind the same exposure caveat above.

## Compatibility Impact

- **No breaking changes for Drasi Server operators**: existing configs without a `telemetry` section work exactly as before, and no REST endpoint changes shape.
- **New dependencies**: `tracing-opentelemetry`, `opentelemetry-otlp`, `opentelemetry_sdk`, `metrics`, `metrics-util`, `metrics-exporter-prometheus`, `metrics-exporter-otel`, `metrics-process`. They are only active when configured.
- **The drasi-lib initializer split is a prerequisite, not a nicety.** Earlier drafts called this "a minor refactor". It is the only way to have OTLP export and working component log streams simultaneously — without it one of the two is silently lost, with no error on either branch. See Requirement 3.
  > It is also a **breaking change to drasi-lib's public API**: `get_or_init_global_registry()` is removed rather than aliased. That breaks *other* drasi-lib embedders, not Drasi Server — Drasi Server is being changed here anyway. Rationale in [LIB — API Design](../../drasi-lib/tracing-logging/00-observability-overview.md#api-design).
- **`SIGTERM` handling is new.** The server currently handles `SIGINT` only, so adding graceful shutdown is a behaviour change in its own right — a container that previously died abruptly on pod termination will now stop components and flush first.

## Supportability

### Quickstart: seeing the telemetry

A recurring review concern was being *"trapped behind some tool"* — telemetry that exists but is
awkward to look at. Logs, traces and metrics conventionally need three different viewers, so the
design deliberately supports a path at each level of effort, and **the cheapest one needs no
infrastructure at all**.

| Level | You install | You see |
|---|---|---|
| **0 — nothing** | Nothing. Omit the `telemetry` section | Structured logs on stdout with span context. `RUST_LOG=drasi_lib=debug` for more |
| **1 — metrics, no backend** | Nothing. `metrics.backend: prometheus` | `curl localhost:8080/metrics` — the full Prometheus text format, readable as-is. No Prometheus server needed to *look* at metrics |
| **2 — traces** | One container: Jaeger all-in-one, which accepts OTLP directly | Set `tracing.endpoint` to its OTLP port; the end-to-end waterfall appears in Jaeger's UI |
| **3 — everything** | An OTel Collector plus a viewer, via compose | Traces and metrics in one stack |

Level 2 is the one worth highlighting: because Jaeger accepts OTLP natively, **seeing a full
distributed trace costs exactly one container and one config line** — no collector in between.

> ⚠️ **A concrete compose file belongs in the repo, not in this design.** Pinned images and ports go
> stale faster than a design document is revised, and an untested snippet here would be worse than
> none. The commitment this design makes is that levels 0–2 each work with no Drasi-side code beyond
> what is specified above; the quickstart artefact itself is a documentation deliverable (step 9).

**What we verify**, so the claim is not aspirational: the integration tests in the table below cover
levels 0, 1 and 2 — default config produces logs, the scrape endpoint returns parseable Prometheus
text, and spans arrive at a mock OTLP receiver.

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Default config (no telemetry) | Integration | Start server with no `telemetry` section; verify stdout logs appear, no crashes |
| OTLP tracing | Integration | Configure endpoint to mock OTLP receiver; verify spans arrive |
| Prometheus scrape | Integration | Enable Prometheus backend; `curl localhost:8080/metrics` on the **API port**; verify drasi-lib metrics appear |
| OTLP metrics | Integration | Configure OTLP metrics endpoint; verify a counter, gauge, and histogram arrive at a mock receiver with attributes and units intact |
| Process metrics | Integration | Enable `processMetrics` with Prometheus backend; `curl localhost:8080/metrics`; verify `process_resident_memory_bytes` and `process_cpu_seconds_total` appear with non-zero values |
| Env var override | Unit | Set `OTEL_ENDPOINT` env var; verify config resolves correctly |
| **SIGTERM handled** | Integration | Send `SIGTERM` (not `SIGINT`) and assert the process shuts down gracefully — this fails today |
| Shutdown flush | Integration | Send SIGTERM; verify pending spans are exported before exit |
| **Profile applied in full** | Integration | Set `profile: persistence` and assert RocksDB statistics actually appear — guards the failure where only the filter half of a profile is applied |
| Unreachable endpoint | Integration | Configure non-existent OTLP endpoint; verify server starts gracefully, logs warning |

## Development Plan

> The numbering below is **implementation sequence within Drasi Server**, unrelated to the telemetry
> Phase 0/1/2 delivery phases in
> [LIB — Phase Plan](../../drasi-lib/tracing-logging/00-observability-overview.md#phase-plan).

| Step | Work Items |
|-------|-----------|
| 1. Config types | Add `TelemetryConfig`, `TracingConfig`, `MetricsConfig`, `FilterConfig` structs to `config/types.rs` with serde deserialization + env var interpolation |
| 1a. Profile wiring | Resolve `TelemetryProfile` once at startup and apply all three outputs: directives, filters, and collection flags |
| 2. Dependencies | Add `tracing-opentelemetry`, `opentelemetry-otlp`, `opentelemetry_sdk`, `metrics`, `metrics-util`, `metrics-exporter-prometheus`, `metrics-exporter-otel`, `metrics-process` to `Cargo.toml` |
| 3. Builder API | Add `.with_telemetry_profile()`, `.with_tracing_endpoint()`, `.with_tracing_service_name()`, `.with_metrics_prometheus()`, `.with_metrics_otlp()`, `.with_process_metrics()` to `DrasiServerBuilder` |
| 4. Tracing setup | Implement `init_tracing()` — compose Registry with EnvFilter + fmt + ComponentLogLayer + optional OTLP. Depends on the drasi-lib `init_component_log_layer()` split |
| 5. Metrics setup | Implement `init_metrics()` with `PrometheusRecorder` + retained `PrometheusHandle`, or `OpenTelemetryRecorder` + retained `SdkMeterProvider`; wrap the selected recorder in `FilterLayer` and mount `/metrics` for Prometheus |
| 5a. Process metrics | Install `metrics-process` `Collector` when `processMetrics` is enabled; wire `collect()` into the `/metrics` handler and/or the OTLP push interval |
| 6. `init` CLI | Extend `drasi-server init` with telemetry prompts (profile, OTLP endpoint, service name, metrics backend, process metrics) |
| 7. Shutdown | **Add a `SIGTERM` handler** — the server handles `SIGINT` only today — then sequence `DrasiLib::shutdown()` → telemetry flush → runtime teardown, with a bounded flush timeout (§6) |
| 8. Tests | Integration tests per backend config, graceful degradation, `SIGTERM`, and the profile-applied-in-full check |
| 9. Documentation | Update Drasi Server docs with telemetry configuration reference |

## Appendix — Future Server-Level Metrics

Beyond drasi-lib's pipeline metrics (source events, query processing, reaction delivery) and the process resource metrics added by this design (`process_*` from `metrics-process`), a future iteration could add Drasi Server's own *application-level* operational metrics for remote monitoring and management — e.g., `drasi.server.uptime` (unit `s`), `drasi.server.sources` (by status), `drasi.server.api.requests`, `drasi.server.api.request.duration` (unit `s`), and `drasi.server.config.saves`. These are distinct from process resource metrics: they describe application state and API traffic rather than OS-level resource usage. They would be recorded in Axum middleware and server lifecycle code (not in drasi-lib) and flow to whatever recorder the telemetry config installs. Names follow the [OpenTelemetry naming convention](../../drasi-lib/tracing-logging/00-observability-overview.md#naming-and-namespacing-conventions), with units and instrument type carried as metadata. This is not in scope for this design but is a natural next step once the telemetry infrastructure is in place.

## References

- [drasi-lib observability overview](../../drasi-lib/tracing-logging/00-observability-overview.md) — Prerequisite design: shared foundations
- [drasi-lib tracing design](../../drasi-lib/tracing-logging/01-tracing.md) — The spans Drasi Server exports
- [drasi-lib metrics design](../../drasi-lib/tracing-logging/02-metrics.md) — The metrics Drasi Server exports
- [drasi-platform query-host init_tracer() / init_metrics()](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Reference OTLP setup in Drasi for Kubernetes
- [Drasi Server repository](https://github.com/drasi-project/drasi-server) — Source repository
- [Drasi Server documentation](https://drasi.io/drasi-server/) — User-facing docs
