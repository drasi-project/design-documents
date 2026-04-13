# Tracing / Logging / Metrics Integration for Drasi Server

* Project Drasi - April 10, 2026 - Ruokun Niu (@ruokun-niu)

## Overview

Drasi Server is a standalone single-process deployment of Drasi that embeds `drasi-lib`. With the [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md), drasi-lib now emits structured tracing spans and `metrics` crate counters/histograms/gauges through facade APIs — but those are no-ops until the embedding application installs a backend. This design makes Drasi Server that embedding application: it wires up the tracing subscriber and metrics recorder so that drasi-lib's telemetry flows to external backends (OTLP, Prometheus, stdout).

Drasi Server already uses a plugin architecture for sources and reactions. This design extends that same plugin model to telemetry backends, so users only install the export backends they need (e.g., OTLP, Prometheus) without bloating the base binary.

## Terms and Definitions

| Term | Definition |
|------|------------|
| OTLP | OpenTelemetry Protocol — a standard for exporting traces and metrics to collectors (e.g., Jaeger, Grafana Tempo, Prometheus via OTLP receiver). |
| Telemetry plugin | A dynamically loaded shared library (`libdrasi_telemetry_*`) that implements the `TelemetryPluginDescriptor` trait. It installs tracing subscriber layers and/or metrics recorders when loaded. Follows the same OCI-based discovery and installation model as source and reaction plugins. |

See the [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md) for definitions of facade crate, span, subscriber, and recorder.

## Objectives

### User Scenarios

1. **Developer running Drasi Server locally**: A developer starts Drasi Server with default config. Structured logs appear on stdout with span context. No extra setup needed.

2. **Operator exporting traces to Jaeger**: An operator sets `otlp_endpoint: "http://jaeger:4317"` in the server config. Drasi Server installs a `tracing-opentelemetry` layer that exports drasi-lib's end-to-end spans (source.ingest → query.process → reaction.dispatch → reaction.receive) to Jaeger.

3. **Operator scraping metrics with Prometheus**: An operator enables the Prometheus scrape endpoint in config. Drasi Server installs a `metrics-exporter-prometheus` recorder and exposes `/metrics` on a configurable port. Prometheus scrapes drasi-lib's counters (events_processed, errors) and histograms (processing_duration_ms).

4. **Operator pushing metrics via OTLP**: An operator configures OTLP metrics export instead of Prometheus scrape. Drasi Server installs a `metrics-exporter-opentelemetry` recorder that pushes to the OTLP endpoint.

### Goals

- Install a `tracing::Subscriber` that sends drasi-lib's spans to stdout and optionally to an OTLP endpoint
- Install a `metrics::Recorder` that exports drasi-lib's metrics via Prometheus scrape endpoint or OTLP push
- Make telemetry configuration available via YAML config and environment variable overrides
- Preserve the existing `ComponentLogLayer` and per-component log streaming API endpoints
- Preserve the existing `logLevel` configuration

### Non-Goals

- Adding new tracing spans specific to Drasi Server's API layer (Axum routes, plugin management). This may be added later.

## Design Requirements

### Requirements

1. **Backward compatible**: Existing Drasi Server deployments with no telemetry config MUST continue to work — structured logs on stdout with the configured `logLevel`.
2. **Opt-in telemetry export**: OTLP tracing and metrics export MUST be opt-in via configuration. No external connections by default.
3. **Layered subscribers**: The `ComponentLogLayer` (installed by drasi-lib's `get_or_init_global_registry()`) MUST coexist with any plugin-provided layers.
4. **Plugin-based backends**: Telemetry backends MUST be dynamically loaded plugins, consistent with the existing source/reaction plugin model. The drasi-server binary MUST NOT statically link OTLP or Prometheus exporter libraries.

### Dependencies

| Dependency | Version | Purpose | Notes |
|------------|---------|---------|-------|
| `drasi-lib` | current | Emits spans and metrics via facades | Existing dependency |
| `drasi_host_sdk` | current | Plugin loading infrastructure | Existing dependency |
| `tracing` | `0.1` | Tracing facade | **Move from dev to production dependency** |
| `tracing-subscriber` | `0.3` | Subscriber composition (Registry, fmt, env-filter) | **New production dependency** |



### Out of Scope

- Server-specific spans for API endpoints, plugin loading, or config parsing.
- Changes to the existing REST API for component logs/events.
- Drasi Server web UI changes.

## Design

### High-Level Design

Drasi Server currently initializes logging by setting `RUST_LOG` from the YAML config's `logLevel` field and calling `get_or_init_global_registry()` from drasi-lib. This design extends that initialization by loading telemetry plugins — dynamically loaded shared libraries that install tracing subscriber layers and metrics recorders. This follows the same pattern as source/reaction plugins: users declare what they need in config, plugins are auto-installed from OCI registries, and the server loads them at startup.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Drasi Server (main.rs)                                             │
│                                                                     │
│  Startup:                                                           │
│                                                                     │
│  1. Load config                                                     │
│  2. Discover & load telemetry plugins (libdrasi_telemetry_*)        │
│  3. Build tracing subscriber:                                       │
│     ┌─ Registry ────────────────────────────────────────────────┐   │
│     │  ┌─────────────────┐  ┌────────────────────────────────┐  │   │
│     │  │ ComponentLogLayer│  │ fmt layer (stdout, built-in)   │  │   │
│     │  │ (from drasi-lib) │  │ (filtered by logLevel)         │  │   │
│     │  └─────────────────┘  └────────────────────────────────┘  │   │
│     │  ┌────────────────────────────────────────────────────┐   │   │
│     │  │ Plugin-provided layers (e.g., OTLP from            │   │   │
│     │  │ libdrasi_telemetry_otlp)                            │   │   │
│     │  └────────────────────────────────────────────────────┘   │   │
│     └───────────────────────────────────────────────────────────┘   │
│                                                                     │
│  4. Plugin-installed metrics recorder (e.g., Prometheus from        │
│     libdrasi_telemetry_prometheus)                                   │
│                                                                     │
│  5. Start DrasiLib instances (unchanged)                            │
│     → drasi-lib emits spans + metrics through facades               │
│     → data flows to plugin-installed subscriber + recorder          │
│                                                                     │
│  6. On shutdown: call plugin shutdown hooks, flush telemetry        │
└─────────────────────────────────────────────────────────────────────┘
```

### Detailed Design

#### 1. Configuration

Telemetry plugins are declared in the same `plugins` array used for sources and reactions, and configured in a new `telemetry` section:

```yaml
# config/server.yaml
apiVersion: drasi.io/v1
id: my-server
host: 0.0.0.0
port: 8080
logLevel: info

# Telemetry plugins listed alongside source/reaction plugins
plugins:
  - ref: source/postgres:0.1.8
  - ref: reaction/webhook:0.1.0
  - ref: telemetry/otlp:0.1.0           # NEW: telemetry plugin
  - ref: telemetry/prometheus:0.1.0     # NEW: telemetry plugin

# NEW: optional telemetry configuration (passed to loaded plugins)
telemetry:
  tracing:
    kind: otlp                             # matches telemetry plugin kind
    endpoint: "http://localhost:4317"
    serviceName: "drasi-server"
  metrics:
    kind: prometheus                       # matches telemetry plugin kind
    port: 9090
    path: "/metrics"
```

All telemetry fields support environment variable interpolation using the existing `${VAR:-default}` syntax:

```yaml
telemetry:
  tracing:
    kind: otlp
    endpoint: "${OTEL_ENDPOINT:-}"
  metrics:
    kind: prometheus
    port: "${METRICS_PORT:-9090}"
```

When no telemetry plugins are installed or the `telemetry` section is omitted, Drasi Server behaves exactly as today — stdout logs at the configured `logLevel`, no external telemetry export.

#### 2. Telemetry Plugin Descriptor Trait

A new `TelemetryPluginDescriptor` trait is added to `drasi_plugin_sdk`, following the same pattern as `SourcePluginDescriptor` and `ReactionPluginDescriptor`:

```rust
// drasi_plugin_sdk::telemetry

/// Descriptor for a telemetry plugin loaded as a shared library.
pub trait TelemetryPluginDescriptor: Send + Sync {
    /// Plugin kind (e.g., "otlp", "prometheus"). Matched against config.
    fn kind(&self) -> &str;

    /// Called during server startup to install a tracing subscriber layer.
    /// Returns a boxed layer that will be composed into the global subscriber.
    /// Return None if this plugin doesn't provide tracing export.
    fn create_tracing_layer(
        &self,
        config: &serde_json::Value,
    ) -> Option<Box<dyn tracing_subscriber::Layer<tracing_subscriber::Registry> + Send + Sync>>;

    /// Called during server startup to install a metrics recorder.
    /// The plugin is responsible for calling metrics::set_global_recorder().
    /// Return None if this plugin doesn't provide metrics export.
    fn install_metrics_recorder(
        &self,
        config: &serde_json::Value,
    ) -> Result<(), Box<dyn std::error::Error>>;

    /// Called on graceful shutdown to flush pending data.
    fn shutdown(&self);
}
```

Plugins are discovered at startup from shared libraries matching `libdrasi_telemetry_*` — the same file naming and loading pattern used for source/reaction plugins.

#### 3. Example Telemetry Plugins

**`libdrasi_telemetry_otlp`** — provides both tracing and metrics via OTLP:

```rust
// Inside the OTLP telemetry plugin crate
struct OtlpTelemetryPlugin;

impl TelemetryPluginDescriptor for OtlpTelemetryPlugin {
    fn kind(&self) -> &str { "otlp" }

    fn create_tracing_layer(&self, config: &Value) -> Option<Box<dyn Layer<Registry> + Send + Sync>> {
        let endpoint = config["endpoint"].as_str()?;
        let service_name = config.get("serviceName")
            .and_then(|v| v.as_str())
            .unwrap_or("drasi-server");

        let tracer = opentelemetry_otlp::new_pipeline()
            .tracing()
            .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_endpoint(endpoint))
            .with_trace_config(/* ... service_name ... */)
            .install_batch(opentelemetry_sdk::runtime::Tokio)
            .ok()?;

        Some(Box::new(tracing_opentelemetry::layer().with_tracer(tracer)))
    }

    fn install_metrics_recorder(&self, config: &Value) -> Result<(), Box<dyn std::error::Error>> {
        // Install OTLP metrics exporter
        Ok(())
    }

    fn shutdown(&self) {
        opentelemetry::global::shutdown_tracer_provider();
    }
}
```

**`libdrasi_telemetry_prometheus`** — provides metrics only via scrape endpoint:

```rust
struct PrometheusTelemetryPlugin;

impl TelemetryPluginDescriptor for PrometheusTelemetryPlugin {
    fn kind(&self) -> &str { "prometheus" }

    fn create_tracing_layer(&self, _config: &Value) -> Option<Box<dyn Layer<Registry> + Send + Sync>> {
        None  // Prometheus is metrics-only, no tracing layer
    }

    fn install_metrics_recorder(&self, config: &Value) -> Result<(), Box<dyn std::error::Error>> {
        let port = config.get("port").and_then(|v| v.as_u64()).unwrap_or(9090) as u16;
        metrics_exporter_prometheus::PrometheusBuilder::new()
            .with_http_listener(([0, 0, 0, 0], port))
            .install()?;
        Ok(())
    }

    fn shutdown(&self) { /* Prometheus recorder is dropped automatically */ }
}
```

#### 4. Server Startup: Loading and Composing Plugins

Drasi Server's startup sequence is extended to load telemetry plugins before building the tracing subscriber:

```
1. Load config (existing)
2. Install source/reaction plugins from OCI (existing)
3. Install telemetry plugins from OCI (NEW — same mechanism)
4. Load telemetry plugins: discover libdrasi_telemetry_* shared libraries
5. Build tracing subscriber:
   a. ComponentLogLayer (from drasi-lib)
   b. fmt layer (stdout, built-in)
   c. For each loaded telemetry plugin where config.telemetry.tracing.kind
      matches plugin.kind(): call create_tracing_layer() and add to subscriber
6. For each loaded telemetry plugin where config.telemetry.metrics.kind
   matches plugin.kind(): call install_metrics_recorder()
7. get_or_init_global_registry() (existing)
8. Build and start DrasiLib instances (existing)
9. Start Axum API server (existing)
```

On shutdown:
```
1. Stop DrasiLib instances (existing)
2. Call shutdown() on each loaded telemetry plugin (NEW)
3. Stop Axum API server (existing)
```

> **Note**: The interaction between the composed subscriber and `get_or_init_global_registry()` needs care. Currently `get_or_init_global_registry()` sets the global subscriber. With this design, Drasi Server sets the global subscriber first (with ComponentLogLayer + plugin layers composed in), so `get_or_init_global_registry()` may need to be refactored to return the layer rather than installing the subscriber itself.

### API Design

N/A — no changes to Drasi Server's REST API. The Prometheus scrape endpoint (`/metrics` on a separate port) is a new HTTP listener but is not part of the management API.

### Alternatives Considered

#### 1. Statically Compile All Backends into the Drasi Server Binary

Add `tracing-opentelemetry`, `opentelemetry-otlp`, `metrics-exporter-prometheus`, etc. directly to drasi-server's `Cargo.toml` and select backends via config at runtime.

**Rejected because**: This bloats the binary with all possible backends even if the user only needs one (or none). It also means adding a new backend requires a drasi-server release. The plugin approach lets users install only what they need and allows third-party telemetry plugins without changes to drasi-server.

#### 2. Let Users Configure Telemetry Entirely Outside Drasi Server

Rely on `RUST_LOG` and OpenTelemetry's auto-instrumentation environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`, etc.) without any Drasi Server config.

**Rejected because**: OpenTelemetry's env var auto-configuration requires the application to opt into it with code. There's no way to auto-install a `tracing-opentelemetry` layer or a `metrics::Recorder` just from env vars — the setup code must exist in the binary (or plugin). The YAML config makes telemetry a first-class, documented feature.

#### 3. Hard-Code OTLP as the Only Export Backend

Always export to OTLP, require users to run an OpenTelemetry Collector to fan out to Prometheus/Jaeger/etc.

**Rejected because**: For simple deployments (single Docker container), requiring an OTel Collector just to get Prometheus metrics is heavy. The plugin approach lets users choose Prometheus directly without an intermediate collector.

## Security

- **OTLP endpoint**: The OTLP exporter connects to a user-configured endpoint. If the endpoint is remote, users should use TLS. Drasi Server does not enforce TLS — this matches the pattern used by drasi-platform's query-host.
- **Prometheus endpoint**: The `/metrics` scrape endpoint is unauthenticated. It exposes operational data only (component IDs, event counts, latency). In production, operators should restrict network access to the metrics port.
- **No secrets in telemetry**: Span fields and metric labels contain component IDs, not user data or credentials.

## Compatibility Impact

- **No breaking changes**: Existing configs without a `telemetry` section work exactly as before.
- **No new heavy dependencies in drasi-server**: OTLP and Prometheus libraries live inside their respective telemetry plugins, not in the server binary.
- **`get_or_init_global_registry()` interaction**: May require a minor refactor in drasi-lib to support composing `ComponentLogLayer` into an externally-built subscriber. This is backward compatible — the function can detect whether a global subscriber is already set.

## Supportability

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Default config (no telemetry) | Integration | Start server with no `telemetry` section; verify stdout logs appear, no crashes |
| OTLP tracing plugin | Integration | Install `telemetry/otlp` plugin, configure endpoint to mock OTLP receiver; verify spans arrive |
| Prometheus plugin | Integration | Install `telemetry/prometheus` plugin; `curl localhost:9090/metrics`; verify drasi-lib metrics appear |
| Unknown plugin kind | Integration | Configure `telemetry.tracing.kind: foo` with no matching plugin; verify graceful warning, server starts without export |
| Env var override | Unit | Set `OTEL_ENDPOINT` env var; verify config resolves correctly |
| Shutdown flush | Integration | Send SIGTERM; verify pending spans are exported before exit |

## Development Plan

| Phase | Work Items |
|-------|-----------|
| 1. Plugin SDK | Add `TelemetryPluginDescriptor` trait to `drasi_plugin_sdk`. Define C ABI vtable for `create_tracing_layer`, `install_metrics_recorder`, `shutdown`. |
| 2. Plugin loader | Extend `PluginLoader` to discover and load `libdrasi_telemetry_*` shared libraries. Add `TelemetryPluginRegistry`. |
| 3. Config types | Add `TelemetryConfig` with `tracing.kind`/`metrics.kind` + generic JSON config pass-through to `config/types.rs`. |
| 4. Server startup | Compose subscriber from ComponentLogLayer + fmt + plugin-provided layers. Call `install_metrics_recorder` for metrics plugins. Resolve `get_or_init_global_registry()` interaction. |
| 5. OTLP plugin | Implement `libdrasi_telemetry_otlp` plugin crate. Publish to OCI registry. |
| 6. Prometheus plugin | Implement `libdrasi_telemetry_prometheus` plugin crate. Publish to OCI registry. |
| 7. Shutdown | Call `shutdown()` on loaded telemetry plugins during graceful shutdown. |
| 8. Tests | Integration tests for each plugin. Test missing plugin graceful degradation. |
| 9. Documentation | Update Drasi Server docs with telemetry plugin configuration reference. |

## Open Issues

1. **`get_or_init_global_registry()` refactor**: This function currently installs the global subscriber. Drasi Server needs to compose `ComponentLogLayer` into its own subscriber alongside plugin-provided layers. Should `get_or_init_global_registry()` be changed to return the layer, or should Drasi Server bypass it and construct the `ComponentLogLayer` directly?

2. **C ABI for tracing layers**: The `tracing_subscriber::Layer` trait uses generics and is not directly FFI-safe. The plugin SDK needs a stable C ABI bridge for passing layers across the shared library boundary. Options include: (a) a `Box<dyn Layer>` with vtable, (b) having the plugin install the global subscriber itself rather than returning a layer, or (c) having the plugin communicate via a simpler interface (e.g., return an OTLP endpoint string and let the server build the layer). This is the key technical challenge of the plugin approach.

3. **Multiple metrics recorders**: The `metrics` crate only supports one global recorder. If a user configures both a Prometheus and an OTLP metrics plugin, which one wins? Options: first-configured wins with a warning, or use `metrics-fanout` crate to fan out to multiple recorders.

4. **Metrics port conflict**: If the Prometheus metrics port conflicts with the Drasi Server API port, should we serve `/metrics` on the same Axum server instead of a separate listener?

5. **Trace sampling**: For high-throughput deployments, should telemetry plugins support a sampling rate config (e.g., `samplingRate: 0.1` to export 10% of traces)?

## References

- [drasi-lib observability design](../../drasi-lib/tracing-logging/00-observability-integration.md) — Prerequisite design for tracing spans and metrics in drasi-lib
- [drasi-platform query-host init_tracer() / init_metrics()](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Reference OTLP setup in Drasi for Kubernetes
- [Drasi Server repository](https://github.com/drasi-project/drasi-server) — Source repository
- [Drasi Server documentation](https://drasi.io/drasi-server/) — User-facing docs
