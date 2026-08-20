# Tracing / Logging / Metrics Integration for Drasi Server

* Project Drasi - Ruokun Niu (@ruokun-niu)
* Last edited on August 19th, 2026

## Overview

Drasi Server is a standalone single-process deployment of Drasi that embeds `drasi-lib`. Under the [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-overview.md) — covering [tracing](../../drasi-lib/tracing-logging/01-tracing.md) and [metrics](../../drasi-lib/tracing-logging/02-metrics.md) — drasi-lib will emit structured tracing spans and `metrics` crate counters/histograms/gauges through facade APIs. This design makes Drasi Server the embedding application that collects them: it wires up the tracing subscriber and metrics recorder so drasi-lib's telemetry flows to external backends (OTLP, Prometheus, stdout).

> **The three signals do not start from the same place, and the difference matters here.** Metrics are
> genuinely no-ops until a `Recorder` is installed. Spans are *created* but exported nowhere until a
> `tracing-opentelemetry` layer exists. **Logs already work**, because drasi-lib installs a subscriber
> of its own today — which is also why the ordering in §5 is a correctness requirement rather than a
> style preference. See
> [Enabling Telemetry as a drasi-lib Consumer](../../drasi-lib/tracing-logging/00-observability-overview.md#enabling-telemetry-as-a-drasi-lib-consumer).

Drasi Server does not manage or run telemetry backends for the user. It exports traces via OTLP (OpenTelemetry Protocol), which is accepted by most observability tools — Jaeger, Grafana Tempo, Datadog, Honeycomb, New Relic, AWS X-Ray, Azure Monitor, and others. The user points Drasi Server at any OTLP-compatible endpoint and runs their own backend.

## Terms and Definitions

| Term | Definition |
|------|------------|
| OTLP | OpenTelemetry Protocol — a standard for exporting traces and metrics to collectors (e.g., Jaeger, Grafana Tempo, Prometheus via OTLP receiver). |

See the [drasi-lib observability overview](../../drasi-lib/tracing-logging/00-observability-overview.md) for definitions of facade crate, span, subscriber, and recorder.

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

### Non-Goals

- Adding new tracing spans specific to Drasi Server's API layer (Axum routes, plugin management). This may be added later.
  > This is a **phasing statement, not a disagreement with drasi-lib.** drasi-lib traces its *own* control-plane operations (`control.*`) in Phase 1, because every embedder has a control plane whether or not a server sits in front of it. Drasi Server's HTTP-layer spans are Phase 2. The two docs describe different layers — see [01 — Tracing](../../drasi-lib/tracing-logging/01-tracing.md#control-plane-rooting--drasi-libs-own-api).
- Managing or running telemetry backends (Jaeger, Prometheus, OTLP collectors) on behalf of the user.

## Design Requirements

### Requirements

1. **Backward compatible**: Existing Drasi Server deployments with no telemetry config MUST continue to work — structured logs on stdout with the configured `logLevel`.
2. **Opt-in telemetry export**: OTLP tracing and metrics export MUST be opt-in via configuration. No external connections by default.
3. **Layered subscribers**: The `ComponentLogLayer` from drasi-lib MUST be composed into Drasi Server's subscriber alongside the OTLP and fmt layers.
   ⚠️ **"Coexist" is stronger than it sounds — today they cannot.** drasi-lib installs its own global subscriber via `let _ = set_global_default(...)`, so **first writer wins silently**: install after `DrasiLib` is built and Drasi Server's OTLP layer is ignored; install before and `ComponentLogLayer` never runs, emptying the component log streams that the REST API, CLI and VS Code extension read. Neither ordering yields both, which is why the [initializer split in LIB](../../drasi-lib/tracing-logging/00-observability-overview.md#api-design) is a **prerequisite** for this design rather than a cleanup.

### Dependencies

| Dependency | Version | Purpose | Notes |
|------------|---------|---------|-------|
| `drasi-lib` | current | Emits spans and metrics via facades | Existing dependency |
| `tracing` | `0.1` | Tracing facade | **Move from dev to production dependency** |
| `tracing-subscriber` | `0.3` | Subscriber composition (Registry, fmt, env-filter) | **New production dependency** |
| `tracing-opentelemetry` | current | Bridge tracing spans → OTLP | **New dependency** — see version note below |
| `opentelemetry` | current | OTLP trace/metrics API | **New dependency** — see version note below |
| `opentelemetry-otlp` | current | OTLP gRPC exporter | **New dependency** |
| `opentelemetry_sdk` | current | OTel runtime; also supplies `SpanData` for the plugin span bridge | **New dependency** |
| `metrics` | `0.23+` | Metrics facade — the recorder is installed against it | **New dependency, was missing from this table** |
| `metrics-util` | `0.20+` | `Stack`, `Fanout`, `FilterLayer`, `Registry` — the layered recorder stack | **New dependency, was missing from this table** |
| `metrics-exporter-prometheus` | `0.16+` | Prometheus rendering | **New dependency** |
| `metrics-process` | `2.4+` | Cross-platform process resource metrics (memory, CPU, fds, threads) via the `metrics` facade | **New dependency** |

> ⚠️ **Version note — do not copy the old pins.** This table previously specified `opentelemetry 0.20+`,
> `tracing-opentelemetry 0.21+` and `opentelemetry-otlp 0.13+`. The OTel Rust crates have moved a long
> way since (`opentelemetry` and `opentelemetry_sdk` are now at 0.32.x), and those floor versions read
> as recommendations rather than the minimums they were meant to be. Pin at integration time against
> whatever is current.
>
> **There is a real skew to reconcile**: `drasi-core/core/Cargo.toml` still declares
> `opentelemetry = "0.20"`. It is a **dead dependency** — zero `.rs` references — so it should simply
> be removed, but until it is, a naive workspace unification could drag the ancient version in.

### Out of Scope

- Server-specific spans for API endpoints, plugin loading, or config parsing.
- Changes to the existing REST API for component logs/events.
- Drasi Server web UI changes.

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
│     FilterLayer → Fanout → { Registry, exporter }                   │
│     → Prometheus: /metrics route on the existing API server         │
│     → OTLP: push to user-provided endpoint                          │
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
  filters:                                # optional targeted overrides on top of the profile
    include: ["drasi.index."]             # re-admit families the profile suppresses
    exclude: ["drasi.plugin."]            # suppress families the profile admits
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

> ⚠️ **The profile *mechanism* is Phase 1, even though the config surface is designed here.**
> Filtering has nothing to act on in Phase 0 — thirteen metrics and seven spans — and the
> `collection` flags have no Phase 0 consumers either, since backend engine statistics are Phase 2.
> So in Phase 0 only `off` and `basic` are meaningful, and they reduce to "install no recorder" and
> "install the recorder". `debug` and `persistence` become distinguishable when Phase 1 lands
> `FilterLayer` and the collection flags. The whole surface is specified now so that the config file
> does not change shape between phases, which is worth more than deferring the schema.

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

`filters` applies targeted adjustments on top of the profile so operators are not forced to step up a
whole rung to gain one metric family. It is two set operations on the deny list the profile resolves
to, applied before the `FilterLayer` is constructed:

| Key | Effect |
|---|---|
| `exclude` | Add patterns to the deny list — suppress something the profile admits |
| `include` | Remove patterns from the deny list — re-admit something the profile suppresses |

Patterns are **substrings**, not globs — `metrics_util::layers::FilterLayer` matches with an
Aho-Corasick automaton across the whole metric key. Write `drasi.index.`, not `drasi.index.*`; a
trailing `*` matches literally and therefore matches nothing.

Because both operations act on the recorder stack, they can only adjust signals that are **being
produced**. `include` cannot switch on collection that the profile left off — requesting engine
statistics under `profile: basic` needs the profile changed, not a filter added. **Drasi Server
validates this at startup and fails fast** rather than starting with a pattern that can never match,
which would otherwise present as telemetry silently missing.

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

- **`prometheus`** — serves a `/metrics` scrape endpoint **on the existing management API server**, not a separate listener. drasi-lib's recorder stack keeps a `metrics_util::registry::Registry` branch that can be read in-process, so the endpoint is an ordinary Axum route over structured values rather than a second HTTP server bound to its own port. `telemetry.metrics.prometheus.port` is therefore optional and exists only for deployments that deliberately want the scrape surface isolated from the API surface.
- **`otlp`** — pushes metrics to the configured OTLP endpoint at a configurable interval
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

**Naming convention**: These use the standard Prometheus `process_*` names rather than the `drasi.` prefix used by pipeline metrics. This is a deliberate, scoped rule: the `drasi.` prefix applies to Drasi-domain metrics (what the pipeline is doing — `drasi.query.events_processed`, `drasi.reaction.errors`), while process/host resource metrics follow their established ecosystem convention (`process_*`, what the OS process is consuming). This mirrors what mature stacks do — e.g., the OTel/Prometheus split between application metrics and `process_*`/host metrics — and means Drasi Server's resource metrics are recognized out-of-the-box by existing Grafana/Prometheus process dashboards and alert rules.

**Collection model**: `metrics-process` requires a periodic `collect()` call to refresh values. The collection point depends on the backend:
- **Prometheus** — call `collector.collect()` inside the `/metrics` scrape handler, so values are refreshed on-demand only when scraped (no idle cost).
- **OTLP** — spawn a lightweight background task that calls `collector.collect()` once per `exportInterval` before each push.

`processMetrics` requires a metrics backend (`prometheus` or `otlp`) to be configured; with `backend: none` there is no recorder to receive the values and the setting is a no-op. It defaults to `false`, so existing deployments are unaffected.

##### Container limits — what `process_*` cannot tell you

**`process_resident_memory_bytes` is not the number that gets you OOM-killed.** In a container the
kernel kills on the **cgroup**'s accounting, and cgroup v2 `memory.current` includes page cache and
kernel memory that RSS does not. A Drasi Server with modest RSS can be terminated while
`process_resident_memory_bytes` still looks healthy — so the process family alone cannot answer the
one question operators actually alert on: *how much headroom is left before the OOM killer fires?*

The same applies to CPU: `cpu.max` quota, not host core count, determines throttling.

When the files are present, Drasi Server reads them directly and emits:

| Metric | Source (cgroup v2) | Meaning |
|---|---|---|
| `drasi.server.cgroup.memory_limit_bytes` | `memory.max` | Hard limit; **absent** when the file reads `max` (unlimited) |
| `drasi.server.cgroup.memory_current_bytes` | `memory.current` | What the kernel actually counts against the limit |
| `drasi.server.cgroup.memory_utilization_ratio` | derived | `current / limit` — the headroom signal, and the thing to alert on |
| `drasi.server.cgroup.cpu_quota_cores` | `cpu.max` (quota ÷ period) | Effective core allowance; **absent** when quota reads `max` |
| `drasi.server.cgroup.cpu_throttled_seconds_total` | `cpu.stat` `throttled_usec` | Time spent throttled — evidence the quota is binding |

> **Naming decision.** These are deliberately **not** `container_*` and **not** `process_*`.
> `container_*` is cAdvisor's namespace, populated from *outside* the container with different labels
> — emitting the same names from inside would produce two families with identical names and
> incompatible semantics, which is exactly the collision the naming convention exists to prevent.
> `process_*` is process-scoped by ecosystem convention and these values are container-scoped.
> `drasi.server.*` is correct on the same reasoning that justifies the rest of that namespace: server
> is a functional domain, and this is *Drasi Server's own view of its container*.

**Degrade silently, never guess.** cgroup v2 is detected by the presence of
`/sys/fs/cgroup/cgroup.controllers`; v1 uses different paths and sentinels (`memory.limit_in_bytes`
with a huge value for unlimited, `cpu.cfs_quota_us` of `-1`). On macOS, Windows, or a
non-containerised Linux host the files are absent. In every one of those cases the metric is **not
emitted at all** — never zero and never a fabricated default, because a limit metric reading `0` or
`unlimited` incorrectly is worse than a missing series: it will silently satisfy an alert rule that
was supposed to fire.

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
5. Install metrics recorder — FilterLayer(metric_filters) → Fanout → { Registry,  (NEW)
   exporter }
6. Install process metrics collector — describe() + wire collect() into scrape/push (NEW)
7. Construct index providers, passing the storage flags from `collection`          (NEW)
8. Build and start DrasiLib instances, passing `collection` to the builder       (CHANGED)
9. Start Axum API server, mounting the /metrics route when the Prometheus         (CHANGED)
   backend is selected
```

**Steps 4 and 5 must precede step 8.** Because drasi-lib installs a subscriber of its own on the
first `DrasiLib` construction and the install silently no-ops if one already exists, building
`DrasiLib` first means Drasi Server's OTLP layer is discarded with no error — see Requirement 3.

**Step 7 is the one that is easy to drop.** The `collection` half of a resolved profile has to reach
the *index-provider constructors*, because backend engine statistics are fixed before drasi-lib is
handed the provider. Skipping it means `profile: persistence` silently produces no engine statistics.
This is the step §1's profile table forward-references.

#### 6. Shutdown

🐛 **This is new work, not an extension of an existing path.** Two defects were found while designing
the [crash-loss semantics](../../drasi-lib/tracing-logging/00-observability-overview.md#export-flush-and-crash-loss-semantics):

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

> **Note**: To compose `ComponentLogLayer` into Drasi Server's subscriber alongside the OTLP layer, drasi-lib's `get_or_init_global_registry()` is replaced by two functions (see [drasi-lib observability overview, API Design](../../drasi-lib/tracing-logging/00-observability-overview.md#api-design)):
> - `init_component_log_layer()` — returns the layer for Drasi Server to compose, installing nothing
> - `init_default_subscriber()` — composes and installs, for simple embedders
>
> Drasi Server calls `init_component_log_layer()` and composes the returned layer with fmt + OTLP into its own subscriber. The drasi-lib log worker thread starts when Drasi Server installs that subscriber, not when the layer is created.

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
- **New dependencies**: `tracing-opentelemetry`, `opentelemetry-otlp`, `opentelemetry_sdk`, `metrics`, `metrics-util`, `metrics-exporter-prometheus`, `metrics-process`. They are only active when configured.
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
| OTLP metrics | Integration | Configure OTLP metrics endpoint; verify metrics arrive at mock receiver |
| Process metrics | Integration | Enable `processMetrics` with Prometheus backend; `curl localhost:8080/metrics`; verify `process_resident_memory_bytes` and `process_cpu_seconds_total` appear with non-zero values |
| Env var override | Unit | Set `OTEL_ENDPOINT` env var; verify config resolves correctly |
| **SIGTERM handled** | Integration | Send `SIGTERM` (not `SIGINT`) and assert the process shuts down gracefully — this fails today |
| Shutdown flush | Integration | Send SIGTERM; verify pending spans are exported before exit |
| **Profile applied in full** | Integration | Set `profile: persistence` and assert RocksDB statistics actually appear — guards the failure where only the filter half of a profile is applied |
| **Unsatisfiable filter rejected** | Unit | `profile: basic` + `include: ["drasi.index.rocksdb."]` must fail at startup, not start with a pattern that can never match |
| Unreachable endpoint | Integration | Configure non-existent OTLP endpoint; verify server starts gracefully, logs warning |

## Development Plan

> The numbering below is **implementation sequence within Drasi Server**, unrelated to the telemetry
> Phase 0/1/2 delivery phases in
> [LIB — Phase Plan](../../drasi-lib/tracing-logging/00-observability-overview.md#phase-plan).

| Step | Work Items |
|-------|-----------|
| 1. Config types | Add `TelemetryConfig`, `TracingConfig`, `MetricsConfig`, `FilterConfig` structs to `config/types.rs` with serde deserialization + env var interpolation |
| 1a. Profile wiring | Resolve `TelemetryProfile` once at startup; apply all three outputs (directives, filters, collection) and **fail fast** on a filter that can never match |
| 2. Dependencies | Add `tracing-opentelemetry`, `opentelemetry-otlp`, `opentelemetry_sdk`, `metrics`, `metrics-util`, `metrics-exporter-prometheus`, `metrics-process` to `Cargo.toml` |
| 3. Builder API | Add `.with_telemetry_profile()`, `.with_tracing_endpoint()`, `.with_tracing_service_name()`, `.with_metrics_prometheus()`, `.with_metrics_otlp()`, `.with_process_metrics()` to `DrasiServerBuilder` |
| 4. Tracing setup | Implement `init_tracing()` — compose Registry with EnvFilter + fmt + ComponentLogLayer + optional OTLP. Depends on the drasi-lib `init_component_log_layer()` split |
| 5. Metrics setup | Implement `init_metrics()` — `FilterLayer` → `Fanout` → { Registry, exporter }; mount `/metrics` as a route on the existing API server |
| 5a. Process metrics | Install `metrics-process` `Collector` when `processMetrics` is enabled; wire `collect()` into the `/metrics` handler and/or the OTLP push interval |
| 6. `init` CLI | Extend `drasi-server init` with telemetry prompts (profile, OTLP endpoint, service name, metrics backend, process metrics) |
| 7. Shutdown | **Add a `SIGTERM` handler** — the server handles `SIGINT` only today — then sequence `DrasiLib::shutdown()` → telemetry flush → runtime teardown, with a bounded flush timeout (§6) |
| 8. Tests | Integration tests per backend config, graceful degradation, `SIGTERM`, and the profile-applied-in-full check |
| 9. Documentation | Update Drasi Server docs with telemetry configuration reference |

## Open Issues

1. ~~**Metrics port conflict**~~ — **RESOLVED: serve `/metrics` from the existing management API server.** The layered recorder stack keeps a `metrics_util::registry::Registry` branch alongside the exporter, so metrics are readable in-process as structured values and the scrape endpoint is just another Axum route. No second listener, no second port to open in a network policy, and no `PrometheusHandle::render()` string to re-parse. §3 above is written accordingly. See [02 — Metrics §2](../../drasi-lib/tracing-logging/02-metrics.md#2-collection-architecture).

2. ~~**Trace sampling**~~ — **RESOLVED elsewhere, deliberately.** The policy — head-based, decided at the source root, propagated on the W3C sampled bit already present in the FFI trace context, never applied to control-plane or bootstrap spans — is settled once in [01 — Tracing, Open Issue 3](../../drasi-lib/tracing-logging/01-tracing.md#open-issues). Drasi Server owns only the **rate**, and even that is normally set by `telemetry.profile` rather than configured per-deployment. A `samplingRate` override under `telemetry.tracing` is a Phase 1 addition; it must not become a second, competing definition of how sampling works.

### Future Consideration: Server-Level Metrics

Beyond drasi-lib's pipeline metrics (source events, query processing, reaction delivery) and the process resource metrics added by this design (`process_*` from `metrics-process`), a future iteration could add Drasi Server's own *application-level* operational metrics for remote monitoring and management — e.g., `drasi.server.uptime_seconds`, `drasi.server.sources_total` (by status), `drasi.server.api_requests_total`, `drasi.server.api_request_duration_seconds`, `drasi.server.config_saves_total`. These are distinct from process resource metrics: they describe application state and API traffic rather than OS-level resource usage. They would be recorded in Axum middleware and server lifecycle code (not in drasi-lib) and flow to whatever recorder the telemetry config installs. Names follow the [naming convention](../../drasi-lib/tracing-logging/00-observability-overview.md#naming-and-namespacing-conventions) — base units in the leaf, so durations are `_seconds` as `f64`, never `_ns`. This is not in scope for this design but is a natural next step once the telemetry infrastructure is in place.

## References

- [drasi-lib observability overview](../../drasi-lib/tracing-logging/00-observability-overview.md) — Prerequisite design: shared foundations
- [drasi-lib tracing design](../../drasi-lib/tracing-logging/01-tracing.md) — The spans Drasi Server exports
- [drasi-lib metrics design](../../drasi-lib/tracing-logging/02-metrics.md) — The metrics Drasi Server exports
- [drasi-platform query-host init_tracer() / init_metrics()](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Reference OTLP setup in Drasi for Kubernetes
- [Drasi Server repository](https://github.com/drasi-project/drasi-server) — Source repository
- [Drasi Server documentation](https://drasi.io/drasi-server/) — User-facing docs
