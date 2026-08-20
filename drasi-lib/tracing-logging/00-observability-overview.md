# Observability for drasi-lib — Overview and Shared Foundations

* Project Drasi - Ruokun Niu (@ruokun-niu)
* Last edited on August 19th, 2026

> **Document set.** This design is split across three documents. Read this one first — it covers
> the concerns shared by all telemetry signals.
>
> | Document | Covers |
> |----------|--------|
> | **00 — Overview and Shared Foundations** (this doc) | Objectives, terminology, the facade principle, the pipeline model, logging, the FFI callback-bridge mechanism, enablement and configuration, API design, phase plan |
> | [01 — Tracing](01-tracing.md) | Span hierarchy, span naming and namespacing, trace context propagation across tasks and FFI, trace rooting, plugin span authoring, ladder diagrams |
> | [02 — Metrics](02-metrics.md) | Metric definitions, collection architecture, sampling vs aggregation, environment and storage metrics, plugin metrics and metric naming |

## Overview

drasi-lib today uses the `log` crate for basic logging and a custom `ComponentLogLayer` (built on `tracing`) to route per-component logs to an internal registry. While this gives developers per-component log streams, there is no structured span hierarchy across the Source → Query → Reaction pipeline, no counters or histograms for operational metrics, and no way to export telemetry to external backends.

This design adds structured tracing spans and explicit metrics to drasi-lib's pipeline so that developers embedding the library can follow an event end-to-end. drasi-lib emits telemetry through the `tracing` and `metrics` facade crates; the embedding application decides where the data goes by installing subscribers and recorders.

## Terms and Definitions

| Term | Definition |
|------|------------|
| Facade crate | A Rust crate that defines a logging/metrics API but defers the implementation to the consumer (e.g., `tracing`, `metrics`, `log`). The facade provides only the call sites (`info_span!()`, `counter!()`); something else must occupy the process-global slot to do anything with them. With that slot empty the calls are **near-zero cost** — a disabled span callsite is an atomic load and a branch, and a pre-registered metric handle is a no-op virtual call. They do not compile away entirely. |
| **Backend** | Umbrella term for **whatever occupies the process-global slot** — a `tracing::Subscriber` or a `metrics::Recorder`. Used throughout this document in the cost sense: *"no backend installed"* means nothing is in that slot, **not** that no collector is reachable.|
| Span | A `tracing::Span` representing a unit of work with a start time, end time, and structured fields. Spans nest to form a tree. |
| Subscriber | A `tracing::Subscriber` implementation that receives span/event data and decides what becomes of it — formatting to stdout, routing to `ComponentLogLayer`, handing it to an exporter, or discarding it. Installed by the embedding application, not drasi-lib. It is a **backend**, not a destination: it may write nowhere at all. |
| Recorder | A `metrics::Recorder` implementation that receives counter/histogram/gauge data and decides what becomes of it — storing it in a registry, exposing it for scrape, or forwarding it to an exporter. Installed by the embedding application, not drasi-lib. |
| Host | The original tokio runtime context that loads and manages plugins — drasi-lib's manager layer, plus whatever application embeds it (typically Drasi Server). The host runs the pipeline spans and owns the single `tracing` subscriber and `metrics` recorder. **Not** to be confused with the *host SDK* (`drasi-host-sdk`), which is the crate providing the plugin-loading machinery, or with a plugin's own isolated tokio runtime. |
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

- **Adding instrumentation to drasi-core**: new spans, index-operation spans, or converting drasi-core's
  `log` call sites to `tracing` are drasi-core changes and are not designed here. What drasi-core
  *already* emits is **not** out of scope — it is inherited, and the next subsection says what that is.
- **Drasi Server changes**: How Drasi Server wires up subscribers/recorders for these new traces is a separate design document.
- **Custom source/reaction plugin internal instrumentation**: Plugin authors can add their own spans inside their plugin. This design provides the FFI infrastructure (trace context propagation, metrics forwarding) to make plugin telemetry visible to the host — see [Plugin Telemetry Across FFI](#plugin-telemetry-across-ffi) below.
- **Log format changes**: The `ComponentLogLayer` output format and API remain unchanged.

## Design

### Signal Model

The design adds two layers of instrumentation to drasi-lib's existing pipeline:

1. **Tracing spans** at pipeline stage boundaries — each stage gets a named span with structured fields. Spans nest naturally as an event flows through Source → Query → Reaction.
2. **Metrics** at the same boundaries — counters for throughput and errors, histograms for latency, and gauges for queue depth.

drasi-lib already has a `ProfilingMetadata` struct (`profiling/mod.rs`) that stamps nanosecond-precision timestamps at each pipeline stage, and a Profiler Reaction plugin that computes running statistics (mean, p50, p95, p99) over sampled events.

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

### What drasi-core already emits

Interval D — `query.process` — is spent inside drasi-core, so it matters what the engine contributes
on its own.

| | State in `drasi-core` today |
|---|---|
| **Spans** | **10 `#[tracing::instrument]` sites.** Seven in `core/src/query/continuous_query.rs` — including `process_source_change` (`:105`) — and three in `core/src/path_solver/mod.rs`. All are `skip_all`, `level = "debug"`, most with `err` |
| **Events** | **None on the `tracing` facade.** Every logging call site uses the `log` crate. `core/Cargo.toml` declares *both* `log` (`:38`) and `tracing` (`:39`) |
| **`drasi-middleware`** | No `tracing` dependency at all. Thirty call sites, all `log::*` |

This design proposes **no changes to drasi-core** — that is [out of scope](#out-of-scope). Core's
spans nest under `query.process` automatically and its `log` records already reach the subscriber
through the existing `tracing_log::LogTracer` bridge, so the engine is inherited as-is.

### Logging

#### Interaction with the Existing ComponentLogLayer

The `ComponentLogLayer` is preserved unchanged. It operates as a `tracing_subscriber::Layer` and intercepts tracing events based on span context (`component_id`, `component_type`). The new spans we add carry these same fields, so:

- Events emitted inside a `source.dispatch` span that carries `component_id` and `component_type = "source"` will be automatically routed to the correct component's log stream by `ComponentLogLayer`.
- The `ComponentLogRegistry` API (`subscribe_component_logs()`, `subscribe_component_events()`) continues to work as before.
- If the embedding application adds additional `tracing::Subscriber` layers (e.g., `tracing-opentelemetry`), spans flow to both `ComponentLogLayer` AND the external backend. This is standard `tracing` layer composition.

#### What This Design Changes About Logging

**Nothing about how log records are produced, routed or delivered.** The two-destination path a
plugin log takes today — synchronous over `LogCallbackFn` to the host, then out through
`log::log!()` to the embedder's subscriber *and* into `ComponentLogRegistry` for the live tail —
is preserved exactly. The registry's retention, its best-effort semantics, and the REST/CLI/VS Code
APIs built on it are unchanged.

Two additions to `FfiLogEntry` are needed, both `repr(C)` trailing-field appends under the
established `sdk_version` gate:

| Change | Why |
|---|---|
| Append `component_type` | 🐛 **Fixes a live misattribution bug.** `FfiLogEntry` (`callbacks.rs:110`) carries no `component_type`, so the host hardcodes `ComponentType::Source` (`host-sdk/src/callbacks.rs:~239`, with a `TODO`). Logs from a **reaction** plugin appear in the wrong component's stream. The plugin already extracts the correct value for routing (`tracing_bridge.rs:109`) and then discards it |
| Append `trace_id` + `span_id` | Log-to-trace correlation — jump from a slow span to the lines emitted inside it. See [01 — Tracing](01-tracing.md#boundary-call-inventory) |

### Plugin Telemetry Across FFI

When sources and reactions are loaded as cdylib dynamic plugins (via `drasi-host-sdk`), they run in a separate shared library with their own tokio runtime and their own `tracing` global subscriber. This creates an FFI boundary that the normal span propagation approach (carrying `tracing::Span` handles through channels, as described in [01 — Tracing](01-tracing.md)) cannot cross.

The reason is that `tracing::Span` handles are tied to the subscriber that created them — they reference internal storage in the subscriber's registry. You cannot pass a `tracing::Span` across the FFI boundary the way you can pass it through an async channel within the same process.

Today, only flat log messages cross the FFI boundary — `FfiTracingLayer` captures tracing events, flattens them to `FfiLogEntry` (level, message, component IDs), and delivers them via a C callback. No span trees, trace IDs, or metrics cross. This means plugin-internal work (e.g., Postgres WAL parsing, change-feed decoding, MQTT publishing) is completely invisible to the host's tracing and metrics systems.

#### Proposed Approach: Centralized Callback Bridge

All three telemetry signals — logs, metrics, and traces — use the same architecture: the plugin serializes telemetry data into a flat C-compatible struct and sends it to the host via a callback function pointer on the vtable. The host receives it and routes it through its own subscriber/recorder. This gives the host full control over filtering, sampling, and export.

In this context, the **host** is the application that loads and manages plugins — i.e., drasi-lib's manager layer (or Drasi Server wrapping it). The host runs the pipeline spans (`source.dispatch`, `query.process`, etc.) and owns the single `tracing` subscriber, `metrics` recorder, and OTLP exporter. Plugins are the cdylib shared libraries loaded into the host process — they do not have their own exporters.

**Part 1: Host-side instrumentation** — all pipeline spans and metrics run on the host side. These are automatic and require no plugin code.

**Part 2: Trace context in, completed spans out** — every FFI crossing uses the same fixed-size
context value:

```rust
#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct FfiTraceContext {
   pub trace_id: [u8; 16],
   pub span_id: [u8; 8],
   pub trace_flags: u8,
}
```

`span_id` is always the sender's current span and becomes the receiver's parent. `trace_flags`
carries the W3C sampling decision. An all-zero value means no parent context.

The value is passed directly as an `FfiTraceContext` parameter when a real FFI call exists
(bootstrap, identity provider, secret store, lifecycle calls). Data events have no such call, so
the same value is appended as a trailing field on their ABI envelopes:

```rust
#[repr(C)]
pub struct FfiSourceEvent {
   // Existing fields remain unchanged.
   pub trace_context: FfiTraceContext,
}

#[repr(C)]
pub struct FfiQueryResult {
   // Existing fields remain unchanged.
   pub trace_context: FfiTraceContext,
}
```

When a plugin span closes, the plugin serializes it to an `FfiCompletedSpan`, including
`trace_flags`, and sends it back via `SpanCallbackFn`. The host feeds it into its exporter, so
plugin spans nest under the pipeline trace and preserve the sampling decision.

> The trace-specific contract and implementation gaps are in
> [01 — Tracing](01-tracing.md#current-sdk-gaps).
>
> ⚠️ One consequence needs review sign-off: `tracing` has **no facade API for an already-finished
> span**, so the host cannot hand a completed plugin span to the embedder's subscriber. This is
> resolved with a narrow `PluginSpanSink` trait and is a **documented exception to
> [Requirement 1](#requirements)**.

**Part 3: Plugin metrics forwarding** — add `FfiMetricEntry` + `MetricsCallbackFn`, installed via a
`set_metrics_recorder` setter on `FfiPluginRegistration` (library-scoped, alongside
`set_log_callback`) rather than on any per-kind vtable. Plugin-sdk installs an `FfiMetricsRecorder`
as the plugin library's global `metrics` recorder, which proxies to the host. Scoping it to the
library rather than to `FfiRuntimeContext` is what makes it reach every plugin kind — see
[Which Plugin Types Get Telemetry](#which-plugin-types-get-telemetry).

**Why centralized export matters**: With third-party source plugins, Drasi needs control over what telemetry is exported. The callback approach ensures the host can filter, sample, or drop plugin spans and metrics before they reach the OTLP exporter — plugins cannot emit telemetry that bypasses the host.

#### Which Plugin Types Get Telemetry

**DECIDED — telemetry is a standard capability of every plugin kind.** Eight extension points exist
(source, reaction, bootstrap provider, identity provider, secret store, index backend, state store,
WAL provider), and two facts shape how telemetry reaches them:

1. **Every kind arrives at `DrasiLibBuilder` as a trait object**, whether linked statically or
   wrapped in a host-sdk proxy — a single universal chokepoint that already exists. This is where
   drasi-lib's own instrumentation decorator goes.
2. **Only sources and reactions receive an `FfiRuntimeContext`.** Bootstrap providers, identity
   providers and secret stores are pure factories, so anything scoped to a component context cannot
   reach them; plugin-emitted telemetry must be **library-scoped** instead.

Three kinds — index backend, state store and WAL provider — are `lib`-only crates with **no FFI
path at all**, which is why the decorator lives at the builder rather than in `drasi-host-sdk`.

> **Full model in [02 — Metrics §8](02-metrics.md#8-plugin-metrics)**: the three tiers (universal
> baseline, per-kind standard set, plugin-author metrics), how tiers 2b and 3 cross FFI via
> `set_metrics_recorder`, the attribution limits, and namespace governance. Plugin **spans** are in
> [01 — Tracing](01-tracing.md#plugin-spans).

##### Library-scoped callbacks, and the ABI rule

Telemetry that originates *inside* a cdylib plugin has to cross FFI, and the transport is
**library-scoped, not component-scoped** — which is what makes it reach every plugin kind rather
than just sources and reactions. `FfiPluginRegistration` already carries library-wide setters that
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
`tracing::warn!()` is forwarded today even though it never sees an `FfiRuntimeContext`. The metrics
recorder (`set_metrics_recorder`) and the span callback follow the same pattern, for the same
reason: hanging them off `FfiRuntimeContext` instead would have limited them to sources and
reactions.

**ABI rule.** New fields append to the end of `FfiPluginRegistration` and the host gates access on
the plugin's reported `sdk_version`, exactly as `identity_provider_plugins` and `set_log_level`
already do — reading a trailing field from a plugin that allocated the older, smaller struct is
undefined behaviour. `validate_plugin_metadata` additionally requires an exact `major.minor` match,
so an SDK-version bump rejects stale plugins outright; the gate covers the case where a plugin
exports no metadata symbol at all.

##### Attribution limits

`instance_id` and `component_id` are populated *from `FfiRuntimeContext` during `initialize`*, so
for bootstrap, identity and secret-store plugins they are **permanently unavailable** — those kinds
never receive a context. Telemetry drasi-lib emits *about* them from the builder decorator is fully
labelled; telemetry they emit *themselves* carries `plugin_kind` but no `component_id`. Closing the
gap means adding an `initialize_fn` to those three vtables, a per-kind ABI change, and is deferred.
Per-signal detail in [02 — Metrics §8.7](02-metrics.md#87-attribution-what-plugin-emitted-metrics-cannot-label).

##### Namespace governance and enablement

Tier 3 is open-ended, so it is the one place a third-party plugin could collide with a Drasi metric
name. The SDK issues pre-labelled handles and `FfiMetricsRecorder` enforces the
`drasi.plugin.<plugin_kind>.*` prefix — with indexes, state stores and WALs a permanent exception
because they have no FFI path for a bridge to occupy. Emission is always optional for the plugin
author; `set_log_level` is the precedent for host-controlled filtering, so a disabled metric should
cost a filter check inside the plugin rather than an FFI crossing. Full reasoning and the reopening
condition are in [02 — Metrics §8.8](02-metrics.md#88-namespace-governance) and
[§8.9](02-metrics.md#89-enablement).

### Plugin Developer Experience: Transparent Bridge

Plugin developers use standard Rust `tracing` and `metrics` macros — no custom API needed. The plugin SDK transparently installs bridge implementations that intercept standard calls and forward them to the host:

| Signal | Bridge installed by plugin SDK | Plugin dev uses | How it forwards |
|--------|-------------------------------|----------------|-----------------|
| **Logs** (existing) | `FfiTracingLayer` | `tracing::info!()`, `tracing::error!()` | Intercepts events → `FfiLogEntry` → `LogCallbackFn` |
| **Metrics** (new) | `FfiMetricsRecorder` | `metrics::counter!()`, `metrics::histogram!()` | Intercepts recordings → `FfiMetricEntry` → `MetricsCallbackFn` |
| **Traces** (new) | Extended `FfiTracingLayer` | `tracing::info_span!()` | Intercepts span open/close → `FfiCompletedSpan` → `SpanCallbackFn` |

For trace context injection, the host cannot rely on task-local storage — the plugin runs on its own tokio runtime with its own subscriber, and task-locals do not cross runtime boundaries. Context must be passed explicitly as `FfiTraceContext`. Calls such as bootstrap, identity-provider, and secret-store operations take it as a parameter. Data-plane events use `FfiSourceEvent.trace_context` and `FfiQueryResult.trace_context` because those paths are push- and pull-based with no host call per event. The plugin bridge uses that value as the parent for spans it creates and copies the resulting ids and `trace_flags` into `FfiCompletedSpan`. This is fully transparent to the plugin developer. See [01 — Tracing](01-tracing.md#final-ffi-trace-context-contract) for the boundary table and transfer behavior.

> **Worked examples live with their signal.** Adding metrics to a plugin — declaring tier 2b/3
> handles, obtaining a pre-labelled emitter, and what the author deliberately does *not* write — is
> in [02 — Metrics §8.10](02-metrics.md#810-worked-example-adding-metrics-to-a-source-plugin).
> Writing spans in a plugin — why you never plumb trace context, the one case where nesting
> silently stops (`tokio::spawn`), the four rules that keep plugin spans useful, and how to see your
> spans without a collector — is in
> [01 — Tracing](01-tracing.md#writing-spans-in-a-plugin).

### Enablement, Filtering and Observability Profiles

**DECIDED — a profile is a named preset whose *vocabulary* is defined by drasi-lib and whose
*application* is split between drasi-lib and the embedder. The split is not a preference; it follows
from when each knob has to be set.**

The naive reading of "observability profiles" is that a profile is a filter list: `debug` means more
`drasi.*` series reach the destination, `basic` means fewer. That reading is wrong for Drasi, and getting
it wrong produces a specific silent failure — an operator sets `profile: debug`, gets a subset of
what debug promises, and receives no error. The reason is that **not all telemetry can be filtered
after the fact.** Some of it is never produced unless something was switched on earlier, and "earlier"
is sometimes before drasi-lib exists.

#### Three knob classes, and why the owner differs

| Class | When it must be set | Who owns it | Example |
|---|---|---|---|
| **Export filtering** | After emission, in the recorder/subscriber | **Embedder** | Which `drasi.*` series reach Prometheus; which span targets are enabled |
| **Pipeline collection** | At `DrasiLibBuilder` time, before components start | **drasi-lib** | Gauge observation cadence, expensive gauges, per-event stamping |
| **Backend engine statistics** | In the plugin's own constructor — **before drasi-lib is handed the object** | **Embedder** | RocksDB `Statistics` |

Export filtering is beyond drasi-lib's reach by construction: [Requirement 1](#requirements) forbids
installing a subscriber or recorder, so the `EnvFilter` directives and the
`metrics_util::layers::FilterLayer` that do the filtering both live in code drasi-lib does not own.

The third row is the one that surprises people, so it is worth stating precisely.

#### The governing rule: whoever holds the object when the knob must be set, owns the knob

RocksDB is the worked example the review asked for, and the code answers it unambiguously.

`DrasiLibBuilder::with_index_provider(name, provider: Arc<dyn IndexBackendPlugin>)`
(`lib/src/builder.rs:227`) takes an **already-constructed** provider — `RocksDbIndexProvider::new`
has fixed every RocksDB option before drasi-lib ever sees the `Arc`. drasi-lib cannot reach the knob
from the config side either: a persistent backend is declared as `kind` plus an **opaque
`serde_json::Value`**, and the type's own docs state the principle — *"drasi-lib does not carry
backend-specific serialization for them"* (`lib/src/indexes/config.rs:29-34`).

So the answer to the review's question — *"must RocksDB metrics be enabled when drasi-lib
initializes the component?"* — is **no, they must be enabled before drasi-lib ever sees the
component**, by adding a field to the plugin's *own* config DTO and a parameter to its constructor.
That is the embedder's call, not drasi-lib's.

This generalises because the *shape* generalises: every backend is an opaque config payload
interpreted by the backend and constructed before injection. Redis/Garnet expose a different
surface and get the same answer. Per-backend detail — including why external engines are scraped
directly rather than proxied — is in
[02 — Metrics §6.2](02-metrics.md#62-storage-backend-statistics).

> **Consequence worth stating plainly.** A profile can express *"RocksDB statistics on"* only in a
> deployment where the same actor owns both the profile and the constructor. Drasi Server is such an
> actor — it builds providers from YAML. A library embedder that constructs its own providers is
> not, and for that embedder the profile's storage row is advisory: drasi-lib will report what the
> provider gives it and nothing more.

#### Profiles reuse the phase ordering rather than inventing a second one

The [phase plan](#phase-plan) already ranks telemetry by importance for *shipping* order (P0 → P3).
Profiles rank it by importance for *runtime* exposure. **These are the same ranking**, and the design
deliberately keeps them as one:

| Profile | Metrics | Traces | Collection cost beyond baseline |
|---|---|---|---|
| `off` | none | none | none |
| `basic` *(default when `telemetry` is configured)* | P0 | pipeline spans, sampled | none |
| `debug` | + P1 and P2 | all spans, sampling 1.0, `drasi_core::query=debug` | expensive gauges (index sizes) |
| `persistence` | P0 + `drasi.index.` + engine statistics | + storage spans | **RocksDB `Statistics` on** |

Two properties are load-bearing:

- **`off` → `basic` → `debug` is a ladder**; each is a superset of the one above. That is the "onion"
  the review asked for, made selectable at runtime rather than only at release time. The set is
  deliberately short: an intermediate rung between "the twelve things that matter" and "everything"
  invites bikeshedding about which side each metric falls on, and an operator who has decided they
  need more than `basic` almost always wants all of it.
- **`persistence` is deliberately not a rung.** It crosses the tiers — a little of P0, a lot of
  `drasi.index.` — which is precisely why the review named it separately. Profiles are a small
  closed set, but they are not required to be totally ordered.

If the profile ladder and the phase ladder were allowed to diverge, Drasi would be publishing two
competing answers to "which telemetry matters most". One ranking, two uses.

`debug` is also where [what drasi-core already emits](#what-drasi-core-already-emits) becomes
visible: its ten `#[tracing::instrument]` sites are `level = "debug"`, so `drasi_core::query=debug` is
the directive that opens up interval D from the inside.

#### Resolving a profile: one call, two outputs

The silent failure described at the top is prevented structurally rather than by documentation. A
profile is resolved exactly once, and the call returns **both** halves, so an embedder that applies
only the filters has an obviously unused value:

```rust
pub enum TelemetryProfile { Off, Basic, Debug, Persistence }

pub struct ResolvedProfile {
    /// Directives the embedder installs into its `EnvFilter`.
    pub trace_directives: String,
    /// Metric-name patterns the embedder installs into `FilterLayer` — a **deny** list.
    pub metric_filters: Vec<String>,
    /// What drasi-lib must switch on internally — passed back to the builder.
    pub collection: CollectionFlags,
}

impl TelemetryProfile {
    pub fn resolve(self) -> ResolvedProfile;
}
```

The vocabulary belongs in drasi-lib for three reasons. It is the only component that knows what
signals exist and what each costs, so a profile is a statement about *Drasi's own telemetry surface*
and the server has no independent knowledge of it. Every embedder — not just Drasi Server — gets the
same presets instead of reinventing the filter list. And when a metric is added there is one place to
update, rather than one per embedder.

#### Profiles are defaults, not a straitjacket

A closed set of four presets cannot cover every deployment, so a profile is a **starting point that
targeted overrides adjust** — `basic` plus `drasi.index.`, without stepping all the way up to
`persistence` and paying for everything else in it.

The important constraint is that `metric_filters` is a **deny list**, because that is the only thing
`metrics_util::layers::FilterLayer` implements: *"if a metric key matches any of the configured
patterns, it will be skipped entirely"*, matched as **substrings** via Aho-Corasick — not globs, so
the pattern is `drasi.index.`, never `drasi.index.*`. There is no allow-list layer in the ecosystem,
and building one would be exactly the custom infrastructure this design set out to avoid.

So overrides are expressed as two set operations on the profile's deny list, resolved **before** the
layer is constructed — one mechanism edited, not a second mechanism added:

$$\text{deny} = (\text{profile\_deny} \setminus \text{include}) \cup \text{exclude}$$

| Key | Meaning |
|---|---|
| `exclude` | Add patterns to the deny list — suppress something the profile admits |
| `include` | Remove patterns from the deny list — re-admit something the profile suppresses |

> **`include` is not unbounded.** It can only re-admit signals that are *being produced*. It cannot
> switch on collection the profile left off, because collection is not a filtering decision — see the
> three knob classes above. Asking for engine statistics under `basic` requires changing the profile,
> not adding a filter, and an embedder should reject that combination rather than accept a pattern
> that can never match.

This global-default-plus-override shape is an established one in this codebase rather than a new
invention: `RuntimeConfig` already carries `default_recovery_policy` (documented as *"Global default
for all queries. Per-query `QueryConfig::recovery_policy` overrides this"*) alongside
`global_priority_queue_capacity` and `global_dispatch_buffer_capacity`
(`lib/src/config/runtime.rs:236`). Note that `RuntimeConfig` has **no telemetry field today**, so this
is purely additive.

#### Propagation into subcomponents

Enablement has to travel in three different directions, and only one of them is automatic:

| Destination | Mechanism | Automatic? |
|---|---|---|
| drasi-lib's own pipeline | `CollectionFlags` on the builder | Yes — one process, one config object |
| Statically linked plugins | Same global recorder and subscriber as the host | Yes — the filter applies to everything |
| **cdylib plugins** | Must be **pushed across FFI**; the plugin has its own subscriber and its own global recorder | **No** |
| **Storage backend engines** | Plugin constructor, before injection | **No** — see the rule above |

The cdylib row has a precedent to follow rather than a mechanism to invent. `set_log_level` already
exists on `FfiPluginRegistration` so the host can tell a plugin its effective level, letting the
plugin drop records *before formatting or forwarding* rather than paying an FFI crossing to have them
discarded on the far side. The profile's metric and span filters should be pushed the same way, for
the same reason: a disabled signal should cost a filter check inside the plugin, not a boundary
crossing. See [Plugin Telemetry Across FFI](#plugin-telemetry-across-ffi).

#### Direct or curated?

The third question in this section's original OPEN block — whether a user gets a backend's native
metrics verbatim or a curated subset under `drasi.` — is answered in
[02 — Metrics §6.2](02-metrics.md#62-storage-backend-statistics): **curated by default, verbatim
behind a flag**, backend-neutral names only where the semantics genuinely match. Profiles select
*how much*; §6.2 decides *in what form*.

### Naming and Namespacing Conventions

**DECIDED — the two signals follow deliberately different rules, and each convention is defined in
its own document.**

| Signal | Rule, in one line | Defined in |
|---|---|---|
| **Metrics** | Lowercase dot-separated namespaces under a `drasi.` root, base units and the unit word in the leaf (`_seconds`, `_bytes`), `_total` on monotonic counters, labels never in the name | [02 — Metrics §9](02-metrics.md#9-naming-and-namespacing-conventions) |
| **Spans** | Same lowercase dotted style but **no `drasi.` prefix and no unit suffixes**; identity lives in fields, and namespacing comes from the `tracing` target and OTel instrumentation scope | [01 — Tracing](01-tracing.md#span-naming-and-namespacing) |

Two points are worth stating here rather than in either document, because they are the reason the
conventions diverge at all:

1. **A metric name is a global key; a span name is not.** Two components emitting
   `events_processed_total` collapse into one series, so metrics need a prefix and the bridge
   enforces one. A span already carries its own attributes, parent and trace id, so grouping happens
   at query time and a prefix would only make every name longer.
2. **Service identity is a resource attribute, never a prefix.** `drasi-lib` and `drasi-server` are
   distinguished by OTel `service.name` or a global exporter label — **there is no `drasi.lib.*`
   namespace**, because that would name telemetry after the crate that compiled it rather than what
   it measures, and would stop the same query being chartable across an embedder and the server.

The one span prefix that *is* used is `control.`, for drasi-lib's own management operations — see
[01 — Tracing](01-tracing.md#the-one-prefix-that-is-used-control).

### Enabling Telemetry as a drasi-lib Consumer
The facade principle says drasi-lib emits and the embedder collects, so the expected default is
"nothing is exported until you wire up a backend". That is true for **metrics** and half-true for
**traces**.

#### What you actually get with nothing installed

| Signal | Today, zero embedder setup | Why |
|---|---|---|
| **Logs** | **Working, on stdout, at `info`** | drasi-lib installs `EnvFilter` + `ComponentLogLayer` + `fmt`. `RUST_LOG` is honoured |
| **Traces** | Spans are **created** but only the `fmt` layer observes them, so they surface as log lines with span context — not traces. No trace ids, no waterfall, no export | Nothing converts `tracing` spans into OTel spans without `tracing-opentelemetry` |
| **Metrics** | **Nothing.** `metrics` facade calls are no-ops | No `Recorder` is installed anywhere in drasi-core today |

#### Target setup, after the split

Once `init_component_log_layer()` returns the layer without installing anything, the whole setup is
ordinary `tracing-subscriber` composition:

```rust
use drasi_lib::{DrasiLib, TelemetryProfile};
use tracing_subscriber::{prelude::*, EnvFilter};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let profile = TelemetryProfile::Basic.resolve();

    // 1. Traces + logs: one subscriber, three layers.
    let log_layer = drasi_lib::init_component_log_layer();

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic().with_endpoint("http://otel:4317"))
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    tracing_subscriber::registry()
        .with(EnvFilter::new(&profile.trace_directives))
        .with(log_layer)                                   // component log streams
        .with(tracing_subscriber::fmt::layer())            // stdout
        .with(tracing_opentelemetry::layer().with_tracer(tracer))  // OTLP export
        .init();

    // 2. Metrics: a layered recorder stack (see 02 — Metrics §2).
    metrics_util::layers::Stack::new(
        metrics_exporter_prometheus::PrometheusBuilder::new().build_recorder(),
    )
    .push(metrics_util::layers::FilterLayer::from_patterns(&profile.metric_filters))
    .install()?;

    // 3. Only now construct DrasiLib, and hand it the collection half of the profile.
    let drasi = DrasiLib::builder()
        .with_telemetry(profile.collection)
        .build()
        .await?;

    // 4. Flush on shutdown, or the last export window is lost — see Export, Flush and Crash-Loss.
    // opentelemetry::global::shutdown_tracer_provider();
    Ok(())
}
```

**Note**:

1. **The subscriber is installed before `build()`.** Given the silent `set_global_default`, this is
   the difference between exporting traces and exporting nothing.
2. **`log_layer` is composed in, not replaced.** Dropping it is what empties the component log
   streams — the failure that has no error message.
3. **The profile is resolved once and used three times** — directives, filters, and collection flags.
   This is the structural guard described in
   [Enablement, Filtering and Observability Profiles](#enablement-filtering-and-observability-profiles):
   an embedder who forgets `with_telemetry` has an unused `collection` field staring at them.
4. **Shutdown flush is the embedder's job.** drasi-lib installs no exporter, so it has nothing to
   flush; whoever installed the pipeline owns draining it.

#### Dependencies the embedder adds

drasi-lib itself gains none of these — they are the embedder's choice of backend, which is the whole
point of the facade.

| Crate | For |
|---|---|
| `tracing-subscriber` | Composing the subscriber, `EnvFilter` |
| `tracing-opentelemetry` + `opentelemetry_sdk` + `opentelemetry-otlp` | Turning spans into exported OTel traces |
| `metrics-exporter-prometheus` *or* an OTLP recorder | Collecting metrics |

An embedder that wants **logs only** adds nothing at all and keeps today's behaviour. An embedder
that wants **no telemetry** installs nothing and pays the near-zero disabled-callsite cost described
in [Signal Model](#signal-model) — an atomic load and a branch per span, a no-op virtual call per
pre-registered metric handle.

### Export, Flush and Crash-Loss Semantics

**drasi-lib installs no exporter, so it has no export cadence and nothing to flush.** Batching,
flush and crash-loss are properties of the pipeline the *embedder* installed — which means the
obligation to drain it on shutdown is the embedder's too. An embedder that does nothing silently
loses its last export window on every restart.

Two asymmetries are worth knowing. **Metrics and logs are cheap to lose** — a cumulative counter
self-corrects on the next export ([02 — Metrics §5.7](02-metrics.md#57-export-temporality)), and
plugin logs cross FFI synchronously so nothing is queued. **Spans are not**: a span that has not
ended is never exported, so an abrupt exit loses the longest-running operation first, and
`BatchSpanProcessor` additionally drops silently once its 2048-entry queue fills — the argument for
sampling, in [01 — Tracing](01-tracing.md#per-source-capability).

Three verified defects block a clean shutdown today:

| 🐛 | Evidence |
|---|---|
| **Drasi Server handles `SIGINT` only.** Kubernetes sends `SIGTERM`, so in the deployment target that matters there is **no graceful shutdown path at all** — no destructors, no flush | `tokio::signal::ctrl_c()` (`drasi-server/src/server.rs:825`); zero occurrences of `SignalKind` / `signal::unix` / `terminate` in the server source |
| **`DrasiLib::shutdown()` flushes no telemetry.** It stops components and releases index handles, then logs `"drasi-lib shut down permanently"` — into a queue nothing will drain, so the shutdown confirmation is itself inside the loss window | `lib/src/lib_core.rs:598` |
| **The log worker thread is never joined.** It correctly runs on a dedicated OS thread with its own runtime, but `spawn_log_worker` discards the `JoinHandle`, so nothing waits for the queue to drain | `lib/src/managers/tracing_layer.rs:117` |

**What an embedder must do**, in this order — reversing steps 2 and 3 discards exactly the shutdown
diagnostics you most want:

1. Handle **`SIGTERM` as well as `SIGINT`** — `signal::unix::SignalKind::terminate()`.
2. Call `DrasiLib::shutdown()`, so component-stop logs and final metric values are recorded.
3. Flush telemetry — `SdkTracerProvider::shutdown()` / `force_flush()`, plus the metrics provider
   equivalent on an OTLP push branch. Bound it with the export timeout so a dead collector cannot
   hang termination past the orchestrator's grace period.
4. *Then* tear down the tokio runtime.

Drasi Server is Drasi's own first embedder and satisfies **none** of these; fixing it is Phase 0
work there, not later polish.

### API Design

No changes to the REST API or CLI, and none to the `DrasiLib` builder. The pipeline instrumentation
itself is purely internal — all new spans and metrics are emitted through facade crates and are
transparent to callers.

The **initialization surface changes, and the change is breaking.** That is deliberate: this design
is a rework of how drasi-lib handles telemetry, and preserving an initializer whose entire behaviour
(installing a global subscriber) [Requirement 1](#requirements) forbids would keep the defect
reachable from a supported API.

| Function | Returns | Installs a subscriber? | For |
|---|---|---|---|
| `init_component_log_layer()` | the `ComponentLogLayer` | **No** | Embedders composing their own subscriber — Drasi Server, or anyone adding an OTLP layer |
| `init_default_subscriber()` | `()` | **Yes** — `EnvFilter` + `fmt` + `ComponentLogLayer`, exactly as today | Simple embedders, examples and tests |

`init_default_subscriber()` is implemented in terms of `init_component_log_layer()`, so there is one
code path for creating the registry, the channel and the worker, and a second, thinner one for
installing.

> **DECIDED: `get_or_init_global_registry()` is removed, not deprecated.** Keeping it as an alias
> would leave three functions where two suffice, and would leave the subscriber-installing behaviour
> reachable — which is the thing being fixed. Callers migrate by replacing it with
> `init_default_subscriber()` for identical behaviour, or with `init_component_log_layer()` if they
> want to compose. `DrasiLib::new()` stops calling either one; installing telemetry becomes the
> embedder's job, which is what [Requirement 1](#requirements) means in practice.
>
> **This narrows [Requirement 2](#requirements).** Backward compatibility is preserved where it was
> promised — `log::info!()` still reaches component log streams, `ComponentLogLayer` behaves
> identically, and the REST log API is unchanged. It is *not* preserved for the initializer itself.

#### When the log worker starts

`init_component_log_layer()` creates a bounded channel and a dedicated `drasi-log-worker` thread
running its own current-thread runtime (`lib/src/managers/tracing_layer.rs:117`). If the returned
layer were never installed, that thread would sit forever draining a channel nothing writes to — a
library leaking a thread into a host process that asked for nothing.

Two caveats worth stating rather than discovering:

- The hook fires on **`Dispatch` construction**, which is marginally earlier than a *successful*
  global install. `try_init()` can still fail afterwards with `SetGlobalDefaultError`, leaving a
  worker running for a subscriber that never took effect. That is a strictly better failure than
  today's, and it is bounded — one thread, one process, one occurrence.
- Scoped installs (`with_default`) construct a `Dispatch` too, so a test that only ever sets a
  local subscriber does start the worker. Correct: those tests genuinely want component logs.

If nobody installs anything, the registry exists but stays empty and the REST log API returns
nothing. drasi-lib should say so once at startup rather than let an operator discover it from an
empty log pane — the same reasoning as the recorder-ordering warning in
[02 — Metrics §2.1](02-metrics.md#21-the-decision).

### Phase Plan

**Three delivery phases, each worth shipping even if the next never lands.** Phase *n* ships the
P*n* metric tier, so `Phase 0` and `P0` name the same cut — one ranking, used for both release
order and the [runtime profiles](#enablement-filtering-and-observability-profiles).

| Phase | Question it answers | Ships |
|---|---|---|
| **0** | *Is it alive, is it keeping up, is it losing data?* | The 13 metrics in [02 §4.2](02-metrics.md#42-the-phase-0-metric-set); the 8 host data-plane spans with explicit context handoff across the five tasks ([01](01-tracing.md#canonical-span-names)); the recorder stack; the `get_or_init_global_registry()` split; the `component_type` log fix |
| **1** | *Why is it slow, and what happened inside the plugin?* | All FFI work — the three trace carriers, `PluginSpanSink` + `FfiCompletedSpan`, `set_metrics_recorder`, `trace_id` on `FfiLogEntry`; conventional source-side W3C parent adoption; plugin tier-1/2 spans and metrics; `control.*` spans; the bootstrap span-tree fix; `TelemetryProfile` + sampling |
| **2** | The long tail | Drasi Server HTTP spans (with the attribute allow-list); plugin tier-2b/3 metrics; storage engine statistics; `identity.resolve` / `secret.fetch`; backdated wait spans |

Two properties are load-bearing. **Phase 0 is host-process only** — no FFI, no ABI change, no
plugin cooperation — so it is unaffected by how plugins are linked and ships independently of
everything else. And **profiles land in Phase 1, not Phase 0**: with 13 metrics and 8 spans there
is nothing to filter; they become necessary exactly when Phase 1 multiplies the surface.

Every FFI addition is **append-only under an `sdk_version` gate**, following the precedent set when
`set_log_level` was added to `FfiPluginRegistration` in SDK 0.12.0, so no phase forces a breaking
plugin release.

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
| drasi-lib installs nothing | Unit | Build and run a full `DrasiLib` without calling either initializer; assert `tracing::dispatcher::has_been_set()` is false and no `drasi-log-worker` thread exists. This pins [Requirement 1](#requirements) against regression |
| Worker starts only on install | Unit | Call `init_component_log_layer()` and drop the layer without installing; assert no `drasi-log-worker` thread is spawned. Then compose and `init()`; assert exactly one is, and that a second `Dispatch` does not spawn a second |
| Embedder composition | Integration | Compose `ComponentLogLayer` with `fmt` and a test OTLP layer into one subscriber; assert component log streams *and* exported spans both receive data — the combination neither ordering could achieve before ([API Design](#api-design)) |
| Build-mode identity parity | Integration | Run the same plugin under `builtin-plugins` and again under `dynamic-plugins`; assert its events resolve to identical `source_id` and `plugin_kind` in both. Instrumentation scope is expected to differ and is deliberately not asserted |

> **Note on "near-zero cost".** Neither facade is literally zero-cost. With no backend installed, a
> disabled `tracing` callsite costs an atomic load and a branch, and a pre-registered `metrics`
> handle costs a no-op virtual call. When a backend *is* installed, no telemetry *export* work
> happens on the critical path — provided the embedder installed a batching exporter, which is the
> embedder's responsibility rather than something drasi-lib can guarantee; see
> [Export, Flush and Crash-Loss Semantics](#export-flush-and-crash-loss-semantics). The benchmark
> above is what backs these claims.

## Open Issues

1. ~~**`get_or_init_global_registry()` split**~~ — **RESOLVED.** Split into
   `init_component_log_layer()` (returns the layer, installs nothing) and `init_default_subscriber()`
   (composes and installs, as today). `get_or_init_global_registry()` is **removed rather than kept
   as an alias** — a deliberate breaking change, accepted because this design is a rework of
   telemetry initialization and an alias would leave the subscriber-installing behaviour reachable
   from a supported API. The log worker thread starts only when a subscriber is actually installed,
   via `Layer::on_register_dispatch`. See [API Design](#api-design).

2. ~~**Management-plane tracing scope**~~ — **RESOLVED.** Split by layer rather than by picking a side: drasi-lib's own mutating control-plane ops (`lib_core_ops`) are traced in Phase 1, because every embedder has a control plane whether or not a server sits in front of it; Drasi Server's HTTP API spans stay a Phase 2 item, so SRV's non-goal stands as written. Control-plane spans carry a `control.` prefix so they can be filtered out wholesale. See [01 — Tracing](01-tracing.md#control-plane-rooting--drasi-libs-own-api).

3. ~~**drasi-core tracing**~~ — **RESOLVED.** The "black box" framing was wrong, but so was the
   proposed correction that drasi-core "already emits standard `tracing`". It emits **spans only, never
   events**, and the spans are off by default. See
   [What drasi-core already emits](#what-drasi-core-already-emits) for the verified position.

## References

- [`tracing` crate](https://crates.io/crates/tracing) — Structured diagnostics facade for Rust
- [`metrics` crate](https://crates.io/crates/metrics) — Metrics facade for Rust
- [OpenTelemetry semantic conventions — naming](https://opentelemetry.io/docs/specs/semconv/general/naming/) — the basis for the dotted-namespace scheme
- [Prometheus — metric and label naming](https://prometheus.io/docs/practices/naming/) — the basis for base units and the `_total` suffix
