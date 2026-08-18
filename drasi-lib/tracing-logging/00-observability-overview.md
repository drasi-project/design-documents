# Observability for drasi-lib — Overview and Shared Foundations

* Project Drasi - April 10, 2026 - Ruokun Niu (@ruokun-niu)

> **Document set.** This design is split across three documents. Read this one first — it covers
> the concerns shared by all telemetry signals.
>
> | Document | Covers |
> |----------|--------|
> | **00 — Overview and Shared Foundations** (this doc) | Objectives, terminology, the facade principle, the pipeline model, logging, plugin telemetry transport across FFI, enablement and configuration, naming conventions, phase plan |
> | [01 — Tracing](01-tracing.md) | Span hierarchy, trace context propagation across tasks, trace rooting, ladder diagrams |
> | [02 — Metrics](02-metrics.md) | Metric definitions, collection architecture, sampling vs aggregation, environment and storage metrics |
>
> Drasi Server's side of this — how the embedding application installs subscribers, recorders and
> exporters — is covered separately in the
> [Drasi Server observability design](../../drasi-server/tracing-logging/00-observability-integration.md).

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
| Host | The original tokio runtime context that loads and manages plugins — drasi-lib's manager layer, plus whatever application embeds it (typically Drasi Server). The host runs the pipeline spans and owns the single `tracing` subscriber and `metrics` recorder. **Not** to be confused with the *host SDK* (`drasi-host-sdk`), which is the crate providing the plugin-loading machinery, or with a plugin's own isolated tokio runtime. |
| Plugin runtime | The isolated tokio runtime inside a cdylib plugin. It has its own `tracing` global subscriber, separate from the host's. |
| ComponentLogLayer | Existing custom `tracing_subscriber::Layer` in drasi-lib that intercepts tracing events and routes them to per-component broadcast channels + circular buffer history. |
| ComponentLogRegistry | Global registry in drasi-lib keyed by `(instance_id, component_type, component_id)` that stores per-component log streams and history. |

## Objectives

### User Scenarios

1. **Embedded Rust developer debugging a pipeline**: A developer using `drasi-lib` in their application wants to see how long each query takes to process source changes and whether events are backing up in the priority queue. They install `tracing_subscriber::fmt` and a `metrics-exporter-prometheus` recorder and immediately get structured logs with span context and Prometheus metrics.

2. **Developer correlating traces across services**: A developer using drasi-lib alongside other instrumented services wants to see drasi-lib spans in the same Jaeger trace as their upstream and downstream calls. Because drasi-lib uses standard `tracing` spans, the `tracing-opentelemetry` layer propagates context automatically.

### Goals

- Add structured `tracing` spans at each pipeline stage boundary (source ingest, query process, reaction dispatch) with meaningful structured fields — see [01 — Tracing](01-tracing.md)
- Add `metrics` crate counters, histograms, and gauges for throughput, latency, queue depth, and error rates — see [02 — Metrics](02-metrics.md)
- Preserve the existing `ComponentLogLayer` and `ComponentLogRegistry`
- Zero runtime cost when no subscriber or recorder is installed (facade pattern)
- No breaking changes to the public `DrasiLib` builder API

### Non-Goals

- Providing a built-in telemetry backend or OTLP exporter within drasi-lib. Perhaps we can include some examples
- Changes to Drasi Server.

## Design Requirements

### Requirements

1. **Facade-only**: drasi-lib MUST NOT install a `tracing::Subscriber` or `metrics::Recorder`. It only emits through the facade APIs.
2. **Backward compatible**: Existing applications that use `log` crate macros and `ComponentLogLayer` MUST continue to work without changes. The `tracing-log` bridge already forwards `log::info!()` events to `tracing`.
3. **Structured fields**: All spans will include identifying fields (`source_id`, `query_id`, `reaction_id`) so that traces can be filtered and correlated.
4. **Metric naming**: All metrics use the `drasi.` root namespace and follow the scheme defined in [Naming and Namespacing Conventions](#naming-and-namespacing-conventions) — dot-separated namespaces, base units in the leaf, `_total` on monotonic counters (e.g. `drasi.query.events_processed_total`, `drasi.query.engine_duration_seconds`).

### Out of Scope

- **drasi-core instrumentation**: The query engine internals (`ContinuousQuery::process_source_change`, index operations) are treated as a black box from the instrumentation perspective.
- **Drasi Server changes**: How Drasi Server wires up subscribers/recorders for these new traces is a separate design document.
- **Custom source/reaction plugin internal instrumentation**: Plugin authors can add their own spans inside their plugin. This design provides the FFI infrastructure (trace context propagation, metrics forwarding) to make plugin telemetry visible to the host — see [Plugin Telemetry Across FFI](#plugin-telemetry-across-ffi) below.
- **Log format changes**: The `ComponentLogLayer` output format and API remain unchanged.

## Design

### Signal Model

The design adds two layers of instrumentation to drasi-lib's existing pipeline:

1. **Tracing spans** at pipeline stage boundaries — each stage gets a named span with structured fields. Spans nest naturally as an event flows through Source → Query → Reaction.
2. **Metrics** at the same boundaries — counters for throughput and errors, histograms for latency, and gauges for queue depth.

drasi-lib already has a `ProfilingMetadata` struct (`profiling/mod.rs`) that stamps nanosecond-precision timestamps at each pipeline stage, and a Profiler Reaction plugin that computes running statistics (mean, p50, p95, p99) over sampled events. The `metrics` crate is required because without it, drasi-lib has no way to export counters, histograms, or gauges to monitoring backends like Prometheus — `ProfilingMetadata` only outputs via the Profiler Reaction (log/file), and `tracing` spans export to trace backends (Jaeger) not metrics backends.

Both tracing and metrics use Rust facade crates (`tracing` and `metrics`) that are near-zero-cost when no backend is installed. The existing `ComponentLogLayer` and `ProfilingMetadata` are preserved unchanged.

### Pipeline Model and Instrumentation Points

Both the tracing and metrics designs are anchored to the same pipeline model, defined here once.

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

Spans are placed at these boundaries ([01 — Tracing](01-tracing.md)); histograms measure these
intervals ([02 — Metrics](02-metrics.md)).

### Logging

#### Interaction with the Existing ComponentLogLayer

The `ComponentLogLayer` is preserved unchanged. It operates as a `tracing_subscriber::Layer` and intercepts tracing events based on span context (`component_id`, `component_type`). The new spans we add carry these same fields, so:

- Events emitted inside a `source.dispatch` span that carries `component_id` and `component_type = "source"` will be automatically routed to the correct component's log stream by `ComponentLogLayer`.
- The `ComponentLogRegistry` API (`subscribe_component_logs()`, `subscribe_component_events()`) continues to work as before.
- If the embedding application adds additional `tracing::Subscriber` layers (e.g., `tracing-opentelemetry`), spans flow to both `ComponentLogLayer` AND the external backend. This is standard `tracing` layer composition.

#### Delivery and Loss Semantics

> **OPEN — to be resolved in this revision.** Plugin log records are believed to be collected
> inside the plugin runtime and forwarded asynchronously to the host, but the exact mechanism is
> unconfirmed. This section must document: whether records are transferred independently, batched,
> piggybacked on another FFI message, or flushed at defined lifecycle points; and **what is lost if
> a plugin crashes before its buffered records cross the boundary**. Operators need to know whether
> the diagnostic information generated immediately before a failure survives.

### Plugin Telemetry Across FFI

When sources and reactions are loaded as cdylib dynamic plugins (via `drasi-host-sdk`), they run in a separate shared library with their own tokio runtime and their own `tracing` global subscriber. This creates an FFI boundary that the normal span propagation approach (carrying `tracing::Span` handles through channels, as described in [01 — Tracing](01-tracing.md)) cannot cross.

The reason is that `tracing::Span` handles are tied to the subscriber that created them — they reference internal storage in the subscriber's registry. You cannot pass a `tracing::Span` across the FFI boundary the way you can pass it through an async channel within the same process.

Today, only flat log messages cross the FFI boundary — `FfiTracingLayer` captures tracing events, flattens them to `FfiLogEntry` (level, message, component IDs), and delivers them via a C callback. No span trees, trace IDs, or metrics cross. This means plugin-internal work (e.g., Postgres WAL parsing, change-feed decoding, MQTT publishing) is completely invisible to the host's tracing and metrics systems.

#### Proposed Approach: Centralized Callback Bridge

All three telemetry signals — logs, metrics, and traces — use the same architecture: the plugin serializes telemetry data into a flat C-compatible struct and sends it to the host via a callback function pointer on the vtable. The host receives it and routes it through its own subscriber/recorder. This gives the host full control over filtering, sampling, and export.

In this context, the **host** is the application that loads and manages plugins — i.e., drasi-lib's manager layer (or Drasi Server wrapping it). The host runs the pipeline spans (`source.dispatch`, `query.process`, etc.) and owns the single `tracing` subscriber, `metrics` recorder, and OTLP exporter. Plugins are the cdylib shared libraries loaded into the host process — they do not have their own exporters.

**Part 1: Host-side instrumentation** — all pipeline spans and metrics run on the host side. These are automatic and require no plugin code.

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

**Ownership contract**: The plugin allocates all memory (`name`, `fields`). Pointers are valid only for the duration of the callback — the host must copy any data it needs before the callback returns. The plugin frees the memory after the callback returns. This is the same ownership model used by the existing `FfiLogEntry` callback.

**Host-side reconstruction**: The host cannot construct a `tracing::Span` with a foreign `trace_id`/`span_id` or preset start/end timestamps — the `tracing` API deliberately does not expose span-ID or timing control. So plugin spans are **not** re-created as `tracing` spans. Instead, the host bridges each `FfiCompletedSpan` directly into the OpenTelemetry SDK it already runs for the pipeline: it constructs an `opentelemetry_sdk::export::trace::SpanData` (setting `span_context` from the plugin's `trace_id` + `span_id`, `parent_span_id`, `start_time`/`end_time` from the `*_ns` fields, and attributes from `fields`) and hands it to the same `SpanExporter` / OTLP pipeline that exports the host's `tracing-opentelemetry` spans. Because the plugin span carries the host-injected `trace_id` and a `parent_span_id` pointing at the live pipeline span, the exported plugin span nests under the pipeline trace with no correlation guesswork. This bridge runs entirely on the host side, so the host retains full control over filtering and sampling before export — plugins never touch the exporter directly.

> **OPEN — to be resolved in this revision.** Two questions remain on this mechanism:
> 1. **Do plugin spans arrive fully formed, or are they reconstructed/enriched host-side?** The
>    description above asserts host-side `SpanData` construction; this needs validation against the
>    implementation.
> 2. **When do they transfer?** Immediate, batched, piggybacked on another FFI message, or flushed
>    at lifecycle points — and what is lost on plugin crash (same question as logging, above).
>
> The revision must also walk through span creation end to end: for each span, who creates it (host
> auto-wrap vs. plugin code vs. host reconstruction), at what point in the call, what parent it
> binds to, and when it closes.

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

**Part 3: Plugin metrics forwarding** — add `FfiMetricEntry` + `MetricsCallbackFn`, installed via a
`set_metrics_recorder` setter on `FfiPluginRegistration` (library-scoped, alongside
`set_log_callback`) rather than on any per-kind vtable. Plugin-sdk installs an `FfiMetricsRecorder`
as the plugin library's global `metrics` recorder, which proxies to the host. Scoping it to the
library rather than to `FfiRuntimeContext` is what makes it reach every plugin kind — see
[Which Plugin Types Get Telemetry](#which-plugin-types-get-telemetry).

**Why centralized export matters**: With third-party source plugins, Drasi needs control over what telemetry is exported. The callback approach ensures the host can filter, sample, or drop plugin spans and metrics before they reach the OTLP exporter — plugins cannot emit telemetry that bypasses the host.

#### Inbound Trace Context

> **OPEN — to be resolved in this revision.** Today a plugin has no way to *consume* an incoming
> `trace_id` / `parent_span_id`. If a trace enters a plugin, the plugin starts a second, unrelated
> trace. This revision must define how the trace ID, parent span ID, and trace flags cross the FFI
> boundary and are rehydrated into a usable standard span context inside the plugin runtime.
>
> Note this changes the scope of the design: propagating externally supplied trace context into
> source plugins was previously listed as a non-goal, and is now in scope. See
> [01 — Tracing](01-tracing.md) for the trace-rooting policy that depends on it.

#### Which Plugin Types Get Telemetry

**DECIDED — telemetry is a standard capability of every plugin kind, structured as three tiers: a
universal baseline every plugin gets, a per-kind standard set, and whatever the plugin author adds
on top. All three land in the same recorder.**

##### The plugin kinds

Eight extension points exist. They do **not** all use the same delivery path, because they do not
all cross the FFI boundary:

| Plugin kind | Registered on `DrasiLibBuilder` as | Loadable as cdylib? |
|---|---|---|
| Source | `with_source(impl SourceTrait)` | yes — `SourcePluginVtable` |
| Reaction | `with_reaction(impl ReactionTrait)` | yes — `ReactionPluginVtable` |
| Bootstrap provider | `set_bootstrap_provider(Arc<dyn BootstrapProvider>)` on the source | yes — `BootstrapPluginVtable` |
| Identity provider | `with_identity_provider(Arc<dyn IdentityProvider>)` | yes — `IdentityProviderPluginVtable` |
| Secret store | `with_secret_store_provider(Arc<dyn SecretStoreProvider>)` | yes — `SecretStorePluginVtable` |
| Index backend | `with_index_provider(Arc<dyn IndexBackendPlugin>)` | **no** — compile-time only |
| State store | `with_state_store_provider(Arc<dyn StateStoreProvider>)` | **no** — compile-time only |
| WAL provider | `with_wal_provider(Arc<dyn WalProvider>)` | **no** — compile-time only |

Two facts drive the whole design:

1. **Every kind arrives at `DrasiLibBuilder` as a trait object.** Whether a source was linked in
   statically or loaded from a `.so` and wrapped in a host-sdk `SourceProxy`, drasi-lib receives a
   `Box<dyn SourceTrait>`. This is a single, universal chokepoint that already exists.
2. **Only sources and reactions receive a host context.** `initialize_fn(state, ctx: *const
   FfiRuntimeContext)` appears on `SourceVtable` and `ReactionVtable` and on no other vtable.
   Bootstrap providers, identity providers and secret stores are pure factories — the host calls
   `create_*_fn(config_json)` and gets back a vtable with one or two methods. **There is no point
   at which the host hands them `instance_id`, `component_id`, or a callback.**

Fact 1 is what makes tiers 1 and 2a possible for all eight kinds; fact 2 is what dictates how
tiers 2b and 3 are transported, and it rules out the obvious approach — see
[How tiers 2b and 3 reach the recorder](#how-tiers-2b-and-3-reach-the-recorder).

##### The three tiers

Every plugin gets metrics in three tiers. Tiers 1 and 2 are **guaranteed** — they exist for a
plugin whose author wrote no instrumentation at all. Tier 3 is the author's own, and is optional.

| Tier | What | Who emits it | Applies to |
|---|---|---|---|
| **1 — Universal baseline** | Same handful of metrics for every plugin, whatever its kind | drasi-lib, automatically | all 8 kinds |
| **2 — Per-kind standard set** | Metrics meaningful for *that* kind: source metrics, reaction metrics, bootstrapper metrics, … | drasi-lib where derivable; plugin where not | all 8 kinds |
| **3 — Plugin-author metrics** | Whatever the author considers useful about their own internals | the plugin, opt-in | any kind |

**All three tiers land in the same recorder.** For a statically linked plugin the `metrics` macros
resolve to the host's global recorder directly; for a cdylib plugin they resolve to the plugin's
own recorder, which is an FFI bridge that forwards to the host's. The destination is identical
either way, so a dashboard cannot tell how a plugin was linked — which is the point.

##### Tier 1 — the universal baseline

drasi-lib wraps each registered trait object in a decorator at the builder. The decorator
implements the same trait, forwards every call, and records around it:

```rust
// Applied inside with_source / with_reaction / with_identity_provider / …
pub fn with_source(mut self, source: impl SourceTrait + 'static) -> Self {
    let instrumented = InstrumentedSource::new(Box::new(source));
    self.source_instances.push((Box::new(instrumented), HashMap::new()));
    self
}
```

This yields, uniformly and with no plugin author involvement:

| Metric | Type | Labels |
|---|---|---|
| `drasi.plugin.up` | gauge | `plugin_kind`, `component_kind`, `component_id` |
| `drasi.plugin.calls` | counter | `plugin_kind`, `component_kind`, `component_id`, `operation` |
| `drasi.plugin.call_duration_seconds` | histogram | `plugin_kind`, `component_kind`, `component_id`, `operation` |
| `drasi.plugin.errors` | counter | `plugin_kind`, `component_kind`, `component_id`, `operation`, `error_kind` |

`operation` is the trait method — `start`, `stop`, `subscribe`, `enqueue_query_result`,
`get_credentials`, `get_secret`, `bootstrap`, `get`, `set`, `delete`, `append`, `read_from`. Four
metrics describe every plugin in the process, so one dashboard panel and one alert rule cover all
of them regardless of kind.

Three properties make the builder the right place for this:

- **It is the only path that reaches index, state-store and WAL plugins at all**, since those never
  cross FFI and so have no other channel.
- **It is identical for static and dynamic plugins.** The decorator sits above the
  static-vs-`SourceProxy` distinction, so there is one implementation rather than two.
- **Attribution is complete.** The builder knows the component id and kind, so every tier 1 metric
  is fully labelled — including for the three kinds that have no `FfiRuntimeContext`.

> **Do not put this decorator in `drasi-host-sdk`.** Wrapping the FFI proxies there would instrument
> dynamic plugins only, and statically linked plugins — which is how Drasi Server ships most
> components today — would silently report nothing.

##### Tier 2 — the per-kind standard set

Tier 1 is deliberately semantic-free: it knows a call happened, not what it meant. Tier 2 adds the
metrics that are meaningful for a specific kind, so that every source reports the same things as
every other source and a Postgres source can be compared against a Kafka one.

Tier 2 splits by whether the value is visible from outside the plugin:

**2a — derivable at the boundary.** drasi-lib computes these from the call it is already wrapping,
so they are automatic and every implementation of that kind reports them identically:

| Kind | Tier 2a metrics |
|---|---|
| Source | `drasi.source.subscriptions`, `drasi.source.active_subscriptions` |
| Reaction | `drasi.reaction.results_processed`, `drasi.reaction.bootstraps` |
| Bootstrap provider | `drasi.bootstrap.runs`, `drasi.bootstrap.duration_seconds`, `drasi.bootstrap.elements_streamed` |
| Identity provider | `drasi.identity.credential_requests`, `drasi.identity.request_duration_seconds` |
| Secret store | `drasi.secret_store.requests`, `drasi.secret_store.request_duration_seconds` |
| Index backend | `drasi.index.operations`, `drasi.index.operation_duration_seconds` |
| State store | `drasi.state_store.operations`, `drasi.state_store.operation_duration_seconds` |
| WAL provider | `drasi.wal.appends`, `drasi.wal.append_duration_seconds` |

The last three rows are exactly the **interaction metrics** that
[02 — Metrics §6.2](02-metrics.md#62-storage-backend-statistics) argues only Drasi can produce, and
[A.10](02-metrics.md#a10-storage-index-state-store-and-wal) catalogues. They are not a separate
mechanism — storage backends are plugins, and the decorator is where their interaction metrics come
from.

**2b — requires plugin cooperation.** Some per-kind metrics are standard in name and meaning but
cannot be observed from outside: whether a connection is currently alive, how far behind an
upstream log the plugin is, how large a batch it just fetched. drasi-lib **declares** these as part
of the kind's contract and the SDK provides the pre-named handles, but only the plugin can supply
values:

| Kind | Tier 2b metrics |
|---|---|
| Source | `drasi.source.connected`, `drasi.source.reconnects`, `drasi.source.batch_size`, `drasi.source.upstream_lag_seconds`, `drasi.source.replication_lag` |
| Reaction | `drasi.reaction.connected`, `drasi.reaction.delivery_duration_seconds`, `drasi.reaction.delivery_attempts`, `drasi.reaction.batch_size` |

A plugin that does not populate them simply has no series for them, which is distinguishable from
a value of zero. The 2a/2b split matters because it determines what an operator may *rely* on: 2a
is guaranteed for every plugin of that kind, 2b is best-effort per implementation.

The SDK exposes tier 2b as a per-kind struct of pre-registered handles — `SourceMetrics`,
`ReactionMetrics` — so the author fills in values rather than inventing names. See
[Plugin Developer Experience](#plugin-developer-experience-transparent-bridge) for a worked
example.

##### Tier 3 — plugin-author metrics

Anything else the author wants to measure about their own internals — WAL parse time, change-feed
decoding, retry loops, cache hits. The author uses the standard `metrics` macros and the SDK routes
them to the same recorder as tiers 1 and 2.

##### How tiers 2b and 3 reach the recorder

Tiers 1 and 2a are emitted by drasi-lib itself, so they need no transport. Tiers 2b and 3 originate
*inside* the plugin, and for a cdylib plugin that means crossing FFI.

The transport is **library-scoped, not component-scoped** — and this is the part that makes the
guarantee hold for every kind. `FfiPluginRegistration` already carries library-wide setters that
the host calls once per loaded `.so`:

```rust
pub struct FfiPluginRegistration {
    // …
    pub set_log_callback:
        extern "C" fn(ctx: *mut c_void, callback: LogCallbackFn),
    pub set_lifecycle_callback: extern "C" fn(ctx: *mut c_void, callback: LifecycleCallbackFn),
    pub set_config_resolver: extern "C" fn(ctx: *mut c_void, callback: ConfigResolverFn),
    pub set_log_level: extern "C" fn(level: FfiLogLevelFilter),
}
```

`set_log_callback` stores the callback and context in plugin-global atomics and installs the
plugin's `tracing` subscriber. Because that state is library-global rather than per-component,
**logging already reaches every plugin kind in the cdylib** — an identity provider's
`tracing::warn!()` is forwarded today even though it never sees an `FfiRuntimeContext`. Metrics get
the same treatment:

```rust
    /// Appended for SDK <next-minor>. Host installs an FFI-backed recorder as the
    /// plugin library's global `metrics` recorder.
    pub set_metrics_recorder:
        extern "C" fn(ctx: *mut c_void, callback: MetricsCallbackFn),
```

The plugin SDK's handler installs an `FfiMetricsRecorder` as the cdylib's global `metrics`
recorder, so `metrics::counter!()` anywhere in the plugin — in any plugin kind — is forwarded to
the host. This is the concrete mechanism behind Part 3 above, and it is why plugin metrics do
**not** flow through the recorder stack an embedder installs in the host: the plugin links its own
copy of the `metrics` crate and has its own global slot.

Had this been hung off `FfiRuntimeContext` instead — the natural-looking place, since the
per-component log callback lives there — tiers 2b and 3 would have been available to sources and
reactions only.

**ABI rule.** New fields append to the end of `FfiPluginRegistration` and the host gates access on
the plugin's reported `sdk_version`, exactly as `identity_provider_plugins` and `set_log_level`
already do — reading a trailing field from a plugin that allocated the older, smaller struct is
undefined behaviour. `validate_plugin_metadata` additionally requires an exact `major.minor` match,
so an SDK-version bump rejects stale plugins outright; the gate covers the case where a plugin
exports no metadata symbol at all.

##### Attribution: what plugin-emitted metrics cannot label

`FfiLogEntry` carries `instance_id` and `component_id`, but they are populated *from
`FfiRuntimeContext` during `initialize`* and are documented as "empty if not yet initialized". For
bootstrap, identity and secret-store plugins that is permanent, because those kinds never receive
a context. The same limit applies to metrics.

| Metric source | `plugin_kind` | `component_id` |
|---|---|---|
| Tier 1 / 2a — emitted by drasi-lib, any kind | yes | **yes** |
| Tier 2b / 3 — emitted by a source or reaction | yes | yes |
| Tier 2b / 3 — emitted by a bootstrap / identity / secret-store plugin | yes | **no** |

This is acceptable rather than ideal: the affected kinds are typically configured once per
deployment, and tiers 1 and 2a supply the fully-labelled view for every call into them. Closing the
gap properly means adding an `initialize_fn` to those three vtables, which is a per-kind ABI change
and is deferred.

##### Namespace governance

Tier 3 is open-ended, so it is the one place a third-party plugin could collide with a Drasi metric
name or squat on `drasi.source.*`. Two rules:

- **The SDK issues pre-labelled handles.** A plugin obtains its tier 2b and tier 3 handles from an
  SDK-provided emitter that has already captured the plugin kind and (where available) the
  component id. This solves naming and attribution together, and is why a plugin author does not
  hand-write labels.
- **The bridge enforces the prefix.** `FfiMetricsRecorder` prefixes anything a plugin emits that is
  not a declared tier 2b name, so tier 3 metrics land under `drasi.plugin.<plugin_kind>.*` and
  cannot shadow a first-party name.

> **Asymmetry to resolve.** Prefix enforcement in the bridge only applies to cdylib plugins — a
> statically linked plugin calling `metrics::counter!("drasi.source.events_dispatched")` reaches
> the global recorder directly with nothing in between. Either the SDK emitter becomes the only
> supported way to emit from a plugin, or static plugins are governed by convention alone. This is
> [02 — Metrics Open Issue 3](02-metrics.md#open-issues) and is not yet settled.

> **Consequence for naming.** The convention `drasi.<component_type>.<plugin_kind>.<metric>`
> assumes a pipeline component type, which identity providers and secret stores do not have. Tier 1
> therefore uses a single `drasi.plugin.*` family with `plugin_kind` and `component_kind` as
> **labels**, consistent with the `drasi.queue.*` decision in [02 — Metrics](02-metrics.md); tier 2
> uses a per-kind prefix (`drasi.source.*`, `drasi.bootstrap.*`, …). To be confirmed against the
> naming convention below.

##### Enablement

Emission is always optional for the plugin author — a plugin that instruments nothing is valid, and
tiers 1 and 2a still report it. `set_log_level` is the precedent for host-controlled filtering: the
host reports its effective level and the plugin drops records *before formatting or forwarding
them*. A metrics equivalent should follow, so that a disabled metric costs a filter check inside
the plugin rather than an FFI crossing.

### Plugin Developer Experience: Transparent Bridge

Plugin developers use standard Rust `tracing` and `metrics` macros — no custom API needed. The plugin SDK transparently installs bridge implementations that intercept standard calls and forward them to the host:

| Signal | Bridge installed by plugin SDK | Plugin dev uses | How it forwards |
|--------|-------------------------------|----------------|-----------------|
| **Logs** (existing) | `FfiTracingLayer` | `tracing::info!()`, `tracing::error!()` | Intercepts events → `FfiLogEntry` → `LogCallbackFn` |
| **Metrics** (new) | `FfiMetricsRecorder` | `metrics::counter!()`, `metrics::histogram!()` | Intercepts recordings → `FfiMetricEntry` → `MetricsCallbackFn` |
| **Traces** (new) | Extended `FfiTracingLayer` | `tracing::info_span!()` | Intercepts span open/close → `FfiCompletedSpan` → `SpanCallbackFn` |

For trace context injection, the host passes `trace_id` + `parent_span_id` explicitly via FFI function arguments (e.g., a `trace_context: FfiTraceContext` parameter on the vtable calls). Task-local storage cannot be used here because the plugin runs on its own tokio runtime, and task-locals do not cross runtime boundaries. The bridge layer in the plugin reads the trace context from the FFI argument when a span is created and includes it in the `FfiCompletedSpan` sent back to the host. This is fully transparent to the plugin developer.

**Worked example: adding metrics to a new source plugin.**

Suppose someone is writing a source that subscribes to a remote change feed and reacts to frames as
the upstream system pushes them. Before they write a single line of instrumentation they already
get tier 1 (`drasi.plugin.up`, `.calls`, `.call_duration_ns`, `.errors`) and tier 2a
(`drasi.source.subscriptions`, `drasi.source.active_subscriptions`), because drasi-lib emits those
from the decorator wrapping their plugin. What follows is only what they add on top.

**Step 1 — declare the handles.** Both tier 2b and tier 3 handles are registered once and cached on
the struct, never created per event. This matters: the `metrics` macros build the metric `Key`
*before* consulting the recorder, so a per-event `counter!("…", "source_id" => id)` allocates on
every event even when nothing is collecting.

```rust
use drasi_plugin_sdk::prelude::*;
use drasi_plugin_sdk::metrics::SourceMetrics;
use metrics::{Counter, Histogram};

struct ChangeFeedMetrics {
    /// Tier 2b — names and labels come from the SDK, values from us.
    std: SourceMetrics,
    /// Tier 3 — specific to this plugin.
    frames_received: Counter,
    decode_duration_ns: Histogram,
    heartbeats: Counter,
}

pub struct ChangeFeedSource {
    id: String,
    config: ChangeFeedConfig,
    metrics: OnceLock<ChangeFeedMetrics>,
}
```

**Step 2 — obtain them from the runtime context.** `context.metrics()` returns an emitter that has
already captured `plugin_kind` and `component_id`, so the author never writes a label. This is also
what keeps tier 3 names inside the plugin's own namespace.

```rust
#[async_trait]
impl Source for ChangeFeedSource {
    async fn initialize(&self, context: SourceRuntimeContext) {
        let m = context.metrics();
        let _ = self.metrics.set(ChangeFeedMetrics {
            std:                m.source(),                    // drasi.source.*
            frames_received:    m.counter("frames_received"),  // drasi.plugin.changefeed.frames_received
            decode_duration_ns: m.histogram("decode_duration_ns"),
            heartbeats:         m.counter("heartbeats"),
        });
    }
```

**Step 3 — record.** Tier 2b is populated exactly like tier 3; the only difference is that its names
are part of the source contract, so a dashboard built for one source works for this one too.

```rust
    async fn start(&self) -> Result<()> {
        let m = self.metrics.get().expect("initialize runs first");
        let mut stream = self.subscribe_upstream().await?;
        m.std.connected.set(1.0);                            // tier 2b — declared by drasi-lib

        while let Some(frame) = stream.next().await {
            let frame = match frame {
                Ok(frame) => frame,
                Err(e) => {
                    m.std.connected.set(0.0);
                    // No error counter here: tier 1 already counts this call's failure.
                    tracing::warn!(error = %e, "upstream stream interrupted");
                    stream = self.resubscribe().await?;
                    m.std.reconnects.increment(1);           // tier 2b
                    m.std.connected.set(1.0);
                    continue;
                }
            };

            m.frames_received.increment(1);                  // tier 3 — ours
            if frame.is_heartbeat() {
                m.heartbeats.increment(1);                   // tier 3 — ours
                continue;
            }

            // How far behind the upstream event time we are.
            m.std.upstream_lag_ns.set(frame.age().as_nanos() as f64);   // tier 2b

            let started = Instant::now();
            let changes = self.decode(frame)?;
            m.decode_duration_ns.record(started.elapsed().as_nanos() as f64);
            m.std.batch_size.record(changes.len() as f64);   // tier 2b

            for change in changes {
                self.dispatch_change(change).await?;
            }
        }
        Ok(())
    }
}
```

Three things worth noting about what the author did *not* write:

- **No labels.** `source_id` / `plugin_kind` are attached by the emitter, so they cannot be
  forgotten, misspelled, or made unbounded.
- **No error counter for the interrupted stream.** The tier 1 decorator already recorded
  `drasi.plugin.errors{operation="start"}` if the call returns `Err`. A plugin should add an error
  counter only for failures it *handles internally* and never surfaces to the host — which is
  exactly the case above, where the loop resubscribes and continues, so `drasi.source.reconnects`
  plus a `tracing::warn!` carry the detail instead.
- **No exporter, no recorder, no configuration.** Whether these land in Prometheus or OTLP, and
  whether this plugin is statically linked or loaded from a `.so`, is decided by the host.

**For plugin kinds with no runtime context** — bootstrap, identity, secret store — there is no
`context.metrics()`, so the SDK exposes a library-scoped emitter instead. It carries `plugin_kind`
but not `component_id`, per the attribution table above:

```rust
use drasi_plugin_sdk::metrics::plugin_metrics;

let m = plugin_metrics();                    // no component_id available
let cache_hits = m.counter("cache_hits");    // drasi.plugin.vault.cache_hits
```

**The raw macros still work.** `metrics::counter!("anything")` reaches the same recorder — the
emitter is a convenience and a governance mechanism, not a gate. For a cdylib plugin the bridge
still prefixes the result; for a statically linked plugin it does not, which is the asymmetry
flagged under [Namespace governance](#namespace-governance).

This all works identically for built-in and cdylib plugins. For built-in plugins the macros go
directly to the host's subscriber/recorder. For cdylib plugins, the bridge implementations
intercept and forward via callbacks. The plugin developer never sees the difference.

> **OPEN — developer guide content.** The plugin developer guide must additionally cover: how a
> plugin author attaches custom spans to the **supplied span context**, and how Drasi-managed
> instrumentation connects automatically when the provided machinery is used.

### Enablement, Filtering and Observability Profiles

> **OPEN — to be resolved in this revision.** Three related questions:
>
> 1. **Component enablement and propagation.** How are logging, tracing and metrics enabled,
>    disabled, filtered, and *propagated into subcomponents* at construction time? The worked
>    example is RocksDB: are its metrics always emitted, or must they be turned on when drasi-lib
>    initializes the index? (Statistics collection has a real cost in RocksDB.) Whatever we decide
>    must generalize to Redis/Garnet, which expose a different surface.
> 2. **Observability profiles.** Rather than configuring every signal individually, define
>    predefined profiles — e.g. `basic`, `debug`, and a persistence-focused one — with filters by
>    namespace/component (RocksDB, queries, sources) and by depth.
> 3. **Direct vs curated.** For a component like RocksDB that exposes hundreds of native metrics,
>    does the user get them directly, or does Drasi curate a subset and re-emit it under the
>    `drasi.` namespace? See [02 — Metrics](02-metrics.md).

### Naming and Namespacing Conventions

**DECIDED — dot-namespaced, lowercase, with the unit and the counter suffix written into the leaf.
Service identity lives in a resource attribute, never in the metric name.**

The two conventions worth following disagree with each other, so the choice turns on one
implementation fact about our stack rather than on taste.

#### What the established conventions actually say

| Rule | OpenTelemetry semconv | Prometheus |
|---|---|---|
| Separator | dot for namespaces, `snake_case` within a component (`http.response.status_code`) | `_` throughout |
| Prefix | namespace required; app developers use their application name | single-word application prefix (`prometheus_`, `process_`) |
| Unit in the name | **No** — units live in instrument metadata | **Yes** — required suffix, plural (`_seconds`, `_bytes`) |
| Base unit | seconds for durations | seconds for durations; never ms/ns |
| Counter suffix | **Never `_total`** — "confusing in delta backends" | **`_total`** for accumulating counts |
| Pluralization | namespaces never; names only for countable instances (`system.disk.operations`) | not prescribed |
| Duration naming | `{operation}.duration` | `{thing}_duration_seconds` |

They agree on more than they disagree: lowercase, an application prefix, base units, no label names
baked into metric names, and that `sum()`/`avg()` across a metric's labels should be meaningful.
They disagree on exactly two points — **unit in the name, and `_total`** — and Prometheus states its
reasoning explicitly: type and unit information is needed when reading PromQL in plain YAML
(alerting and recording rules), and omitting units causes collisions such as `process_cpu` meaning
seconds in one place and milliseconds in another.

#### The fact that decides it

Under the OpenTelemetry SDK the disagreement is a non-issue: you write the OTel name, set the unit
in metadata, and the Prometheus exporter mechanically produces the Prometheus name — replacing `.`
with `_`, appending the unit word, and appending `_total` to monotonic sums. Both conventions are
satisfied because the exporter translates between them.

**Drasi does not emit through the OpenTelemetry SDK.** It emits through the `metrics` facade
([02 — Metrics §2](02-metrics.md#2-collection-architecture)), and `metrics-exporter-prometheus`
performs *no* such translation. Its documented name handling is limited to replacing invalid
characters with `_`; its `formatting` module offers only `sanitize_metric_name`, `write_help_line`,
`write_type_line` and `write_metric_line`. There is no unit suffixing and no `_total` appending.
`metrics::Unit` exists and `describe_histogram!` records it, but the Prometheus exporter does not
use it to build the name.

So **the name we write is, after `.` → `_` substitution, the name that ships**. Nothing downstream
will add what we leave out. If we follow OTel's "no unit in the name" rule, Prometheus receives
`drasi_query_engine_duration` — no unit, no type — which is precisely the ambiguity Prometheus
warns about, and the `Unit::Seconds` we carefully declared is silently discarded.

#### The convention

1. **Lowercase, dot-separated namespaces, `snake_case` within each component.** This is OTel's
   general naming rule and it survives sanitization intact: `drasi.query.engine_duration_seconds`
   renders as `drasi_query_engine_duration_seconds`.
2. **`drasi` is the root namespace** for everything Drasi emits.
3. **Durations use seconds, as `f64`** — never `_ms` or `_ns`. Both conventions require base units.
   Sub-microsecond values are represented exactly by `f64` seconds, so no precision is lost.
4. **The unit is part of the leaf**, plural: `_seconds`, `_bytes`, `_ratio`. Unitless counts of
   discrete things (`drops`, `errors`, `frames`) take no unit suffix — Prometheus explicitly
   excludes countable things from this rule.
5. **Monotonic counters end in `_total`.** Gauges, histograms and UpDownCounters never do.
6. **Pluralize only counts of discrete instances.** `drasi.queue.drops_total` yes;
   `drasi.queue.depth` no. Namespaces are never pluralized.
7. **Also call `describe_*`** with a `Unit` and a description. It populates `# HELP`/`# TYPE`, and
   it keeps the metadata correct for an OTLP branch even though Prometheus ignores the unit.
8. **Labels never appear in names.** `drasi.source.events_total{source_id="x"}`, never
   `drasi.source.x.events_total`.
9. **`sum()` or `avg()` across a metric's labels must be meaningful.** This is the test that
   justifies one `drasi.queue.*` family labelled by `component_kind` rather than a family per
   component type.

> **This is a deliberate divergence from OTel semconv on rules 4 and 5**, and it is reversible.
> It is correct *because* our exporter is a passthrough. If Drasi ever emits through the OTel SDK,
> the suffixes must be removed at the same time — otherwise the SDK's exporter appends its own and
> produces `..._seconds_seconds` and `..._total_total`.

#### Namespace layout

| Namespace | Owner |
|---|---|
| `drasi.source.*`, `drasi.query.*`, `drasi.reaction.*` | pipeline stages |
| `drasi.pipeline.*` | metrics spanning the whole pipeline |
| `drasi.queue.*` | backpressure, labelled by `component_kind` |
| `drasi.component.*` | lifecycle, any component type |
| `drasi.plugin.*` | the universal plugin baseline (tier 1) |
| `drasi.plugin.<plugin_kind>.*` | plugin-author metrics (tier 3), prefix enforced by the bridge |
| `drasi.index.*`, `drasi.state_store.*`, `drasi.wal.*` | storage interaction |
| `drasi.index.rocksdb.*` | engine-native stats, kept engine-specific by design |
| `process_*` | **exception** — ecosystem-standard, no `drasi` prefix |

Engine-specific statistics keep an engine-specific namespace rather than being forced into a shared
name, following OTel's own reasoning for preferring `jvm.gc.*` over `gc.*`: implementations differ
enough that a shared name invites false comparison.

#### Service identity is a resource attribute, not a prefix

**`drasi-lib` and `drasi-server` are NOT differentiated in the metric name.** They are distinguished
by the OTel `service.name` resource attribute, or an equivalent global label on the Prometheus
exporter — which is what those mechanisms exist for.

The reason is comparability: `drasi.query.events_processed_total` should mean the same thing and be
chartable on the same panel whether the query ran inside Drasi Server or inside a user's own
binary. Encoding the host in the name makes that impossible and doubles the number of names for no
gain.

#### Span names

Spans follow the same lowercase dotted style but **carry no `drasi.` prefix and no unit rules**:
`source.dispatch`, `query.process`, `reaction.receive`. Span names must stay low-cardinality —
identity goes in fields (`query_id`, `source_id`), never in the name.

Namespacing for spans comes free from two mechanisms that already exist and need no convention:

- **`tracing` targets** are crate paths, so `drasi_lib`, `drasi_core` and plugin crates are
  filterable without any naming effort — this is how `EnvFilter` directives are written.
- **`service.name`** separates processes, as above.

#### Consequences

This settles the open question of units and supersedes the `_ns` convention inventoried in
[02 — Metrics §3.7](02-metrics.md#37-naming-and-unit-conventions-already-in-use). The existing
`ProfilingMetadata` fields stay in nanoseconds internally — only the *exported* metric converts, via
`as_secs_f64()`. It also closes
[02 — Metrics Open Issue 2](02-metrics.md#open-issues), because histogram buckets must now be
expressed in seconds.

### Enabling Telemetry as a drasi-lib Consumer

> **OPEN — to be resolved in this revision.** drasi-lib emits through facades but installs no
> backend, so telemetry is *not* on by default — an embedding application must install a subscriber
> and/or recorder. This section needs a concrete code example showing how a consuming application
> initializes the tracing provider, log provider and metrics recorder before constructing
> `DrasiLib`, plus a statement of what the actual default behaviour is with nothing installed.

### API Design

No changes to the public `DrasiLib` builder API, REST API, or CLI. The pipeline instrumentation itself is purely internal — all new tracing spans and metrics are emitted through facade crates and are transparent to callers.

There is one additive change to drasi-lib's public initialization surface: to let embedders (e.g., Drasi Server) compose `ComponentLogLayer` into their own multi-layer subscriber alongside an OTLP layer, drasi-lib will add a public `init_component_log_layer()` helper that returns the layer, and keep `init_default_subscriber()` (current `get_or_init_global_registry()` behavior) for simple embedders. This is a non-breaking, additive API change — existing callers of `get_or_init_global_registry()` continue to work unchanged.

### Phase Plan

> **OPEN — to be resolved in this revision.** Review direction was to take an "onion" approach:
> implement the most important observability capabilities first, then expand coverage, leveraging
> existing OpenTelemetry tooling and minimizing new infrastructure. This section needs an explicit
> phase 1 / later-phase breakdown across all three signals, including the definitive phase-1 metric
> list from [02 — Metrics](02-metrics.md).

### Alternatives Considered

#### 1. Use OpenTelemetry SDK Directly (Instead of Facade Crates)

Instrument drasi-lib directly with `opentelemetry` crate APIs (`tracer.start("span")`, `meter.u64_counter()`).

**Rejected because**: This would hard-couple drasi-lib to the OpenTelemetry SDK, requiring all embedding applications to use OTel. The facade approach (`tracing` + `metrics`) lets users choose any backend. This is also the approach used by the broader Rust ecosystem — libraries use facades, applications choose backends.

#### 2. Replace `log` Crate Usage with `tracing` Events Everywhere

Convert all existing `log::info!()`, `log::error!()` calls to `tracing::info!()`, `tracing::error!()`.

**Deferred**: This would be a nice cleanup but is not necessary for this design. The `tracing-log` bridge already forwards `log` events to the `tracing` subscriber. We can do this incrementally as we touch files.

## Security

No new security concerns. Tracing spans and metrics expose operational data (component IDs, event counts, latency) but not user data, credentials, or query content. Span fields use component identifiers that are already visible in logs today.

If an embedding application exports traces to an external collector (e.g., Jaeger, OTLP), the security of that transport is the application's responsibility — not drasi-lib's.

## Compatibility Impact

- **No breaking changes**: The public `DrasiLib` API is unchanged. No new required configuration.
- **New dependency**: `metrics 0.24` is added to `Cargo.toml`. This is a lightweight facade crate with no transitive dependencies beyond `portable-atomic`.
- **Behavioral change**: Applications that already have a `tracing::Subscriber` installed will see new spans (`source.dispatch`, `query.process`, `reaction.receive`) in their output. This is additive and should not break existing behavior.
- **Existing `log` crate usage**: Continues to work. `tracing-log` bridge is preserved.

## Supportability

### Telemetry

This design *is* the telemetry story for drasi-lib. After implementation, the following telemetry is available:

| Signal | What | How to enable |
|--------|------|---------------|
| Structured logs | `tracing::info!()` / `tracing::error!()` events with span context | Install any `tracing::Subscriber` (e.g., `tracing_subscriber::fmt::init()`) |
| Distributed traces | Nested spans across Source → Query → Reaction | Install `tracing-opentelemetry` layer with an OTLP/Jaeger exporter |
| Metrics | Counters, histograms, gauges | Install any `metrics::Recorder` (e.g., `metrics-exporter-prometheus`) |
| Per-component logs | Log streams per Source/Query/Reaction | Existing `ComponentLogLayer` — no changes needed |

### Verification

Signal-specific test plans live in [01 — Tracing](01-tracing.md) and [02 — Metrics](02-metrics.md).
Shared checks:

| Test | Scope | Approach |
|------|-------|----------|
| ComponentLogLayer compatibility | Integration | Existing tests for `subscribe_component_logs()` must continue to pass with the new spans in place |
| Near-zero cost when no backend | Unit / Bench | Process events without any subscriber/recorder installed; verify no panics, and benchmark the hot path under a counting allocator to confirm no per-event allocation from instrumentation |

> **Note on "near-zero cost".** Neither facade is literally zero-cost. With no backend installed, a
> disabled `tracing` callsite costs an atomic load and a branch, and a pre-registered `metrics`
> handle costs a no-op virtual call. When a backend *is* installed, no telemetry *export* work
> happens on the critical path — it is handed to an asynchronous batching worker. The benchmark
> above is what backs these claims.

## Open Issues

1. **`get_or_init_global_registry()` split**: To support Drasi Server composing `ComponentLogLayer` into its own multi-layer subscriber (with OTLP), we propose splitting `get_or_init_global_registry()` into two functions:
   - `init_component_log_layer()` — creates the registry, channel, and worker thread, returns the `ComponentLogLayer` for the caller to compose into their own subscriber
   - `init_default_subscriber()` — calls `init_component_log_layer()`, composes it with the `fmt` layer, and installs the global subscriber (same behavior as today)

   Simple embedders call `init_default_subscriber()` and get current behavior. Drasi Server calls `init_component_log_layer()`, adds the OTLP layer alongside it, and installs its own subscriber. This is a non-breaking change.

2. **Management-plane tracing scope**: Review agreed that management operations (e.g. creating a query) should be traced, but the Drasi Server design explicitly lists API-layer spans as a non-goal. The two documents must be reconciled — either management-plane spans are in phase 1, or they are documented and explicitly deferred.

3. **drasi-core tracing**: This document lists drasi-core instrumentation as out of scope, but drasi-core already emits standard `tracing` — it only needs wiring and namespace filtering. The Out of Scope entry should be corrected to say so.

## References

- [`tracing` crate](https://crates.io/crates/tracing) — Structured diagnostics facade for Rust
- [`metrics` crate](https://crates.io/crates/metrics) — Metrics facade for Rust
- [OpenTelemetry semantic conventions — naming](https://opentelemetry.io/docs/specs/semconv/general/naming/) — the basis for the dotted-namespace scheme
- [Prometheus — metric and label naming](https://prometheus.io/docs/practices/naming/) — the basis for base units and the `_total` suffix
