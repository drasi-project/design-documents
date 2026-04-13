# Tracing / Logging / Metrics Integration for Drasi Server

* Project Drasi - April 12, 2026 - Ruokun Niu (@ruokun-niu)

## Overview

Drasi Server is a standalone single-process deployment of Drasi that embeds `drasi-lib`. With the [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md), drasi-lib now emits structured tracing spans and `metrics` crate counters/histograms/gauges through facade APIs — but those are no-ops until the embedding application installs a backend. This design makes Drasi Server that embedding application: it wires up the tracing subscriber and metrics recorder so that drasi-lib's telemetry flows to external backends (OTLP, Prometheus, stdout).

Drasi Server does not manage or run telemetry backends for the user. It exports traces via OTLP (OpenTelemetry Protocol), which is accepted by most observability tools — Jaeger, Grafana Tempo, Datadog, Honeycomb, New Relic, AWS X-Ray, Azure Monitor, and others. The user points Drasi Server at any OTLP-compatible endpoint and runs their own backend.

## Terms and Definitions

| Term | Definition |
|------|------------|
| OTLP | OpenTelemetry Protocol — a standard for exporting traces and metrics to collectors (e.g., Jaeger, Grafana Tempo, Prometheus via OTLP receiver). |

See the [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md) for definitions of facade crate, span, subscriber, and recorder.

## Objectives

### User Scenarios

1. **Developer running Drasi Server locally**: A developer starts Drasi Server with default config. Structured logs appear on stdout with span context. No extra setup needed.

2. **Operator exporting traces to Jaeger**: An operator runs Jaeger themselves and points Drasi Server at it via `telemetry.tracing.endpoint: "http://jaeger:4317"`. Drasi Server exports drasi-lib's end-to-end spans to Jaeger via OTLP.

3. **Operator scraping metrics with Prometheus**: An operator runs Prometheus themselves and enables the scrape endpoint in Drasi Server config. Drasi Server exposes `/metrics` on a configurable port.

4. **Operator pushing metrics via OTLP**: An operator runs an OTLP collector and configures Drasi Server to push metrics to it.

### Goals

- Install a `tracing::Subscriber` that sends drasi-lib's spans to stdout and optionally to a user-provided OTLP endpoint
- Install a `metrics::Recorder` that exports drasi-lib's metrics via a Prometheus scrape endpoint or OTLP push to a user-provided endpoint
- Make telemetry configuration available via YAML config and environment variable overrides
- Preserve the existing `ComponentLogLayer` and per-component log streaming API endpoints
- Preserve the existing `logLevel` configuration

### Non-Goals

- Adding new tracing spans specific to Drasi Server's API layer (Axum routes, plugin management). This may be added later.
- Managing or running telemetry backends (Jaeger, Prometheus, OTLP collectors) on behalf of the user.

## Design Requirements

### Requirements

1. **Backward compatible**: Existing Drasi Server deployments with no telemetry config MUST continue to work — structured logs on stdout with the configured `logLevel`.
2. **Opt-in telemetry export**: OTLP tracing and metrics export MUST be opt-in via configuration. No external connections by default.
3. **Layered subscribers**: The `ComponentLogLayer` (installed by drasi-lib's `get_or_init_global_registry()`) MUST coexist with the OTLP and fmt layers.

### Dependencies

| Dependency | Version | Purpose | Notes |
|------------|---------|---------|-------|
| `drasi-lib` | current | Emits spans and metrics via facades | Existing dependency |
| `tracing` | `0.1` | Tracing facade | **Move from dev to production dependency** |
| `tracing-subscriber` | `0.3` | Subscriber composition (Registry, fmt, env-filter) | **New production dependency** |
| `tracing-opentelemetry` | `0.21+` | Bridge tracing spans → OTLP | **New dependency** |
| `opentelemetry` | `0.20+` | OTLP trace/metrics SDK | **New dependency** |
| `opentelemetry-otlp` | `0.13+` | OTLP gRPC exporter | **New dependency** |
| `opentelemetry_sdk` | `0.20+` | OTel runtime | **New dependency** |
| `metrics-exporter-prometheus` | `0.16+` | Prometheus scrape endpoint | **New dependency** |

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
│     → Prometheus: expose /metrics on configured port                │
│     → OTLP: push to user-provided endpoint                         │
│                                                                     │
│  4. Start DrasiLib instances (unchanged)                            │
│     → drasi-lib emits spans + metrics through facades               │
│     → data flows to installed subscriber + recorder                 │
│                                                                     │
│  5. On shutdown: flush pending spans/metrics, shutdown provider     │
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
  tracing:
    endpoint: "http://jaeger:4317"       # gRPC OTLP endpoint; omit to disable
    serviceName: "drasi-server"           # OTel service.name resource attribute
  metrics:
    backend: prometheus                   # "prometheus" or "otlp"
    prometheus:
      port: 9090                          # Scrape endpoint port
      path: "/metrics"                    # Scrape path
    otlp:
      endpoint: "http://otel-collector:4317"
      exportInterval: 30                  # Export interval in seconds
```

All fields support environment variable interpolation:

```yaml
telemetry:
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

- **`prometheus`** — starts an HTTP listener on the configured port exposing a `/metrics` scrape endpoint
- **`otlp`** — pushes metrics to the configured OTLP endpoint at a configurable interval
- **Not configured** — no recorder installed, `metrics` facade calls are no-ops

#### 4. Startup Order

```
1. Load config (existing)
2. Build and install tracing subscriber (NEW)
3. Install metrics recorder (NEW)
4. get_or_init_global_registry() (existing)
5. Build and start DrasiLib instances (existing)
6. Start Axum API server (existing)
```

#### 5. Shutdown

On graceful shutdown (SIGTERM/SIGINT), Drasi Server flushes any pending traces/metrics before exiting.

> **Note**: To compose `ComponentLogLayer` into Drasi Server's subscriber alongside the OTLP layer, drasi-lib's `get_or_init_global_registry()` will be split into two functions (see [drasi-lib design doc, Open Issue #5](../../drasi-lib/tracing-logging/00-observability-integration.md)):
> - `init_component_log_layer()` — returns the layer for Drasi Server to compose
> - `init_default_subscriber()` — current behavior for simple embedders
>
> Drasi Server calls `init_component_log_layer()` and composes the returned layer with fmt + OTLP into its own subscriber.

### API Design

N/A — no changes to Drasi Server's REST API. The Prometheus scrape endpoint (`/metrics` on a separate port) is a new HTTP listener but is not part of the management API.

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
- **Prometheus endpoint**: The `/metrics` scrape endpoint is unauthenticated. It exposes operational data only (component IDs, event counts, latency). In production, operators should restrict network access to the metrics port.
- **No secrets in telemetry**: Span fields and metric labels contain component IDs, not user data or credentials.

## Compatibility Impact

- **No breaking changes**: Existing configs without a `telemetry` section work exactly as before.
- **New dependencies**: `tracing-opentelemetry`, `opentelemetry-otlp`, `metrics-exporter-prometheus` are added to the binary. They are only active when configured.
- **`get_or_init_global_registry()` interaction**: May require a minor refactor in drasi-lib to support composing `ComponentLogLayer` into an externally-built subscriber.

## Supportability

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Default config (no telemetry) | Integration | Start server with no `telemetry` section; verify stdout logs appear, no crashes |
| OTLP tracing | Integration | Configure endpoint to mock OTLP receiver; verify spans arrive |
| Prometheus scrape | Integration | Enable Prometheus backend; `curl localhost:9090/metrics`; verify drasi-lib metrics appear |
| OTLP metrics | Integration | Configure OTLP metrics endpoint; verify metrics arrive at mock receiver |
| Env var override | Unit | Set `OTEL_ENDPOINT` env var; verify config resolves correctly |
| Shutdown flush | Integration | Send SIGTERM; verify pending spans are exported before exit |
| Unreachable endpoint | Integration | Configure non-existent OTLP endpoint; verify server starts gracefully, logs warning |

## Development Plan

| Phase | Work Items |
|-------|-----------|
| 1. Config types | Add `TelemetryConfig`, `TracingConfig`, `MetricsConfig` structs to `config/types.rs` with serde deserialization + env var interpolation |
| 2. Dependencies | Add `tracing-opentelemetry`, `opentelemetry-otlp`, `opentelemetry_sdk`, `metrics-exporter-prometheus` to `Cargo.toml` |
| 3. Tracing setup | Implement `init_tracing()` — compose Registry with fmt + ComponentLogLayer + optional OTLP layer. Resolve `get_or_init_global_registry()` interaction |
| 4. Metrics setup | Implement `init_metrics()` — Prometheus scrape endpoint and OTLP push |
| 5. Shutdown | Add tracer provider shutdown to the existing graceful shutdown handler |
| 6. Tests | Integration tests for each backend config + graceful degradation |
| 7. Documentation | Update Drasi Server docs with telemetry configuration reference |

## Open Issues

1. **Metrics port conflict**: If the Prometheus metrics port conflicts with the Drasi Server API port, should we serve `/metrics` on the same Axum server instead of a separate listener?

2. **Trace sampling**: For high-throughput deployments, should the config support a sampling rate (e.g., `samplingRate: 0.1` to export 10% of traces)?

## References

- [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md) — Prerequisite design for tracing spans and metrics in drasi-lib
- [drasi-platform query-host init_tracer() / init_metrics()](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Reference OTLP setup in Drasi for Kubernetes
- [Drasi Server repository](https://github.com/drasi-project/drasi-server) — Source repository
- [Drasi Server documentation](https://drasi.io/drasi-server/) — User-facing docs
