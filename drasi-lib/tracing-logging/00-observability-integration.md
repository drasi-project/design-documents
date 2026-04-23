# Tracing / Logging / Metrics Integration for drasi-lib

* Project Drasi - April 10, 2026 - Ruokun Niu (@ruokun-niu)

## Overview

drasi-lib today uses the `log` crate for basic logging and a custom `ComponentLogLayer` (built on `tracing`) to route per-component logs to an internal registry. While this gives developers per-component log streams, there is no structured span hierarchy across the Source → Query → Reaction pipeline, no counters or histograms for operational metrics, and no way to export telemetry to external backends such as Jaeger or Prometheus.

This design adds structured tracing spans and explicit metrics to drasi-lib's pipeline so that developers embedding the library can follow an event end-to-end. drasi-lib emits telemetry through the `tracing` and `metrics` facade crates; the embedding application decides where the data goes by installing subscribers and recorders.

Today drasi-lib produces isolated log lines per component:

```
[INFO] source postgres-src: Starting source
[INFO] source postgres-src: Received event
[INFO] query q1: Processing source change
[ERROR] query q1: Error processing source change: timeout
[INFO] reaction webhook: Dispatching results
```

You can filter by component, but you can't tell how long anything took, whether the error was related to the received event, or how long an event waited in the queue. With this design, structured spans wrap the existing log events to add duration, causality, and nesting:

```
TRACE [0.8ms] source.dispatch { source_id=postgres-src, op=insert, label=Order, element_id=Order:42 }
  └── TRACE [0.1ms] query.receive { source_id=postgres-src, query_id=q1 }
        └── TRACE [45.3ms] query.process { query_id=q1, source_id=postgres-src }
              ├── [INFO] Processing source change      ← existing log event, now inside a timed span
              └── TRACE [3.1ms] query.dispatch { query_id=q1, added=1 }
                    └── TRACE [1.2ms] reaction.receive { reaction_id=webhook, query_id=q1 }
```

The existing log events continue to work — they just now appear inside spans that provide timing context and cross-component causality.

## Terms and Definitions

| Term | Definition |
|------|------------|
| Facade crate | A Rust crate that defines a logging/metrics API but defers the backend implementation to the consumer (e.g., `tracing`, `metrics`, `log`). A "backend" in this case is the component that actually *does something* with the telemetry data — writing it to stdout, sending it to Jaeger/Prometheus, etc. This is also called a subscriber or recorder. The facade itself only provides the call sites (`info_span!()`, `counter!()`); without a backend installed, those calls compile down to no-ops with zero runtime cost. |
| Span | A `tracing::Span` representing a unit of work with a start time, end time, and structured fields. Spans nest to form a tree. |
| Subscriber | A `tracing::Subscriber` implementation that receives span/event data and routes it to a backend (stdout, OTLP, Jaeger, etc.). Installed by the embedding application, not drasi-lib. |
| Recorder | A `metrics::Recorder` implementation that receives counter/histogram/gauge data and routes it to a backend (Prometheus, OTLP, etc.). Installed by the embedding application, not drasi-lib. |
| ComponentLogLayer | Existing custom `tracing_subscriber::Layer` in drasi-lib that intercepts tracing events and routes them to per-component broadcast channels + circular buffer history. |
| ComponentLogRegistry | Global registry in drasi-lib keyed by `(instance_id, component_type, component_id)` that stores per-component log streams and history. |

## Objectives

### User Scenarios

1. **Embedded Rust developer debugging a pipeline**: A developer using `drasi-lib` in their application wants to see how long each query takes to process source changes and whether events are backing up in the priority queue. They install `tracing_subscriber::fmt` and a `metrics-exporter-prometheus` recorder and immediately get structured logs with span context and Prometheus metrics.

2. **Developer correlating traces across services**: A developer using drasi-lib alongside other instrumented services wants to see drasi-lib spans in the same Jaeger trace as their upstream and downstream calls. Because drasi-lib uses standard `tracing` spans, the `tracing-opentelemetry` layer propagates context automatically.

### Goals

- Add structured `tracing` spans at each pipeline stage boundary (source ingest, query process, reaction dispatch) with meaningful structured fields
- Add `metrics` crate counters, histograms, and gauges for throughput, latency, queue depth, and error rates
- Preserve the existing `ComponentLogLayer` and `ComponentLogRegistry`
- Zero runtime cost when no subscriber or recorder is installed (facade pattern)
- No breaking changes to the public `DrasiLib` builder API

### Non-Goals

- Providing a built-in telemetry backend or OTLP exporter within drasi-lib. Perhaps we can include some examples
- Changes to Drasi Server.
- Distributed trace context propagation from external callers into drasi-lib source plugins. Source plugins that want to propagate parent context can do so via their own spans.

## Design Requirements

### Requirements

1. **Facade-only**: drasi-lib MUST NOT install a `tracing::Subscriber` or `metrics::Recorder`. It only emits through the facade APIs.
2. **Backward compatible**: Existing applications that use `log` crate macros and `ComponentLogLayer` MUST continue to work without changes. The `tracing-log` bridge already forwards `log::info!()` events to `tracing`.
3. **Structured fields**: All spans will include identifying fields (`source_id`, `query_id`, `reaction_id`) so that traces can be filtered and correlated.
4. **Metric naming**: All metrics use the `drasi.` prefix and follow the pattern `drasi.<component>.<metric_name>` (e.g., `drasi.query.events_processed`).

### Out of Scope

- **drasi-core instrumentation**: The query engine internals (`ContinuousQuery::process_source_change`, index operations) are treated as a black box from the instrumentation perspective.
- **Drasi Server changes**: How Drasi Server wires up subscribers/recorders for these new traces is a separate design document.
- **Custom source/reaction plugin internal instrumentation**: Plugin authors can add their own spans inside their plugin. This design provides the FFI infrastructure (trace context propagation, metrics forwarding) to make plugin telemetry visible to the host — see [Section 8: Plugin Observability Across FFI](#8-plugin-observability-across-ffi).
- **Log format changes**: The `ComponentLogLayer` output format and API remain unchanged.

## Design

### High-Level Design

The design adds two layers of instrumentation to drasi-lib's existing pipeline:

1. **Tracing spans** at pipeline stage boundaries — each stage gets a named span with structured fields. Spans nest naturally as an event flows through Source → Query → Reaction.
2. **Metrics** at the same boundaries — counters for throughput and errors, histograms for latency, and gauges for queue depth.

drasi-lib already has a `ProfilingMetadata` struct (`profiling/mod.rs`) that stamps nanosecond-precision timestamps at each pipeline stage, and a Profiler Reaction plugin that computes running statistics (mean, p50, p95, p99) over sampled events. The `metrics` crate is required because without it, drasi-lib has no way to export counters, histograms, or gauges to monitoring backends like Prometheus — `ProfilingMetadata` only outputs via the Profiler Reaction (log/file), and `tracing` spans export to trace backends (Jaeger) not metrics backends.

Both tracing and metrics use Rust facade crates (`tracing` and `metrics`) that are zero-cost when no backend is installed. The existing `ComponentLogLayer` and `ProfilingMetadata` are preserved unchanged.


### Detailed Design

#### 1. New Dependency: `metrics` Crate

Add `metrics = "0.24"` to `lib/Cargo.toml`. This is the facade crate only — no exporter.

```toml
# lib/Cargo.toml
[dependencies]
metrics = "0.24"
# tracing, tracing-subscriber, tracing-log already present
```

#### 2. Tracing Span Hierarchy and End-to-End Trace Propagation

Spans are placed at the boundaries of each pipeline stage. They form a **single connected trace tree** for each event flowing through the system, even though the pipeline uses separate async tasks connected by channels (PriorityQueue, ChangeDispatcher).

**Today**: drasi-lib's pipeline runs across 3 independent tokio tasks connected by async channels. By default, `tracing` spans don't propagate across channel boundaries — each task would create an unrelated root span, resulting in 3 disconnected traces per event instead of one.

**Proposed Solution**: When a span is created in one task, we capture a lightweight `tracing::Span` handle and carry it through the channel alongside the event data. The downstream task then creates its span as a child of the carried handle using `follows_from` or by entering the parent span's context. This produces a single trace tree:

```
Single trace per event (connected across task boundaries):

  span: source.dispatch (source_id, op, label, element_id)     [Interval A]
  │  Task T1: source plugin
  │  - wraps event in SourceEventWrapper
  │  - dispatches to ChangeDispatcher channel(s)
  │
  └──► span: query.receive (source_id, query_id)               [Interval B end]
       │  Task T2: query forwarder
       │  - receives event from dispatcher channel
       │  - enqueues (event + span handle) to PriorityQueue
       │                                                        [Interval C: PQueue wait]
       └──► span: query.process (query_id, source_id)          [Intervals D+E]
            │  Task T3: event processor
            │  - dequeues from PriorityQueue
            │  - calls process_source_change()                  [Interval D]
            │  - calls dispatch_query_results()                 [Interval E]
            │
            └─► span: query.dispatch (query_id, counts)         [child of: query.process]
               │  Task T3 (same task, nested span)
               │  - converts results, dispatches to reaction channels
               │                                                [Interval F: channel wait]
               └──► span: reaction.receive (reaction_id)        [Interval G start]
                      Task T4: reaction forwarder
                      - receives result from dispatcher channel
                      - enqueues to reaction's PriorityQueue
```

In Jaeger or any trace viewer, this appears as one trace with 5 spans showing the full event lifecycle. The gaps between spans represent time spent in channels and queues (intervals B, C, F).

##### Multi-Branch Trace Trees (Fan-Out)

The diagram above shows the simple case: 1 source → 1 query → 1 reaction. The trace tree below shows how a single source event fans out into a **branching trace tree** when multiple queries and reactions are subscribed:

```
source.dispatch { source_id=postgres-src, element_id=Order:42 }          ← root span
├──► query.receive { query_id=q1 }                                        ← fan-out #1: N queries
│    └── query.process { query_id=q1 }
│        └── query.dispatch { query_id=q1, added=1 }
│            ├──► reaction.receive { reaction_id=webhook }                 ← fan-out #2: M reactions
│            └──► reaction.receive { reaction_id=logger }
├──► query.receive { query_id=q2 }
│    └── query.process { query_id=q2 }
│        └── query.dispatch { query_id=q2, added=0 }                      ← no results = no reaction spans
└──► query.receive { query_id=q3 }
     └── query.process { query_id=q3 }
         └── query.dispatch { query_id=q3, added=1 }
             └──► reaction.receive { reaction_id=webhook }                 ← same reaction, different query
```

All branches share the same `trace_id` with `source.dispatch` as the root. In Jaeger this renders as a single expandable trace tree. When no `tracing::Subscriber` is installed, all `info_span!()` calls compile to no-ops — zero allocation, zero cost. When a subscriber is installed, the cost is proportional to the pipeline topology that the user explicitly configured. Each span is ~200 bytes in a typical subscriber (span name + fields + timestamps).

#### 3. Pipeline Stages and Instrumentation Points

The pipeline runs across 5 tokio tasks connected by channels and priority queues. Each arrow (`──▶`) crosses a task boundary:

```
Source Plugin ──▶ Query Forwarder ──▶ Query Processor ──▶ Reaction Forwarder ──▶ Reaction Processor
   (T1)              (T2)               (T3)                  (T4)                  (T5)
     │                 │                  │                      │                    │
  dispatch          channel →         PQueue →              channel →            PQueue →
  event to          PQueue           process +               reaction            reaction
  channel(s)                         dispatch                PQueue              processing
     │                 │                  │                      │                    │
  Interval A      Interval B+C       Interval D+E           Interval F           Interval G
```

**Intervals we want to capture**:

- **A**: Source dispatch — time for source to wrap event and send to channel(s)
- **B**: Source→Query channel wait — time event sits in dispatcher channel
- **C**: Query PQueue wait — time event waits in priority queue for processor
- **D**: Query engine — time inside `process_source_change()`
- **E**: Result dispatch — time to convert results and send to reaction channels
- **F**: Query→Reaction channel wait — time result sits in dispatcher channel
- **G**: Reaction enqueue + processing — time in reaction's priority queue + plugin processing

#### 4. Trace Context Propagation Across Channels

To link spans across task boundaries, we carry a `tracing::Span` handle through the channel alongside the event data. The downstream task uses `follows_from` to establish the causal relationship.

##### Where to put `parent_span`: wrapper vs event

Trace context needs to cross two types of channel boundaries:
1. **ChangeDispatcher** (T1→T2 source events, T3→T4 query results) — dispatches `Arc<T>` directly
2. **PriorityQueue** (T2→T3, T4→T5) — wraps `Arc<T>` in `PriorityQueueEvent<T>`


**Design**: Use different strategies for each boundary:

| Boundary | Strategy |
|----------|----------|
| **PriorityQueue** | Add `parent_span: Option<Span>` to `PriorityQueueEvent<T>` (the wrapper) |
| **ChangeDispatcher** | Add `parent_span: Option<Span>` to the dispatched result type (`QueryResult`) |

**PriorityQueue wrapper** (carries span alongside the event, not inside it):

```rust
// channels/priority_queue.rs
struct PriorityQueueEvent<T> {
    event: Arc<T>,
    parent_span: Option<tracing::Span>,  // NEW: on the wrapper, not inside Arc<T>
}
```

This preserves the zero-copy property: `Arc<T>` is cloned (just a refcount bump) when needed, but `T` itself is never cloned. The `parent_span` lives on the wrapper and is consumed when the event is dequeued — it does not add to the shared `Arc<T>` allocation.

**ChangeDispatcher path** (for query results crossing to reaction forwarders):

`QueryResult` gains an optional span field:

```rust
// QueryResult (queries/manager.rs or channels/events.rs)
pub struct QueryResult {
    pub query_id: String,
    pub timestamp: DateTime<Utc>,
    pub results: Vec<ResultDiff>,
    pub metadata: HashMap<String, serde_json::Value>,
    pub profiling: Option<ProfilingMetadata>,
    pub parent_span: Option<tracing::Span>,  // NEW: 8 bytes, None when no tracing backend
}
```

At each task boundary, the downstream task creates its span and links it to the carried `parent_span` using `follows_from`. This produces the connected trace tree shown in Section 2.

#### 5. Metrics Definitions

The metrics below are organized by the pipeline interval they measure (see Section 3 diagram).

**Histograms** (latency/duration — all in nanoseconds):

| Metric | Labels | Interval | Where Recorded |
|--------|--------|----------|----------------|
| `drasi.source.dispatch_duration_ns` | `source_id` | A | `SourceBase::dispatch_source_change()` — time to wrap and dispatch event |
| `drasi.query.engine_duration_ns` | `query_id` | D | Event processor — time inside `process_source_change()` only |
| `drasi.query.dispatch_duration_ns` | `query_id` | E | `dispatch_query_results()` — time to convert results + dispatch to channels |
| `drasi.reaction.enqueue_duration_ns` | `reaction_id`, `query_id` | G (partial) | Reaction forwarder — time for `enqueue_query_result()` |

**Counters** (throughput and errors):

| Metric | Labels | What | Where Recorded |
|--------|--------|------|----------------|
| `drasi.source.events_dispatched` | `source_id` | Events dispatched by source | `SourceBase::dispatch_source_change()` |
| `drasi.query.events_received` | `query_id`, `source_id` | Events received by query forwarder | Query forwarder task on `receiver.recv()` |
| `drasi.query.events_processed` | `query_id` | Events successfully processed by query engine | Event processor after `process_source_change` returns Ok |
| `drasi.query.errors` | `query_id`, `error_type` | Query engine errors | Event processor on `process_source_change` error |
| `drasi.reaction.events_enqueued` | `reaction_id`, `query_id` | Results enqueued to reaction | Reaction forwarder after `enqueue_query_result()` |
| `drasi.reaction.errors` | `reaction_id`, `error_type` | Reaction enqueue errors | Reaction forwarder on `enqueue_query_result()` error |

#### 6. Interaction with Existing ComponentLogLayer

The `ComponentLogLayer` is preserved unchanged. It operates as a `tracing_subscriber::Layer` and intercepts tracing events based on span context (`component_id`, `component_type`). The new spans we add carry these same fields, so:

- Events emitted inside a `source.ingest` span that carries `component_id` and `component_type = "source"` will be automatically routed to the correct component's log stream by `ComponentLogLayer`.
- The `ComponentLogRegistry` API (`subscribe_component_logs()`, `subscribe_component_events()`) continues to work as before.
- If the embedding application adds additional `tracing::Subscriber` layers (e.g., `tracing-opentelemetry`), spans flow to both `ComponentLogLayer` AND the external backend. This is standard `tracing` layer composition.

##### Advantages

- **Zero-cost by default**: No overhead when no subscriber/recorder is installed — facade calls compile to no-ops
- **No breaking changes**: Existing `ComponentLogLayer`, `log` crate usage, and public API are unchanged
- **Standard ecosystem**: Uses `tracing` and `metrics` — the dominant Rust observability crates — enabling direct integration with Jaeger, Prometheus, Datadog, OTLP, etc.
- **Composable**: Multiple subscribers/recorders can coexist. `ComponentLogLayer` + `tracing_opentelemetry` + `fmt` all work together
- **Embeddable into existing traces**: Because drasi-lib uses the `tracing` facade without installing its own subscriber, applications that already have a `tracing` subscriber (e.g., with `tracing-opentelemetry`) can embed drasi-lib and its pipeline spans will automatically nest under the application's active span. This means drasi-lib's `source.dispatch` → `query.process` → `reaction.receive` trace tree becomes a subtree of the application's own trace — no special configuration or API needed.


#### 7. Plugin Observability Across FFI

When sources and reactions are loaded as cdylib dynamic plugins (via `drasi-host-sdk`), they run in a separate shared library with their own tokio runtime and their own `tracing` global subscriber. This creates an FFI boundary that the normal span propagation approach (carrying `tracing::Span` handles through channels, as described in Section 4) cannot cross.

The reason is that `tracing::Span` handles are tied to the subscriber that created them — they reference internal storage in the subscriber's registry. You cannot pass a `tracing::Span` across the FFI boundary the way you can pass it through an async channel within the same process.

Today, only flat log messages cross the FFI boundary — `FfiTracingLayer` captures tracing events, flattens them to `FfiLogEntry` (level, message, component IDs), and delivers them via a C callback. No span trees, trace IDs, or metrics cross. This means plugin-internal work (e.g., Postgres WAL parsing, HTTP polling, MQTT publishing) is completely invisible to the host's tracing and metrics systems.

### Proposed Approach: Centralized Callback Bridge

All three telemetry signals — logs, metrics, and traces — use the same architecture: the plugin serializes telemetry data into a flat C-compatible struct and sends it to the host via a callback function pointer on the vtable. The host receives it and routes it through its own subscriber/recorder. This gives the host full control over filtering, sampling, and export.

In this context, the **host** is the application that loads and manages plugins — i.e., drasi-lib's manager layer (or Drasi Server wrapping it). The host runs the pipeline spans (`source.dispatch`, `query.process`, etc.) and owns the single `tracing` subscriber, `metrics` recorder, and OTLP exporter. Plugins are the cdylib shared libraries loaded into the host process — they do not have their own exporters.

**Part 1: Host-side instrumentation** — all pipeline spans/metrics from Sections 2–6 run on the host side. These are automatic and require no plugin code.

**Part 2: Trace context injection + completed span callback** — when the host calls into a plugin (or a plugin calls back into the host via `dispatch_change()`), the host passes its current `trace_id` and `parent_span_id` to the plugin via FFI structs. The plugin uses these IDs when creating spans internally. When a plugin span closes, the plugin's `FfiTracingLayer` serializes it to an `FfiCompletedSpan` and sends it back to the host via `SpanCallbackFn`. The host feeds the completed span into its own tracing subscriber for export — giving the host full control over filtering and sampling.

```rust
#[repr(C)]
pub struct FfiCompletedSpan {
    pub name: *const c_char,
    pub trace_id: [u8; 16],        // inherited from host
    pub span_id: [u8; 8],          // generated by plugin
    pub parent_span_id: [u8; 8],   // host's span or plugin's own parent
    pub start_time_ns: u64,
    pub end_time_ns: u64,
    pub fields: *const FfiSpanField,
    pub field_count: usize,
}
```

This means plugin spans appear as children of the pipeline trace. For example, a source plugin's `wal_parse` span becomes a child of `source.dispatch`, and a reaction plugin's `mqtt_publish` span becomes a child of `reaction.receive`:

```
source.dispatch { source_id=postgres-src }         ← host
  ├── wal_parse { duration=1.2ms }                 ← source plugin, via callback
  └── query.receive { query_id=q1 }                ← host
       └── query.process { query_id=q1 }           ← host
            └── reaction.receive { reaction_id=mqtt } ← host
                 └── mqtt_publish { topic=orders }  ← reaction plugin, via callback
```

All spans share the same `trace_id` and flow through the host's single OTLP exporter. Spans created under a host-injected `parent_span_id` (i.e., pipeline-related work like `wal_parse` during `dispatch_change()`) are accepted as children of the pipeline trace. Any additional spans the plugin creates outside of a host-injected context are emitted as their own separate traces, keeping the Drasi pipeline trace clean.

**Part 3: Plugin metrics forwarding** — add `FfiMetricEntry` + `MetricsCallbackFn` to the vtable. Plugin-sdk installs `FfiMetricsRecorder` that proxies to the host's single `metrics::Recorder`. Metric naming: `drasi.<component_type>.<plugin_kind>.<metric>`.

**Why centralized export matters**: With third-party source plugins, Drasi needs control over what telemetry is exported. The callback approach ensures the host can filter, sample, or drop plugin spans and metrics before they reach the OTLP exporter — plugins cannot emit telemetry that bypasses the host.

### Plugin SDK: `TelemetryHandle`

Plugin developers should not need to understand FFI trace context or W3C TraceContext. Following the `StateStoreProvider` pattern, we provide a `TelemetryHandle` via runtime context:

```rust
pub struct TelemetryHandle { /* internal: trace context + metrics proxy */ }

impl TelemetryHandle {
    pub fn span(&self, name: &str) -> SpanGuard { /* ... */ }
    pub fn count(&self, name: &str) { /* ... */ }
    pub fn count_by(&self, name: &str, value: u64) { /* ... */ }
    pub fn record_duration_ns(&self, name: &str, nanos: u64) { /* ... */ }
    pub fn set_gauge(&self, name: &str, value: f64) { /* ... */ }
    pub fn start_timer(&self, name: &str) -> TimerGuard { /* ... */ }
}
```


**Example: Source plugin** — no tracing/metrics imports needed:

```rust
async fn start(&self) -> Result<()> {
    let telemetry = self.base.telemetry();
    loop {
        let _span = telemetry.span("wal_recv");
        let _timer = telemetry.start_timer("wal_parse");
        let changes = self.recv_wal_changes().await?;
        telemetry.count_by("wal_events", changes.len() as u64);
        for change in changes { self.dispatch_change(change).await?; }
    }
}
```

Plugin developers write the same code regardless of whether their plugin is built-in or loaded as a cdylib. The `TelemetryHandle` abstracts the difference:

- **Built-in plugins**: `span()` delegates directly to `tracing::info_span!()`, `count()` to `metrics::counter!()` — no indirection.
- **cdylib plugins**: `span()` creates a span locally using the host-injected `trace_id`/`parent_span_id`; on close, it serializes to `FfiCompletedSpan` and calls back to the host via `SpanCallbackFn`. `count()` and `record_duration_ns()` serialize to `FfiMetricEntry` and call `MetricsCallbackFn`.

In both cases, telemetry flows through the host's single subscriber/recorder — plugins never need their own exporter. This follows the same pattern as `StateStoreProvider`, where plugin devs call `self.base.state_store().get("key")` without knowing the backing store.


### API Design

N/A — no changes to the public `DrasiLib` builder API, REST API, or CLI. The instrumentation is purely internal to drasi-lib's pipeline implementation. All new tracing spans and metrics are emitted through facade crates and are transparent to the public API.

### Alternatives Considered

#### 1. Use OpenTelemetry SDK Directly (Instead of Facade Crates)

Instrument drasi-lib directly with `opentelemetry` crate APIs (`tracer.start("span")`, `meter.u64_counter()`).

**Rejected because**: This would hard-couple drasi-lib to the OpenTelemetry SDK, requiring all embedding applications to use OTel. The facade approach (`tracing` + `metrics`) lets users choose any backend. This is also the approach used by the broader Rust ecosystem — libraries use facades, applications choose backends.

#### 2. Replace `log` Crate Usage with `tracing` Events Everywhere

Convert all existing `log::info!()`, `log::error!()` calls to `tracing::info!()`, `tracing::error!()`.

**Deferred**: This would be a nice cleanup but is not necessary for this design. The `tracing-log` bridge already forwards `log` events to the `tracing` subscriber. We can do this incrementally as we touch files.

#### 3. Add `#[instrument]` to All Public Functions

Automatically create spans for every public function using the `#[instrument]` attribute.

**Rejected because**: This creates too many fine-grained spans that add noise and overhead. The pipeline boundary approach (source ingest → query process → reaction dispatch) gives the right level of granularity for debugging and monitoring.

#### 4. Embed Metrics in ComponentLogLayer

Extend the existing `ComponentLogLayer` to also track counters and histograms internally rather than adding the `metrics` crate.

**Rejected because**: `ComponentLogLayer` is a log routing mechanism, not a metrics system. The `metrics` crate provides the standard Rust interface for counters/histograms/gauges with ecosystem support for exporters. Mixing concerns in `ComponentLogLayer` would make it harder to maintain.

#### 5. Independent Spans Per Task (No Cross-Channel Trace Linking)

Create spans only within each task's scope and don't carry trace context through the PriorityQueue or ChangeDispatcher channels. Each task would create a root span, producing 3 disconnected traces per event:

- **Trace A**: `source.ingest` (source forwarder task)
- **Trace B**: `query.process` → `reaction.dispatch` (event processor task)
- **Trace C**: `reaction.receive` (reaction forwarder task)

**Rejected because**: The primary value of distributed tracing is following a single event end-to-end. Three disconnected traces per event makes it impossible to correlate what happened to a specific source change across the pipeline — you'd have to manually match them by timestamp and field values. Carrying a span handle through the channel is a small amount of additional data (one `Arc` clone per event) and standard practice in async Rust applications that use channel-based architectures. The `follows_from` relationship in the `tracing` crate exists specifically for this use case.


## Security

No new security concerns. Tracing spans and metrics expose operational data (component IDs, event counts, latency) but not user data, credentials, or query content. Span fields use component identifiers that are already visible in logs today.

If an embedding application exports traces to an external collector (e.g., Jaeger, OTLP), the security of that transport is the application's responsibility — not drasi-lib's.

## Compatibility Impact

- **No breaking changes**: The public `DrasiLib` API is unchanged. No new required configuration.
- **New dependency**: `metrics 0.24` is added to `Cargo.toml`. This is a lightweight facade crate with no transitive dependencies beyond `portable-atomic`.
- **Behavioral change**: Applications that already have a `tracing::Subscriber` installed will see new spans (`source.ingest`, `query.process`, `reaction.dispatch`) in their output. This is additive and should not break existing behavior.
- **Existing `log` crate usage**: Continues to work. `tracing-log` bridge is preserved.

## Supportability

### Telemetry

This design *is* the telemetry story for drasi-lib. After implementation, the following telemetry is available:

| Signal | What | How to enable |
|--------|------|---------------|
| Structured logs | `tracing::info!()` / `tracing::error!()` events with span context | Install any `tracing::Subscriber` (e.g., `tracing_subscriber::fmt::init()`) |
| Distributed traces | Nested spans across Source → Query → Reaction | Install `tracing-opentelemetry` layer with an OTLP/Jaeger exporter |
| Metrics | Counters, histograms, gauges (see table above) | Install any `metrics::Recorder` (e.g., `metrics-exporter-prometheus`) |
| Per-component logs | Log streams per Source/Query/Reaction | Existing `ComponentLogLayer` — no changes needed |

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Span creation | Unit | Use `tracing_subscriber::fmt::TestWriter` or `tracing-test` crate to assert that expected spans are created with correct fields when processing a mock source change |
| Metric recording | Unit | Install `metrics-util::debugging::DebuggingRecorder`, process events, assert counter/histogram values |
| ComponentLogLayer compatibility | Integration | Existing tests for `subscribe_component_logs()` must continue to pass with the new spans in place |
| Zero-cost when no backend | Unit | Process events without any subscriber/recorder installed; verify no panics, no overhead (benchmark if needed) |
| End-to-end with OTLP | Manual / Integration | Example app with `tracing-opentelemetry` + Jaeger; verify spans appear in Jaeger UI with correct nesting and fields |


## Open Issues

1. **Metric label cardinality**: With dynamic `query_id` / `source_id` labels, if a user creates a very large number of components, the metrics cardinality could grow. Should we cap or provide a configuration to disable per-component metric labels?

2. **Bootstrap span granularity**: During bootstrap, `process_source_change` is called once per initial data element (potentially thousands). Should each bootstrap event get its own `query.bootstrap` span, or should there be a single parent span for the entire bootstrap phase with lightweight events per element?

3. **Histogram bucket configuration**: The `metrics` crate leaves bucket configuration to the recorder. Should drasi-lib document recommended histogram buckets for `processing_duration_ns` and `dispatch_duration_ns`, or leave that entirely to the user?

4. **Span naming convention**: Should span names use dots (`query.process`) or slashes (`query/process`) or OpenTelemetry-style (`drasi.query.process`)? The current proposal uses dots. This should be consistent with whatever convention drasi-platform adopts.

5. **`get_or_init_global_registry()` split**: To support Drasi Server composing `ComponentLogLayer` into its own multi-layer subscriber (with OTLP), we propose splitting `get_or_init_global_registry()` into two functions:
   - `init_component_log_layer()` — creates the registry, channel, and worker thread, returns the `ComponentLogLayer` for the caller to compose into their own subscriber
   - `init_default_subscriber()` — calls `init_component_log_layer()`, composes it with the `fmt` layer, and installs the global subscriber (same behavior as today)

   Simple embedders call `init_default_subscriber()` and get current behavior. Drasi Server calls `init_component_log_layer()`, adds the OTLP layer alongside it, and installs its own subscriber. This is a non-breaking change.

6. **Plugin trace context granularity**: Should trace context be passed per-event (each `dispatch_change()` / reaction invocation carries the current `trace_id` + `parent_span_id`) or once at plugin initialization? Per-event links each plugin span to its specific pipeline event. Per-init is simpler but all plugin work shares one parent span.

7. **Plugin metrics naming governance**: Plugin metrics use `drasi.<component_type>.<plugin_kind>.<metric>`. Should drasi-lib enforce this prefix in the `FfiMetricsRecorder`, or trust plugin authors to follow the convention? Enforcement prevents namespace collisions but limits flexibility.

## References

- [drasi-lib crate](https://crates.io/crates/drasi-lib) — Published Rust crate
- [drasi-core repository](https://github.com/drasi-project/drasi-core) — Source repository for drasi-lib
- [drasi-platform query-host](https://github.com/drasi-project/drasi-platform/tree/main/query-container/query-host) — Reference implementation of OpenTelemetry tracing and metrics in Drasi
- [`tracing` crate](https://crates.io/crates/tracing) — Structured diagnostics facade for Rust
- [`metrics` crate](https://crates.io/crates/metrics) — Metrics facade for Rust
- [drasi-platform query-host `init_tracer()` and `init_metrics()`](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Existing OTLP setup pattern used in Drasi for Kubernetes
