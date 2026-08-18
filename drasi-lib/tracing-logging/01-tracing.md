# Tracing for drasi-lib

* Project Drasi - April 10, 2026 - Ruokun Niu (@ruokun-niu)

> Part of the drasi-lib observability design set. Read
> [00 — Overview and Shared Foundations](00-observability-overview.md) first: it defines the facade
> principle, the pipeline model and task/interval labels used below, the plugin FFI transport, the
> naming conventions, and the enablement/profile model. Metrics are covered in
> [02 — Metrics](02-metrics.md).

## Overview

drasi-lib has no structured span hierarchy across the Source → Query → Reaction pipeline today.
Log lines are per-component and isolated, so you can't tell how long anything took, whether an
error related to a particular event, or how long an event waited in a queue:

```
[INFO] source postgres-src: Starting source
[INFO] source postgres-src: Received event
[INFO] query q1: Processing source change
[ERROR] query q1: Error processing source change: timeout
[INFO] reaction webhook: Dispatching results
```

This document defines structured `tracing` spans that wrap those existing log events to add
duration, causality, and nesting:

```
TRACE [0.8ms] source.dispatch { source_id=postgres-src, op=insert, label=Order, element_id=Order:42 }
  └── TRACE [0.1ms] query.receive { source_id=postgres-src, query_id=q1 }
        └── TRACE [45.3ms] query.process { query_id=q1, source_id=postgres-src }
              ├── [INFO] Processing source change      ← existing log event, now inside a timed span
              └── TRACE [3.1ms] query.dispatch { query_id=q1, added=1 }
                    └── TRACE [1.2ms] reaction.receive { reaction_id=webhook, query_id=q1 }
```

The existing log events continue to work — they just now appear inside spans that provide timing context and cross-component causality.

## Design

### Span Hierarchy and End-to-End Trace Propagation

Spans are placed at the boundaries of each pipeline stage. They form a **single connected trace tree** for each event flowing through the system, even though the pipeline uses separate async tasks connected by channels (PriorityQueue, ChangeDispatcher).

**Today**: drasi-lib's pipeline runs across 5 independent tokio tasks connected by async channels (see the [pipeline model](00-observability-overview.md#pipeline-model-and-instrumentation-points)). By default, `tracing` spans don't propagate across channel boundaries — each task would create an unrelated root span, resulting in several disconnected traces per event instead of one.

**Proposed Solution**: When a span is created in one task, we capture a lightweight `tracing::Span` handle and carry it through the channel alongside the event data. The downstream task then creates its span with the carried handle as its **explicit parent** (`info_span!(parent: &carried_span, ...)`), producing true parent-child nesting. We use explicit parent-child throughout — **not** `follows_from`: the carried handle keeps the parent span alive until the child is created, so the child records the parent's span ID and the trace renders as the nested tree shown below (`follows_from` would produce loose causal links rather than the nesting the diagrams depict). This produces a single trace tree:

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

#### Multi-Branch Trace Trees (Fan-Out)

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

All branches share the same `trace_id` with `source.dispatch` as the root. In Jaeger this renders as a single expandable trace tree. When no `tracing::Subscriber` is installed, a disabled `info_span!()` costs an atomic load and a branch and allocates nothing — near-zero, not literally zero. When a subscriber is installed, the cost is proportional to the pipeline topology that the user explicitly configured. Each span is ~200 bytes in a typical subscriber (span name + fields + timestamps).

#### Canonical Span Names

These five span names are authoritative and used consistently throughout this document — the pipeline instrumentation introduces no other span names:

| Span | Task | Key fields |
|------|------|-----------|
| `source.dispatch` | Source plugin (T1) | `source_id`, `op`, `label`, `element_id` |
| `query.receive` | Query forwarder (T2) | `source_id`, `query_id` |
| `query.process` | Event processor (T3) | `query_id`, `source_id` |
| `query.dispatch` | Event processor (T3) | `query_id`, `added`, `updated`, `deleted` |
| `reaction.receive` | Reaction forwarder (T4) | `reaction_id`, `query_id` |

> **OPEN.** These names are subject to the naming convention still being decided in
> [00 — Overview](00-observability-overview.md#naming-and-namespacing-conventions).

### Trace Rooting and Lifetimes

> **OPEN — to be resolved in this revision.** The direction agreed in review:
>
> - **Data plane**: each inbound source change **starts a new trace at the source**.
> - If the source or an external caller supplies a trace parent (e.g. an HTTP `traceparent`),
>   **honour it** rather than creating a new root. This depends on the inbound trace-context
>   mechanism in [00 — Overview](00-observability-overview.md#inbound-trace-context).
> - **Bootstrap and management operations** root on the host or management side — not on the query.
> - Traces stay **independent**. Do *not* make a source or query container the universal parent of
>   every span it produces; this was tried in drasi-platform/Kubernetes and overwhelmed the tracing
>   infrastructure. Use span **attributes** for source/query identity so grouping happens at query
>   time in the trace backend.
>
> **Retrospective / historical spans — to investigate.** When no pre-existing parent is supplied,
> we still know timestamps for work that already happened (source change time → time Drasi received
> it → time it was dispatched). Back-filling spans over those intervals would show pre-dispatch
> latency instead of the trace starting cold. This is not straightforward with the standard
> `tracing` libraries, which deliberately do not expose start/end timestamp control — it likely
> needs the same OTel-SDK `SpanData` construction path used for the FFI bridge. The drasi-platform
> (Kubernetes) test framework contains a reaction that creates historical spans for a whole query
> lifecycle and is worth cribbing from. Decide whether this is an option/config flag or dropped
> from phase 1.

### Trace Flows: Where Context Must Cross

> **OPEN — to be resolved in this revision.** Logs flow one way (plugin → host → out), but **traces
> flow both ways**. This section must enumerate every host→plugin and plugin→host call that has to
> carry trace context — query invocation, bootstrap, `dispatch_change()`, reaction invocation,
> identity-provider calls, secret-store calls — and document each with a sequence/ladder diagram:
>
> 1. Source-originated change (e.g. Postgres CDC) — the activity *starts inside the plugin*, moves
>    to the host query, then into a reaction plugin
> 2. Bootstrap
> 3. Query → reaction
> 4. Management-plane operation (scope still to be reconciled — see
>    [00 — Overview, Open Issues](00-observability-overview.md#open-issues))

### Trace Context Propagation Across Channels

To link spans across task boundaries, we carry a `tracing::Span` handle through the channel alongside the event data. The downstream task creates its span with that handle as its **explicit parent**, establishing parent-child nesting (see [Span Hierarchy](#span-hierarchy-and-end-to-end-trace-propagation) above).

#### Where to put `parent_span`: wrapper vs event

Trace context needs to cross two types of channel boundaries:
1. **ChangeDispatcher** (T1→T2 source events, T3→T4 query results) — dispatches `Arc<T>` directly
2. **PriorityQueue** (T2→T3, T4→T5) — wraps `Arc<T>` in `PriorityQueueEvent<T>`


**Design**: Use different strategies for each boundary:

| Boundary | Carrier | Strategy |
|----------|---------|----------|
| **PriorityQueue** (T2→T3, T4→T5) | `PriorityQueueEvent<T>` wrapper | Add `parent_span: Option<Span>` to the wrapper |
| **ChangeDispatcher — source events** (T1→T2) | `SourceEventWrapper` | Add `parent_span: Option<Span>` to the wrapper |
| **ChangeDispatcher — query results** (T3→T4) | `QueryResult` | Add `parent_span: Option<Span>` to the dispatched type |

**PriorityQueue wrapper** (carries span alongside the event, not inside it):

```rust
// channels/priority_queue.rs
struct PriorityQueueEvent<T> {
    event: Arc<T>,
    parent_span: Option<tracing::Span>,  // NEW: on the wrapper, not inside Arc<T>
}
```

This preserves the zero-copy property: `Arc<T>` is cloned (just a refcount bump) when needed, but `T` itself is never cloned. The `parent_span` lives on the wrapper and is consumed when the event is dequeued — it does not add to the shared `Arc<T>` allocation.

**ChangeDispatcher path** — the `ChangeDispatcher` is used for two boundaries, so both carriers get a `parent_span` field:

Source events (T1→T2) — `SourceEventWrapper` gains the field so the `source.dispatch` span propagates to `query.receive` (field names illustrative):

```rust
// SourceEventWrapper (channels/events.rs)
pub struct SourceEventWrapper {
    pub source_id: String,
    pub change: Arc<SourceChange>,
    pub parent_span: Option<tracing::Span>,  // NEW: carries source.dispatch span; None when no tracing backend
}
```

Query results (T3→T4) — `QueryResult` gains the field so the `query.dispatch` span propagates to `reaction.receive`:

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

At each task boundary, the downstream task creates its span with the carried `parent_span` as its **explicit parent** (`info_span!(parent: carried_span, ...)`). This produces the connected trace tree shown above.

### Advantages of This Approach

- **Near-zero cost by default**: with no subscriber installed, a disabled `info_span!()` costs an atomic load plus a branch and allocates nothing
- **No breaking changes**: existing `ComponentLogLayer` and `log` crate usage are unchanged
- **Standard ecosystem**: uses `tracing`, enabling direct integration with Jaeger, Datadog, OTLP, etc.
- **Composable**: `ComponentLogLayer` + `tracing_opentelemetry` + `fmt` all work together
- **Embeddable into existing traces**: because drasi-lib uses the `tracing` facade without installing its own subscriber, an application that already has a `tracing` subscriber (e.g. with `tracing-opentelemetry`) can embed drasi-lib and its pipeline spans will automatically nest under the application's active span — the `source.dispatch` → `query.process` → `reaction.receive` tree becomes a subtree of the application's own trace, with no special configuration

### Plugin Spans

Spans created inside cdylib plugins cannot use the `tracing::Span`-handle mechanism above, because
span handles are tied to the subscriber that created them and plugins run their own runtime and
subscriber. They reach the host through the centralized callback bridge described in
[00 — Overview](00-observability-overview.md#plugin-telemetry-across-ffi), which covers the
`FfiCompletedSpan` struct, host-side reconstruction, inbound trace-context injection, and the
open questions on span formation and transfer timing.

### Alternatives Considered

#### 1. Add `#[instrument]` to All Public Functions

Automatically create spans for every public function using the `#[instrument]` attribute.

**Rejected because**: This creates too many fine-grained spans that add noise and overhead. The pipeline boundary approach (source ingest → query process → reaction dispatch) gives the right level of granularity for debugging and monitoring.

#### 2. Independent Spans Per Task (No Cross-Channel Trace Linking)

Create spans only within each task's scope and don't carry trace context through the PriorityQueue or ChangeDispatcher channels. Each task would create a root span, producing 3 disconnected traces per event:

- **Trace A**: `source.dispatch` (source forwarder task)
- **Trace B**: `query.process` → `query.dispatch` (event processor task)
- **Trace C**: `reaction.receive` (reaction forwarder task)

**Rejected because**: The primary value of distributed tracing is following a single event end-to-end. Disconnected traces per event make it impossible to correlate what happened to a specific source change across the pipeline — you'd have to manually match them by timestamp and field values. Carrying a span handle through the channel is a small amount of additional data (one `Arc` clone per event) and standard practice in async Rust applications that use channel-based architectures. Holding the parent handle across the channel lets each downstream span declare it as an explicit parent, producing the nested tree above.

## Supportability

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Span creation | Unit | Use `tracing_subscriber::fmt::TestWriter` or `tracing-test` crate to assert that expected spans are created with correct fields when processing a mock source change |
| Trace connectivity | Unit | Push an event through a mock pipeline and assert all spans share one `trace_id` with the expected parent-child nesting across task boundaries |
| End-to-end with OTLP | Manual / Integration | Example app with `tracing-opentelemetry` + Jaeger; verify spans appear in Jaeger UI with correct nesting and fields |
| Inbound trace parent | Integration | Supply an external `traceparent` at a source and assert Drasi's spans attach to it rather than starting a new root |
| Plugin span nesting | Integration | Load a cdylib plugin that emits its own span; assert it is exported under the pipeline `trace_id` with the host span as parent |

Security, compatibility impact, and the shared verification checks are covered in
[00 — Overview](00-observability-overview.md).

## Open Issues

1. **Bootstrap span granularity**: During bootstrap, `process_source_change` is called once per initial data element (potentially thousands). Should each bootstrap event get its own `query.bootstrap` span, or should there be a single parent span for the entire bootstrap phase with lightweight events per element? Resolve alongside the bootstrap trace root in [Trace Rooting and Lifetimes](#trace-rooting-and-lifetimes).

2. **Plugin trace context granularity**: Should trace context be passed per-event (each `dispatch_change()` / reaction invocation carries the current `trace_id` + `parent_span_id`) or once at plugin initialization? Per-event links each plugin span to its specific pipeline event. Per-init is simpler but all plugin work shares one parent span. Review direction favours per-event; confirm when resolving the FFI mechanics in [00 — Overview](00-observability-overview.md#plugin-telemetry-across-ffi).

3. **Trace sampling**: For high-throughput deployments, head-based sampling may be needed. Decide whether phase 1 ships a sampling rate and how it interacts with host-side filtering of plugin spans. Note this is a *trace* control — metric volume is managed by aggregation instead, see [02 — Metrics](02-metrics.md).

## References

- [`tracing` crate](https://crates.io/crates/tracing) — Structured diagnostics facade for Rust
- [drasi-platform query-host](https://github.com/drasi-project/drasi-platform/tree/main/query-container/query-host) — Reference implementation of OpenTelemetry tracing in Drasi
- [drasi-platform query-host `init_tracer()`](https://github.com/drasi-project/drasi-platform/blob/main/query-container/query-host/src/main.rs) — Existing OTLP setup pattern used in Drasi for Kubernetes
