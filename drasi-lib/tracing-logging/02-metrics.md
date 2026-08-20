# Metrics for drasi-lib

* Project Drasi - Ruokun Niu (@ruokun-niu)
* Last edited on August 18th, 2026

> Part of the drasi-lib observability design set. Read
> [00 — Overview and Shared Foundations](00-observability-overview.md) first: it defines the facade
> principle, the pipeline model and interval labels used below, the plugin FFI transport, the
> naming conventions, and the enablement/profile model. Tracing is covered in
> [01 — Tracing](01-tracing.md).

## Overview

drasi-lib already measures a fair amount — three `Atomic*`-backed metric structs under
`lib/src/metrics/`, queue depth and backpressure counters in the priority queue, and per-event
nanosecond timestamps in `ProfilingMetadata` — but **none of it can leave the process**. The metric
structs are readable only through the synchronous inspection API, the queue counters are read by
nothing at all, and the profiling timestamps surface only as `info!()` lines from the opt-in
Profiler Reaction. There is no `metrics` facade, no OpenTelemetry, and no exporter anywhere in
`drasi-core`.

This document adds the dependency and settles how measurements are collected (§1–§2), inventories
what drasi-lib already measures (§3), then defines the metrics it should emit (§4) and how their
volume is controlled (§5). **§3 and §4 are adjacent deliberately**, so the existing measurements and
the proposed set can be read side by side — most of Phase 0 is a promotion of something §3 already
lists.

## Design

### 1. New Dependency: `metrics` Crate

Add `metrics = "0.24"` to `lib/Cargo.toml`. This is the facade crate only — no exporter.

```toml
# lib/Cargo.toml
[dependencies]
metrics = "0.24"
# tracing, tracing-subscriber, tracing-log already present
```

**This is the only dependency drasi-lib takes.** The `metrics-util` types used in §2.2
(`FanoutBuilder`, `Stack`, `Registry`) and any exporter crate are the **embedder's** dependencies,
not drasi-lib's — they are needed to *collect* metrics, and §2.1 puts collection outside the
library. Drasi Server therefore takes `metrics-util` and an exporter; a library consumer that wants
no metrics at all takes neither.

### 2. Collection Architecture

#### 2.1 The Decision

**drasi-lib emits into the process-global recorder and does nothing else.** The recorder is a
static owned by the `metrics` crate, not by Drasi — `set_global_recorder()` fills that slot exactly
once per process, and a second attempt fails with `SetRecorderError`. That set-once rule is why the
library must never install one: an embedding application that already uses `metrics` for its own
instrumentation would otherwise be silently overridden. This is the facade principle from
[00 — Overview](00-observability-overview.md), applied to metrics exactly as it is to tracing.

So the contract drasi-lib imposes on an embedder is one line long:

| Responsibility | Owner |
|---|---|
| Emit via `counter!` / `gauge!` / `histogram!` | drasi-lib, plugins |
| Install a recorder — *if any metrics are wanted at all* | The embedding application |

**Most embedders need to do nothing.** If the host application already installs a recorder, Drasi's
metrics flow into it automatically. If it installs none, every call is a near-no-op and there is no
collection. An embedder who wants Drasi's metrics but has no recorder yet installs an off-the-shelf
one — they never write a recorder themselves:

```rust
fn main() -> anyhow::Result<()> {
    // Installed by the embedding program, not by drasi-lib. Must come first — see below.
    metrics_exporter_prometheus::PrometheusBuilder::new()
        .with_http_listener(([0, 0, 0, 0], 9000))
        .install()?;

    let drasi = DrasiLib::builder().build().await?;
    // ...
}
```

> **Install the recorder before building Drasi.** `counter!()` returns a handle from whatever
> recorder is installed *at the moment it is called*, and with none installed that is a **noop
> handle**. Because §4 requires handles to be registered once at component construction and cached
> on the component struct, any component created before the recorder is installed caches a noop
> handle and keeps it — permanently, even after a recorder is installed later. The failure is
> silent: no error, no warning, just components that never report. Ordering is therefore part of
> the contract, and drasi-lib should log a warning at startup if no recorder is installed.

#### 2.2 Reference Configuration — Fanout to a Registry and an Exporter

Everything below is a **recommended configuration for embedders that want to read metrics back
in-process**, not a requirement of drasi-lib. Drasi Server adopts it because it serves metrics from
its management API and renders them in its UI; the specifics of those endpoints belong to the
[Drasi Server observability design](../../drasi-server/tracing-logging/00-observability-integration.md).
An embedder that only wants to export to Prometheus or OTLP should use the two-line setup above and
ignore this section.

The configuration is a single stack that fans every measurement out to two destinations:

| # | Step | Who does it |
|---|---|---|
| 1 | **Fan out** every measurement to both branches via `metrics_util::layers::Fanout` | The stack |
| 2 | **Hold current values** in a Drasi-owned `metrics_util::registry::Registry` | Branch A — the registry |
| 3 | **Export** by Prometheus scrape or OTLP push | Branch B — an off-the-shelf exporter |
| 4 | **Read in-process** for an API, UI counters, or the telemetry Source (§7) | Consumers reading branch A |
| 5 | **Evict** series for components that have been removed | Branch A, via `delete_*` / `retain_*` |

```rust
use metrics_util::layers::{FanoutBuilder, Stack};

// Branch A — a Drasi-owned store, readable in-process.
let drasi_recorder = DrasiRecorder::new();      // wraps metrics_util::registry::Registry
let registry = drasi_recorder.handle();         // MUST be captured before install() — see §2.5
// Branch B — an off-the-shelf exporter, chosen by configuration.
let exporter = match cfg.backend {
    Backend::Prometheus => PrometheusBuilder::new().build_recorder(),
    Backend::Otlp       => otlp_recorder(&cfg)?,
};

let fanout = FanoutBuilder::default()
    .add_recorder(drasi_recorder)
    .add_recorder(exporter)
    .build();

Stack::new(fanout)
    .push(FilterLayer::from_patterns(cfg.disabled_metric_patterns))  // later phase — see §2.3
    .install()?;                                                     // fills the global slot
```

> **Plugins do not share this recorder.** A cdylib plugin links its own copy of the `metrics`
> crate, so it has its own global slot — `install()` in the host has no effect inside a dynamically
> loaded plugin, exactly as a host `tracing` subscriber does not reach plugin spans. Plugin metrics
> reach the host through the FFI bridge described in
> [00 — Overview](00-observability-overview.md), not through this stack.

#### 2.3 The Stack

```
   drasi-lib  ·  plugins  ·  Drasi Server
              │
              │  metrics facade — counter! / gauge! / histogram!
              ▼
   ┌──────────────────────────────────┐
   │  Stack (installed global recorder)│
   │                                   │
   │   FilterLayer                     │  enablement + namespace filtering
   │        │                          │
   │      Fanout                       │
   │        ├──────────────┐           │
   └────────┼──────────────┼───────────┘
            ▼              ▼
     Drasi Registry    Exporter
     (metrics_util)    (Prometheus scrape │ OTLP push)
            │
            ├─► /metrics on the existing management API port
            ├─► UI counters
            └─► telemetry Source (§7)
```

| Piece | Crate item | Purpose |
|---|---|---|
| Stack | `metrics_util::layers::Stack` | Compose layers, `install()` as the global recorder |
| Filter | `layers::FilterLayer` (feature `layer-filter`) | Discard metrics by name pattern — the mechanism behind observability profiles and per-component enablement |
| Prefix | `layers::PrefixLayer` | Optional namespacing |
| Fanout | `layers::FanoutBuilder` | One emission, several destinations |
| Router | `layers::RouterBuilder` (feature `layer-router`) | Later: route by prefix to *different* destinations (e.g. verbose engine stats to a local sink only) |
| Registry | `metrics_util::registry::Registry` | The Drasi-owned store |

#### 2.4 What the Registry Buys That an Exporter Alone Does Not

| Capability | Why the exporter alone is insufficient |
|---|---|
| **Structured in-process reads** | `visit_counters()` / `get_counter_handles()` return typed values and a point-in-time snapshot. `PrometheusHandle::render()` returns a rendered `String` — parsing our own text output to power an API would be absurd |
| **No second port** | `/metrics` can be served from the existing management API rather than opening a dedicated port, which answers the "don't open another port" objection directly |
| **Telemetry as a Source (§7)** | A local read path with no network loopback back into Drasi |
| **Stale-series removal** | `delete_counter()` / `retain_*()`, plus `Recency` and `GenerationalAtomicStorage`, let series for a deleted query or reaction be evicted. Without this, deleting a query leaves `drasi.component.up` reporting `1` forever |

That last row is not a nicety. Drasi supports adding and removing components at runtime, so without
explicit eviction every removed component leaves permanently frozen series behind.

#### 2.5 Write Path and Read Path

The registry does not *read from* the stack — it is a **leaf of** the stack, written to on every
measurement. Consumers then read the registry. Getting this direction right matters, because it
determines where the handle comes from and what the read semantics are.

**Write.** `Fanout` calls `register_*` on each inner recorder and returns a single composite handle
that wraps all of them, so one `.increment()` performs **one atomic add per branch**. Two branches
means two atomic operations per measurement rather than one — still nanoseconds, but it is not free
and it scales with the number of destinations.

**Read.** The `Arc<Registry>` handle must be cloned out **before** `Stack::install()`, because
installing moves the recorder into the global slot and it can never be retrieved afterwards. This is
why §2.2 captures `drasi_recorder.handle()` on the line before `install()`, and it mirrors
`PrometheusBuilder::build()` returning a `(recorder, handle)` pair. Consumers then use
`get_counter_handles()` for a point-in-time snapshot or `visit_counters()` to walk the registry
without locking it whole.

> **Histograms are not multi-reader, and this constrains the design.** Counters and gauges are
> `AtomicU64` loads — non-destructive and repeatable, so any number of consumers can read them
> independently. A histogram under `AtomicStorage` is an `AtomicBucket<f64>` holding raw samples,
> and the only ways to read it are `data()` / `data_with()` (non-destructive but O(every sample
> ever recorded)) or `clear_with()` (drains). So if nothing drains, the bucket grows without bound;
> and if the `/metrics` handler drains, the UI reading afterwards sees nothing. **In-process
> consumers steal samples from each other.** Note this is a hazard *within* a branch only — the
> exporter has its own independent storage, so the two branches never interfere.

Two ways to resolve it:

| Option | Consequence |
|---|---|
| **`metrics_util::storage::Summary`** — a quantile sketch with relative-error guarantees | Bounded memory, non-destructive reads, safe for any number of consumers. Raw samples are lost, so exact max and arbitrary re-aggregation are unavailable |
| **One designated drainer** that `clear_with()`s on the export interval and publishes an immutable snapshot everyone else reads | Keeps raw samples and exact quantiles; adds a snapshot-publishing step and makes every consumer's view as stale as the last drain. This is what `metrics-exporter-prometheus` does internally |

> **DECIDED: the designated drainer, running on the §5.2 export interval.** It keeps the exported
> and in-process views identical by construction, which matters because a `/metrics` endpoint and a
> dashboard disagreeing about p99 would be worse than either being slightly stale. Raw samples are
> also worth keeping because bucket boundaries are permanently the embedder's choice (Open Issue 2)
> — a sketch
> would foreclose re-aggregating them later.
>
> **No consumer other than the drainer may call `clear_with()`.** That is the whole of the
> contract: reads by the `/metrics` handler, the inspection API and the UI all go to the published
> snapshot, never to the `AtomicBucket`. The same one-reader/idempotent-export rule governs
> `drasi.queue.depth_max` (§4.2), and for the same reason — these are two instances of one pattern,
> not two independent decisions.


### 3. Existing Metrics Inventory
#### 3.1 `lib/src/metrics/` — formal counters, exposed via the inspection API

Three `Atomic*`-backed structs, each with a `.snapshot()` returning a plain `Clone` struct. These
are the only existing metrics reachable by an embedder. They are surfaced on `DrasiLib` in
`lib/src/inspection.rs`:

| Accessor | Scope | Returns |
|---|---|---|
| `get_query_output_metrics(query_id)` | per query | `QueryOutputMetricsSnapshot` |
| `get_reaction_metrics(reaction_id)` | per reaction | `HashMap<query_id, ReactionMetricsSnapshot>` |
| `get_lifecycle_metrics()` | global | `LifecycleMetricsSnapshot` |

**`QueryOutputMetrics`** — per query, `lib/src/metrics/query_metrics.rs`

| Field | Type today | Semantics | Unit |
|---|---|---|---|
| `outbox_size` | `AtomicUsize` | Entries in the outbox ring buffer | gauge, count |
| `outbox_earliest_seq` | `AtomicU64` | Sequence of the oldest outbox entry (0 if empty) | gauge, sequence |
| `outbox_latest_seq` | `AtomicU64` | Sequence of the newest outbox entry (= `as_of_sequence`) | gauge, sequence |
| `result_seq_advances` | `AtomicU64` | Times a new result advanced the sequence counter | counter |
| `live_results_count` | `AtomicUsize` | Live (non-deleted) results tracked by the query | gauge, count |
| `outer_transaction_duration_ns_last` | `AtomicU64` | Most recent outer-transaction duration | ns |
| `outer_transaction_duration_ns_max` | `AtomicU64` | Max outer-transaction duration observed | ns |
| `snapshot_fetch_count` | `AtomicU64` | Times a reaction fetched a snapshot from this query | counter |

**`ReactionMetrics`** — per (reaction, query) pair, `lib/src/metrics/reaction_metrics.rs`

| Field | Type today | Semantics | Unit |
|---|---|---|---|
| `checkpoint_sequence` | `AtomicU64` | Latest checkpoint sequence persisted for the pair | gauge, sequence |
| `checkpoint_lag` | `AtomicU64` | Query's latest outbox sequence − checkpoint | gauge, count |
| `dedup_skip_count` | `AtomicU64` | Events skipped as already processed | counter |
| `gap_detection_count` | `AtomicU64` | Sequence gaps detected in the broadcast stream | counter |
| `recovery_strict_count` | `AtomicU64` | Strict recovery policy triggered | counter |
| `recovery_auto_reset_count` | `AtomicU64` | AutoReset recovery policy triggered | counter |
| `recovery_auto_skip_gap_count` | `AtomicU64` | AutoSkipGap recovery policy triggered | counter |
| `fetch_snapshot_count` | `AtomicU64` | Full snapshots fetched during bootstrap/recovery | counter |
| `fetch_outbox_count` | `AtomicU64` | Outbox catch-up fetches during bootstrap/recovery | counter |

**`LifecycleMetrics`** — global, `lib/src/metrics/lifecycle_metrics.rs`

| Field | Type today | Semantics | Unit |
|---|---|---|---|
| `startup_rejection_durable_no_store` | `AtomicU64` | Durable reaction with no state store | counter |
| `startup_rejection_durable_on_volatile` | `AtomicU64` | Durable reaction on a volatile store | counter |
| `startup_rejection_snapshot_skip_gap` | `AtomicU64` | `needs_snapshot_on_fresh_start` + AutoSkipGap | counter |
| `startup_rejection_no_snapshot_auto_reset` | `AtomicU64` | No snapshot + AutoReset | counter |
| `auto_reset_completions` | `AtomicU64` | Successful AutoReset full re-bootstraps | counter |
| `hash_mismatch_count` | `AtomicU64` | Config hash mismatches detected at startup | counter |

#### 3.2 `PriorityQueueMetrics` — collected but never surfaced

`lib/src/channels/priority_queue.rs` maintains six lock-free values with a `pub async fn metrics()`
accessor:

| Field | Type today | Semantics |
|---|---|---|
| `total_enqueued` | `AtomicU64` | Cumulative enqueues |
| `total_dequeued` | `AtomicU64` | Cumulative dequeues |
| `current_depth` | `AtomicUsize` | Instantaneous queue depth |
| `max_depth_seen` | `AtomicUsize` | High-water mark |
| `drops_due_to_capacity` | `AtomicU64` | Events dropped when at capacity |
| `blocked_enqueue_count` | `AtomicU64` | Times `enqueue_wait()` blocked on backpressure |

#### 3.3 `ProfilingMetadata` — per-event timestamps, already unconditional

`lib/src/profiling/mod.rs` defines eleven `Option<u64>` nanosecond stamps carried on each event:
`source_ns`, `reactivator_start_ns`, `reactivator_end_ns` (intended to be supplied by the external
source); `source_receive_ns`, `source_send_ns`, `query_receive_ns`, `query_core_call_ns`,
`query_core_return_ns`, `query_send_ns`, `reaction_receive_ns`, `reaction_complete_ns`.

> **The first three are never populated.** A repository-wide search finds **zero code that writes
> `source_ns`, `reactivator_start_ns` or `reactivator_end_ns`** — in `lib/` or in any plugin. They
> are always `None`. The FFI payload reinforces this: `SourceEventPayload`
> (`components/plugin-sdk/src/ffi/payload.rs:122`) deliberately omits profiling entirely, commented
> as "`None` at the point a source emits an event and is populated later by the framework" — so a
> plugin has no way to supply them even if it wanted to. **Pre-Drasi latency is therefore not
> measured by anything today**, and any metric or span that claims to show it would be reporting a
> value that does not exist.

Seven intervals are derived from them: `elapsed_source_to_query`, `elapsed_query_processing`,
`elapsed_query_to_reaction`, `elapsed_reaction_processing`, `elapsed_total`,
`elapsed_source_internal`, `elapsed_query_internal`, plus `elapsed_summary_ms()`.

#### 3.4 Profiler Reaction — aggregation that exists but only logs

`components/reactions/profiler/src/profiler.rs` maintains a sliding window over the five derived
intervals and computes `MetricStats { count, mean, variance, std_dev, min, max, p50, p95, p99 }`. Output is a periodic `info!()` line per interval, on a
`report_interval_secs` timer. There is no file, endpoint, or structured export. It is an opt-in
user-configured reaction, not built-in behaviour.

#### 3.5 What does not exist today

Confirmed absent by repository-wide search:

- **No `metrics` facade, no OpenTelemetry, no Prometheus** anywhere in `drasi-core`.
- **No exporter or endpoint.** `drasi-server` exposes no metrics route.
- **No memory or footprint instrumentation.** No `footprint`, `sysinfo`, `jemalloc`, or heap accounting.
- **No storage/index backend statistics.** The RocksDB index components never enable RocksDB's `Statistics` API — no cache hit rates, compaction stats, level distribution, or file sizes.
- **No per-plugin metrics.** All 22 source plugins and 18 of the 19 reaction plugins (all but Profiler) have zero instrumentation — no connection state, retry counts, batch sizes, error counts, or lag. The atomics that do appear under `components/` are either test scaffolding or positional state (e.g. Postgres `read_lsn` / `flush_fence_lsn`), not metrics — though the difference between those two LSNs is a replication-lag metric waiting to be promoted.

#### 3.6 Disposition of existing values

| Existing | Disposition |
|---|---|
| `lib/src/metrics/` three structs | **Keep and additionally emit.** They serve a different purpose (synchronous in-process inspection for the API/UI); the `metrics` facade adds export. Emit alongside rather than replacing, so the inspection API is unchanged. |
| `PriorityQueueMetrics` | **Promote.** `current_depth` → `drasi.queue.depth` gauge, `drops_due_to_capacity` → `drasi.queue.drops_total`, `blocked_enqueue_count` → `drasi.queue.blocked_enqueues_total`. Closes the biggest existing gap at almost no cost. |
| `ProfilingMetadata` stamps | **Reuse as the source of the latency histograms** in §4.2. Already written unconditionally, so no gating change is needed — but the clock source must be fixed first (§5.6). |
| `ProfilingConfig` / `should_profile()` | **Delete (§5.6).** Zero callers; the configuration implies an opt-in and a sampling rate that do not exist. Removing it makes the current unconditional behaviour honest, and avoids reintroducing a gate the exported histograms must not sit behind (§5.5). |
| Profiler Reaction statistics | **Keep unchanged.** Aggregation moves to the recorder for exported metrics; the Profiler stays as a standalone opt-in tool. |
| `opentelemetry = "0.20"` in `core/Cargo.toml` | **Remove.** Dead dependency, zero references. |
| Throttled queue `debug!` lines | **Keep** as logs; they become redundant once the gauges are exported, but they are cheap and aid local debugging. |

#### 3.7 Naming and unit conventions already in use

A new naming convention must either match these or explicitly supersede them:

- Counters: `*_count`, `*_advances`, `*_completions`; cumulative totals use `total_*`.
- Gauges: `*_size`, `*_depth`, `*_lag`, `*_sequence`; instantaneous uses `current_*`; high-water marks use `max_*_seen`.
- Durations: `_ns` suffix universally; milliseconds appear only in human-readable output.
- Pipeline stage prefixes: `source_*`, `query_*`, `reaction_*`, with lifecycle verbs `receive`, `send`, `call`, `return`, `complete`, `start`, `end`.
- Scope (global / per-query / per-reaction) is conveyed by struct ownership, not by the name — which is exactly what labels will need to replace.

### 4. Proposed Metrics — Phase 0

Phase 0 is the metric set this design **commits to implementing**: thirteen metrics, chosen to be
the smallest surface that proves the architecture end to end — facade → recorder → exporter →
dashboard — while still being operationally useful on its own.

A further 61 candidate metrics across eleven categories are recorded in
[Appendix A](#appendix-a--future-metrics-p1p3) as forward-looking scope. Nothing in the appendix is
committed, and it is not part of what this document asks reviewers to approve.

Each row below carries:

- **Type** — `counter` (monotonic), `gauge` (up/down), `histogram` (distribution). These map onto
  OTel Sum / UpDownCounter / Histogram respectively.
- **Origin** — `new` (no equivalent exists), `promote` (a value already computed today, per §3, that
  only needs to be emitted through the facade), or `derive` (computable from `ProfilingMetadata`
  stamps that already exist).

#### 4.1 Standard Labels

Defined once, applied consistently across all phases. Cardinality is bounded by the number of
configured components, not by traffic — with the deliberate exception of `error_kind`, which must
be a small closed enum and never a free-form message.

| Label | Applies to | Values | In Phase 0 |
|---|---|---|---|
| `source_id` | source-scoped metrics | configured source name | yes |
| `query_id` | query-scoped metrics | configured query name | yes |
| `reaction_id` | reaction-scoped metrics | configured reaction name | yes |
| `component_kind`, `component_id` | metrics spanning component types | `source` \| `query` \| `reaction`, plus the id | yes |
| `change_kind` | result counters | `add` \| `update` \| `delete` | yes |
| `error_kind` | error counters | closed enum, never a message string | yes |
| `phase` | metrics valid in both phases | `bootstrap` \| `steady` | yes |
| `plugin_kind` | plugin-emitted metrics | e.g. `postgres`, `http`, `kafka` | no — Appendix A |

> **`phase` is not optional.** Nothing distinguishes bootstrap from steady-state traffic today, so
> bootstrap latency and steady-state latency mix into the same distribution. Bootstrap is bulk load
> and will dominate the tail, making the steady-state percentiles — the ones anyone alerts on —
> unreadable. Every latency and throughput metric must be separable by `phase`.
>
> **This is new work.** The only existing hint of the distinction is `ProfilingConfig`'s
> `include_bootstrap` flag, which is dead code and *gated* profiling rather than labelling it; it is
> being deleted (§5.6). `phase` requires a bootstrap marker carried on the event envelope, which
> does not exist.

#### 4.2 The Phase 0 Metric Set

Thirteen metrics, chosen to answer five operational questions:

| # | Metric | Type | Labels | Origin | Question answered |
|---|---|---|---|---|---|
| 1 | `drasi.component.up` | gauge | `component_kind`, `component_id` | new | Is it up? |
| 2 | `drasi.source.events_dispatched_total` | counter | `source_id`, `phase` | new | Is it doing work? |
| 3 | `drasi.query.events_processed_total` | counter | `query_id`, `phase` | new | Is it doing work? |
| 4 | `drasi.query.results_emitted_total` | counter | `query_id`, `change_kind` | new | Is it doing work? |
| 5 | `drasi.reaction.results_processed_total` | counter | `reaction_id`, `query_id` | new | Is it doing work? |
| 6 | `drasi.queue.depth` | gauge | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 7 | `drasi.queue.depth_max` | gauge | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 8 | `drasi.queue.blocked_enqueues_total` | counter | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 9 | `drasi.reaction.checkpoint_lag_events` | gauge | `reaction_id`, `query_id` | promote (§3.1) | Is it keeping up? |
| 10 | `drasi.queue.drops_total` | counter | `component_kind`, `component_id` | promote (§3.2) | Is it losing data? |
| 11 | `drasi.errors_total` | counter | `component_kind`, `component_id`, `error_kind` | new | Is it failing? |
| 12 | `drasi.query.engine_duration_seconds` | histogram | `query_id` | derive (interval D) | How slow is it? |
| 13 | `drasi.pipeline.end_to_end_duration_seconds` | histogram | `query_id`, `phase` | derive (A→G) | How slow is it? |

> **Metric 13 is deliberately *not* labelled `source_id` or `reaction_id`.** Those three labels
> multiply rather than add — see [§5.4](#54-cardinality-is-the-constraint-that-actually-bites),
> where the decision is made and the arithmetic given. Attributing a slow end-to-end path to a
> specific source–reaction pair is a tracing question, and [01 — Tracing](01-tracing.md) answers it
> exactly rather than statistically.

> **`drasi.queue.depth_max` reports a windowed maximum, and the reset is internal — not on read.**
> Each exported value is the high-water mark *for the last observation window*, not for all time; an
> all-time maximum stops being informative after the first burst. The underlying `max_depth_seen`
> field (§3.2) is monotonic, so a **single designated sampler inside Drasi** reads-and-resets it on
> the observation cadence (§5.3) and publishes the result as an ordinary gauge.
>
> ⚠️ **It must not reset when the metric is read**, which an earlier draft specified. Under the
> Phase 0 Prometheus **scrape** branch a "read" is a scrape, and scrape endpoints are not
> single-consumer: an HA Prometheus pair is a normal deployment, and an operator running `curl`
> during an incident is another reader. Destructive reads mean each consumer sees only what
> accumulated since whichever one read last, so the two Prometheus replicas would disagree and the
> `curl` would silently corrupt both. This is the same defect class as the histogram
> `clear_with()` steal in [§2](#2-collection-architecture), and it gets the same remedy:
> **one designated reader, idempotent export.**

Intervals D and A→G are defined in the pipeline model in
[00 — Overview](00-observability-overview.md#pipeline-model-and-instrumentation-points). Names
follow the convention in
[00 — Overview](00-observability-overview.md#naming-and-namespacing-conventions): dot-namespaced,
`_total` on monotonic counters, and durations in **seconds** rather than the `_ns` suffix used by
the internal fields in §3.

#### 4.3 Why These Thirteen

**Seven of the thirteen are promotions or derivations** of values drasi-lib already computes, so the
implementation is mostly wiring rather than new measurement. Of the six marked `new`, three
(`events_dispatched_total`, `events_processed_total`, `results_processed_total`) are a counter
increment at an existing dispatch point; only `drasi.component.up`,
`drasi.query.results_emitted_total` and `drasi.errors_total` require genuinely new bookkeeping.

**Every stage is represented** — source, query, reaction, and the queues between them — so the
label scheme, the handle-registration pattern, and the `phase` label all get exercised at every
kind of call site. A narrower cut would leave parts of the design unproven.

**All three signal shapes are present.** Counter, gauge and histogram each appear, so the recorder,
the aggregation story in §5, and histogram bucket configuration are all validated before the
remaining 61 candidates are considered.

Notes on individual choices:

- **`drasi.queue.drops_total` is the most important metric in this document.** Dropping events at
  capacity is silent data loss, and today it is visible only as a throttled `debug!` line (§3.2).
  Together with `depth`, `depth_max` and `blocked_enqueues_total` it promotes values that
  `lib/src/channels/priority_queue.rs` already maintains and that nothing currently reads.
- **`drasi.queue.depth_max` earns its place because `depth` alone cannot see a burst.** A queue that
  fills and drains between two exports is invisible to an instantaneous gauge, and that burst is
  exactly what an operator needs (§5.3). The value already exists, so the metric is free.
- **`drasi.query.results_emitted_total`**, labelled by `change_kind`, answers "is this query actually
  producing output, and of what shape". It has no equivalent today and is the cheapest way to
  distinguish a silent query from an idle source.
- **`drasi.reaction.checkpoint_lag_events`** is the single best consumer-health signal Drasi already
  computes and never exports.
  > **The `_events` suffix is doing real work.** The value is a *sequence-number difference* —
  > the query's latest outbox sequence minus the reaction's checkpoint (§3.1) — so it counts
  > **events behind**, not time behind. Named bare, `checkpoint_lag` reads as a duration to almost
  > every operator, and the [naming convention](00-observability-overview.md#naming-and-namespacing-conventions)
  > puts the unit in the leaf precisely to stop that. A time-based lag is not available as an
  > alternative: checkpoints carry sequence numbers, not timestamps, so there is nothing to subtract
  > to get seconds.
- **`drasi.component.up`** is deliberately trivial — it is the metric every dashboard and alert
  rule starts from.
- **The two histograms** derive from `ProfilingMetadata` stamps already written on every event
  (§3.3), so they add no hot-path timing call. They carry two prerequisites: the monotonic-clock
  fix in §5.6, and — for `end_to_end` — a bootstrap marker on the event envelope to populate
  `phase` (§4.1). Neither exists today.
  > Their **bucket boundaries are the embedder's**, and the Prometheus defaults are the wrong shape
  > for Drasi's microsecond-scale intervals — see [Open Issue 2](#open-issues) before charting
  > percentiles off either of these.

**DECIDED — one queue metric family, not two.** A single `drasi.queue.*` family labelled by
`component_kind` (`query` \| `reaction`) is used, rather than separate `drasi.query.queue_*` and
`drasi.reaction.queue_*` families. Backpressure is a system-wide concern: an operator wants one
chart showing every queue in the process, and one alert rule that fires regardless of which
component is saturated. This deliberately departs from the struct-ownership convention in §3.7,
where scope is encoded in the name — here scope moves into a label, which is the general direction
of §4.1.

#### 4.4 Errors

Phase 0 ships **one rollup error counter** rather than a counter per subsystem:

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `drasi.errors_total` | counter | `component_kind`, `component_id`, `error_kind` | Any component error |

The rules that govern it, and every error counter that follows in later phases:

- `error_kind` is a **closed enum**, never a message or a formatted string. This is the single
  largest cardinality risk in the design and the reason Phase 0 includes an error metric at all —
  the convention needs to be established and tested before it is replicated.
- Error counters live next to the operation that failed, so an error rate can always be divided by
  the corresponding throughput counter.
- Errors are additionally emitted as `tracing` events: the metric gives the rate, the trace gives
  the instance. The two are not redundant.

Later phases replace this rollup with per-subsystem counters — `drasi.plugin.errors_total` for the
plugin boundary ([§8.3](#83-tier-1--the-universal-baseline)),
`drasi.index.errors_total` for storage, and so on — all carrying the same `error_kind` discipline.


### 5. Sampling, Aggregation and Reporting Intervals


The instinct to apply a sampling rate carries over from tracing, where it is essential. It does not
transfer, because the two signals have different cost models.

#### 5.1 Why Sampling Is the Wrong Lever for Metrics

**Metric export volume is independent of event throughput.** A counter incremented a million times
per second still exports exactly one value per reporting interval. A trace, by contrast, produces
one span per event — its volume is directly proportional to throughput, which is precisely why
traces must be sampled. For metrics, the quantity that determines volume is
`series_count × reporting_frequency`, and event rate does not appear in it.

So the levers that actually control metric cost are:

| Lever | Controls | Where settled |
|---|---|---|
| **Reporting interval** | Export volume and downstream storage | §5.2 |
| **Label cardinality** | Series count — the dominant term | §4.1, Open Issue 1 |
| **Enablement / profiles** | Which metrics exist at all | [00 — Overview](00-observability-overview.md#enablement-filtering-and-observability-profiles) |
| **Observation cadence** | Cost of *reading* expensive gauges | §5.3 |

Sampling appears in none of them.

##### How the profiles map onto this catalogue

The profile vocabulary is defined in
[00 — Overview](00-observability-overview.md#enablement-filtering-and-observability-profiles). What
belongs here is the mapping onto the phase tiers this document defines, because **the profile ladder
and the phase ladder are deliberately the same ordering** — one ranking of importance, used twice
(release order in §4 and Appendix A, runtime exposure here).

| Profile | Metric set | Series, mid-sized deployment | Notes |
|---|---|---|---|
| `off` | none | 0 | No recorder installed; facade calls are no-ops |
| `basic` | Phase 0 (§4.2) | ~1,700 (§5.4) | The default. Sized to be safe unattended |
| `debug` | + P1 and P2 (Appendix A) | Highest | Includes expensive gauges — see §5.3 for why cadence must stay decoupled from export |
| `persistence` | Phase 0 + `drasi.index.` + engine statistics | ~`basic` + backend surface | Crosses tiers rather than extending the ladder; requires collection enabled in the provider constructor, not just a filter (§6.2) |

There is deliberately **no intermediate rung between `basic` and `debug`.** Splitting P1 out as its
own profile would force a per-metric argument about which side of the line each one falls on, and an
operator who has already decided `basic` is insufficient is usually diagnosing something and wants
the full picture. The P1/P2 distinction still governs *release* order in Appendix A, where it costs
nothing.

Two consequences specific to metrics:

- **`persistence` is the only profile that cannot be satisfied by filtering alone.** Every other
  profile adds or removes series that are already being produced; `persistence` requires RocksDB
  `Statistics` to have been switched on before drasi-lib received the provider. §6.2 covers the
  exposure question, and the ownership rule is in 00 — Overview.
- **No profile changes sampling, because there is none to change.** Profiles select *which* metrics
  exist; §5.1 through §5.4 govern what each one costs once selected. A profile that appeared to
  "reduce sampling" would be selecting a smaller metric set, and should say so.

**Sampling also breaks the metrics that matter most.** `drasi.queue.drops_total` and
`drasi.errors_total` are
counters of rare, important events. Sampling at 1% means a single dropped event is 99% likely to be
invisible, and the reported count is an estimate of a number that must be exact. A metric whose
entire purpose is to detect silent data loss cannot itself lose data. The same argument applies to
`drasi.reaction.checkpoint_lag_events` and `drasi.component.up`, where a sampled reading is simply a stale
reading.

**Counters and histograms are already aggregations.** Incrementing a counter is one relaxed atomic
add. Recording a histogram sample is a bucket index plus an atomic add. Neither allocates and
neither grows with cardinality of the data. There is no accumulating cost for sampling to relieve.

#### 5.2 Reporting Interval — the Real Volume Control

| Setting | Default | Rationale |
|---|---|---|
| Export interval | **10s** | Between the Prometheus default scrape (15s) and the OTel periodic reader default (60s). Fast enough that a 30-second saturation episode is visible in three data points |
| Configurable range | 1s – 300s | 1s for debugging a live incident; 300s for cost-sensitive long-horizon collection |
| Applies to | all metrics uniformly in Phase 0 | Per-metric intervals are a Phase 1 concern and add configuration surface for little gain at 13 metrics |

Reducing the interval increases downstream volume linearly and costs nothing extra in-process,
because the values are being maintained continuously either way. This is the knob to reach for if
volume becomes a problem — not sampling.

#### 5.3 Gauge Observation Cadence

Gauges are the one place where something *resembling* sampling is unavoidable: a gauge is read at
export time, so its value between exports is unobserved. That is an observation-cadence question,
not a statistical-sampling one, and it has a real consequence for Phase 0:

> **`drasi.queue.depth` read every 10s will miss short saturation bursts.** A queue that fills and
> drains within 200ms is invisible at a 10s cadence, yet it is exactly the event an operator needs
> to see. The instantaneous gauge alone is not sufficient evidence of backpressure.

**DECIDED — `drasi.queue.depth_max` ships in Phase 0** as the thirteenth metric (§4.2), reporting the
high-water mark *for the last observation window* rather than for all time. The value is already
maintained as `max_depth_seen` (§3.2), so the metric costs nothing to produce.

**The read-and-reset belongs to one designated sampler inside Drasi, on the observation cadence —
not to the exporter.** The distinction matters because the two readers have incompatible
requirements: the sampler *must* reset to produce a per-window figure, while the export path *must*
be idempotent, since under Prometheus pull it can be invoked by an HA replica pair and an ad-hoc
`curl` at the same time. Separating them satisfies both; conflating them makes every extra reader
steal data from the others. The same separation is the recommended answer for histogram reads in
Open Issue 4.

The two alternatives were considered and rejected as insufficient on their own:

- `drasi.queue.blocked_enqueues_total` and `drasi.queue.drops_total` are counters and therefore lose
  nothing — but they only fire once the queue is *already* full, so they detect saturation rather
  than approach to it. They remain in Phase 0, but as a last line rather than an early warning.
- Shortening the export interval is expensive, applies globally, and still only narrows the window
  rather than closing it.

For Phase 2 gauges that are genuinely expensive to read — RocksDB statistics and on-disk index
sizes — the observation cadence should be **decoupled** from the export interval and default to
something longer (30–60s), so a 1s export interval for debugging does not start interrogating the
storage engine a thousand times a minute.

#### 5.4 Cardinality Is the Constraint That Actually Bites

Series count, not sampling, is what determines whether this design scales. For a mid-sized
deployment of 5 sources, 20 queries and 10 reactions, metrics 1–12 produce roughly 1,200 time
series — comfortable for any backend.

With one exception. **`drasi.pipeline.end_to_end_duration_seconds` was originally specified
multiplicatively**: labelling it `source_id` × `query_id` × `reaction_id` yields 5 × 20 × 10 = 1,000
label combinations, and a histogram multiplies that by its bucket count (12 in this sizing) —
roughly 12,000 series from a single metric, ten times the rest of Phase 0 combined. Worse, it grows
as the *product* of deployment size rather than the sum, so it degrades fastest on exactly the
deployments that can least afford it.

> **DECIDED: label `drasi.pipeline.end_to_end_duration_seconds` with `query_id` and `phase` only.**
> End-to-end latency is charted per query in practice; attribution to a specific source–reaction
> pair is a tracing question, and [01 — Tracing](01-tracing.md) already answers it exactly rather
> than statistically. The metric becomes 20 queries × 2 phases × 12 buckets ≈ **480 series** — a 25×
> reduction, and one that now grows linearly with query count instead of as a three-way product.
>
> `phase` is retained even though it doubles the count, because it is the one dimension that must
> never be mixed: bootstrap is bulk load and would otherwise dominate the steady-state tail (§4.1).

With metric 13 corrected, Phase 0 totals roughly **1,700 series** for the sizing above.

#### 5.5 What Existing Instrumentation Should Do

| Existing mechanism | Sampling disposition |
|---|---|
| `lib/src/metrics/` atomic structs (§3.1) | Raw. Already exact, already cheap, no change |
| `PriorityQueueMetrics` (§3.2) | Raw. These are the loss-detection metrics — sampling them would defeat their purpose |
| `ProfilingMetadata` stamping (§3.3) | Unconditional, as it already is. The per-event cost is clock reads, addressed in §5.6 |
| Profiler Reaction `sampling_rate` | **Stays a Profiler-only control.** It samples which events get statistically summarised in the Profiler's own sliding window. It must not become a metrics control, and the exported histograms must not be placed behind it |

#### 5.6 Clock Source for Latency Histograms

The latency histograms in §4.2 have **no gating dependency** — the stamps they derive from are
already written on every event, because `ProfilingConfig` is never consulted (§3.3) — and it is
being deleted, so no gate will appear later either. What they do
have is a clock-source problem.

`profiling::timestamp_ns()` returns `SystemTime::now().duration_since(UNIX_EPOCH)`. `SystemTime` is
wall-clock and **not monotonic**: an NTP step or a manual clock change can move it backwards. A
derived interval can therefore be negative — which, computed on unsigned integers, becomes an
absurdly large positive value that permanently corrupts a histogram's maximum and upper
percentiles. A single clock adjustment can poison a p99 for the lifetime of the process. This is
tolerable for a Profiler log line a human reads and discards; it is not tolerable for an exported
histogram an alert fires on.

| Option | Consequence |
|---|---|
| **Keep `SystemTime`, clamp negatives to zero** | Minimal change, but silently under-reports across a clock step and still cannot distinguish a clamp from a genuinely fast event |
| **Add a parallel `Instant` stamp for intervals** | Correct durations; `Instant` is monotonic and cheaper to read. Costs an extra field per stamp point, and `Instant` is not serializable across the FFI boundary or across processes |
| **`Instant` for intra-process intervals, `SystemTime` retained for cross-process correlation** | Correct and complete, most work — durations come from `Instant`, absolute event times stay `SystemTime` for correlating with an upstream source's timestamps |

> **DECIDED: the third — two clocks, each used for what it is correct at.** Intervals A–G are all
> intra-process, so every duration is computed from `Instant`, which is monotonic by construction
> and cannot produce a negative interval no matter what NTP does. `SystemTime` is retained *only*
> for absolute event times — `source_ns` / `reactivator_*_ns` — which can only ever be wall-clock
> because they originate outside the process and must line up with an upstream system's timestamps.
>
> The rule that follows is short enough to enforce in review: **never subtract two `SystemTime`
> stamps to get a duration.** If a duration is wanted, there must be an `Instant` pair for it.
>
> Two honest costs. The `Option<u64>` wall-clock fields stay as they are, so `ProfilingMetadata`
> grows by the `Instant` stamps rather than swapping them in — the struct gets wider on a per-event
> path. And the cross-process half is **provisioning, not preservation**: `source_ns` and
> `reactivator_*_ns` are never populated today (§3.3), so retaining `SystemTime` protects a
> capability Drasi does not yet have. That is deliberate — the alternative is discovering the need
> after the stamp format is load-bearing.

> **Consequence for `ProfilingConfig`: DECIDED — delete it.** `ProfilingConfig` and
> `should_profile()` have zero callers, so deleting them changes no behaviour; it only removes a
> configuration surface that advertises an opt-in and a sampling rate Drasi does not implement.
> Wiring it up instead would put a gate in front of the very stamps the Phase 0 histograms derive
> from, which §5.5 rules out. Sampling control for the Profiler Reaction stays where it already
> lives, in the Profiler's own `sampling_rate` (§5.5).
>
> One thing the deletion does *not* solve: `include_bootstrap` was the only existing hint of a
> bootstrap/steady distinction, and it gated profiling rather than labelling it. The `phase` label
> (§4.1) still needs a bootstrap marker carried on the event envelope — new work that the deletion
> neither creates nor removes.

#### 5.7 Export Temporality

✅ **DECIDED: cumulative temporality for Phase 0.**

The choice matters only on a crash or a failed export, and there the two options are not
comparable. A **cumulative** counter reports an absolute running total, so a missed export is
self-correcting — the next successful one carries everything the lost one would have. A **delta**
counter reports only what changed since the last export, so a missed export is permanent data loss
with nothing to indicate it happened.

Under the Prometheus-scrape branch this is automatic: a scrape reads current values, so cumulative
is the only thing it can be. That is one more reason scrape is the right P0 default (§2). If an
OTLP push branch is added later, it must be configured cumulative **explicitly** — the OTLP
specification permits either, and some backends default to delta.

Crash-loss windows for the other signals are in
[00 — Overview](00-observability-overview.md#export-flush-and-crash-loss-semantics).

### 6. Environment and Storage Metrics

#### 6.1 Process and Runtime Metrics — Why Not in drasi-lib

drasi-lib runs inside someone else's process. RSS and CPU measure the host application plus Drasi,
so a `drasi.*` process metric emitted by the library would be a wrong number, not an unavailable
one. A library also has no standing to spawn a background collection task the host did not ask for,
or to publish series that collide with ones the host already exports.

Attributing memory to Drasi specifically would require allocator accounting, and that route is
closed for two independent reasons:

1. **Custom global allocators are banned by design.** Rich types cross the cdylib plugin boundary
   as owned `Box`es and are sound only because plugin and host share the process-global System
   allocator. `drasi-core`'s `deny.toml` enforces this with a `cargo-deny` `[bans]` rule listing
   twelve allocator crates (issue #378), so `jemalloc` statistics — the standard answer — are
   unavailable.
2. **A library can never set `#[global_allocator]`** — only the final binary can. This is the one
   genuine capability difference between drasi-lib and a Drasi binary, and reason 1 makes it moot.

Consequences for this design:

- **drasi-lib does not emit process metrics by default.** An embedder who knows the process is
  effectively dedicated to Drasi may enable them through an explicit opt-in — only the embedder can
  know that.
- **Per-query, per-component and per-index memory attribution is not achievable** and must not be
  implied anywhere in this document. Entry counts and on-disk index sizes ([A.10](#a10-storage-index-state-store-and-wal))
  are proxies, not memory.

Process metrics for the Drasi Server deployment — collection mechanism, metric names,
configuration and platform coverage — are designed in
[Drasi Server observability §4](../../drasi-server/tracing-logging/00-observability-integration.md#4-process-resource-metrics),
which specifies the `metrics-process` collector and the standard `process_*` family. This document
does not restate them.

#### 6.2 Storage Backend Statistics

**DECIDED — it depends on whether the backend runs inside the Drasi process, and the split is not
a preference but a consequence of what each observer can actually see.**

Two different things get called "storage metrics", and they have different owners:

| | What it is | Who can produce it |
|---|---|---|
| **Interaction metrics** | Drasi's use of the backend: operation latency as Drasi experiences it, operation counts, errors — labelled by `query_id`, `index_kind`, `operation` | **Only Drasi.** A backend's own exporter sees requests aggregated across all clients and cannot attribute them to a query |
| **Backend-internal metrics** | The engine's own state: cache hit ratio, compaction, memtable size, keyspace, server memory | Depends on where the backend runs |

Interaction metrics ([A.10](#a10-storage-index-state-store-and-wal)'s `drasi.index.*`) always flow
through Drasi, for every backend. They are the reason this section exists at all: today a slow
query cannot be attributed to the engine versus the storage layer, and no external exporter can
close that gap.

Backend-internal metrics divide by deployment topology:

| Backend | Where it runs | Backend-internal metrics |
|---|---|---|
| In-memory | in-process | Drasi — entry counts only; **no memory attribution**, per §6.1 |
| RocksDB (`components/indexes/rocksdb`) | in-process, on-disk | **Through Drasi.** Nothing else can see them |
| redb (`components/wals/redb`, state store) | in-process, on-disk | **Through Drasi.** Nothing else can see them |
| Garnet / Redis (`components/indexes/garnet`) | **external server**, reached over the Redis protocol | **Scraped directly.** Drasi does not proxy them |

> RocksDB and redb store their data *outside* the process — a directory or mounted volume that
> outlives it — but **execute inside it**, linked in as libraries with no daemon and no network
> endpoint. Persistent data is not the same as an external system: there is nothing to scrape, so
> Drasi is the only possible exporter. Garnet is the only backend here that is a separate process.

#### Why Drasi must not proxy external backend metrics

For an embedded engine there is no choice — RocksDB statistics exist only inside the Drasi process,
so Drasi is the only possible exporter. For Garnet the opposite holds, and proxying would be a
mistake for five reasons:

1. **Attribution — the same argument as §6.1.** A Garnet instance may serve several queries,
   several Drasi instances, or an unrelated workload. Republishing its global keyspace and memory
   under a `drasi.*` name attributes someone else's numbers to Drasi. That is a *wrong* number,
   not a missing one.
2. **Reinvention.** Mature exporters already exist for the Redis protocol and are far more
   complete than anything Drasi would curate, without Drasi having to track upstream changes.
3. **A new failure mode.** Drasi would become a metrics proxy, so backend visibility disappears
   exactly when Drasi is unhealthy — the moment it is most needed. It also adds Drasi's own
   collection interval on top of the scrape interval, making every value staler.
4. **Volume.** A full `INFO` response is hundreds of fields. Pumping that through Drasi's own
   pipeline is precisely the "telemetry overloading Drasi" risk §5 exists to prevent.
5. **Trust boundary.** Proxying republishes another system's internals through Drasi's endpoint,
   potentially to consumers who are not authorised to see the backend directly.

#### The real requirement is correlation, and it is solved with a label

The legitimate need behind "flow it through Drasi" is wanting Drasi latency and backend health on
one screen. That belongs in the **observability backend**, not in the collection path — Grafana
joins two series far better than Drasi proxies one.

What Drasi must supply is the **join key**. Every `drasi.index.*` metric carries a label
identifying the backend instance the index is bound to:

| Label | Value | Cardinality |
|---|---|---|
| `index_kind` | `memory` \| `rocksdb` \| `garnet` \| `redb` | 4 |
| `backend_instance` | host:port for networked backends, path for on-disk, `inline` for in-memory | one per configured backend |

A dashboard then joins `drasi.index.operation_duration_seconds{index_kind="garnet"}` against the Redis
exporter's series on that endpoint. This is cheap, bounded, and it is the actual deliverable — not
a proxy.

> **Secret hygiene.** `backend_instance` is derived from connection configuration, which may embed
> credentials. It must be the sanitised host:port only. A label is high-visibility — it lands in
> every scrape and every dashboard — so this needs an explicit test.

#### Exposing embedded-engine statistics

For RocksDB and redb, where Drasi is the only possible exporter. **The two engines turned out to be
nothing alike**, so they get separate answers rather than a shared policy.

##### RocksDB — cheap typed properties, and one gap caused by the pinned version

`components/indexes/rocksdb/Cargo.toml:32` pins **`rocksdb = "0.21.0"`**, and that version's
statistics surface is narrower than a reader of the RocksDB C++ docs would expect:

| API | In 0.21? | Notes |
|---|---|---|
| `DB::property_int_value(name)` (`db.rs:1840`) | **Yes** | Typed `u64`. Needs no `enable_statistics()`, so it carries none of its overhead |
| `Options::enable_statistics()` (`db_options.rs:2517`) | Yes | |
| `Options::get_statistics() -> Option<String>` (`db_options.rs:2523`) | Yes | A **formatted text blob**. Parsing it to get numbers is the only route to tickers on 0.21 |
| `get_ticker_count()`, `get_histogram_data()`, `set_statistics_level()` | **No** | Arrive in 0.22. ⚠️ An earlier draft of this section specified statistics *levels* (`kExceptDetailedTimers`, `kExceptTimeForMutex`, `kAll`) as configuration — **that is not implementable on the pinned version.** |

So the curated set is mostly reachable through `property_int_value`, which is the cheap path:

| Curated statistic | Property (0.21) | Status |
|---|---|---|
| SST bytes on disk | `total-sst-files-size`, `live-sst-files-size` | ✅ typed `u64` |
| Memtable size | `cur-size-all-mem-tables`, `size-all-mem-tables` | ✅ typed `u64` |
| Pending compaction bytes | `estimate-pending-compaction-bytes` | ✅ typed `u64` |
| Write stall | `actual-delayed-write-rate`, `is-write-stopped` | ✅ typed — but a *rate and a flag*, not cumulative stall time |
| **Block cache hit ratio** | — | ❌ **Not available typed on 0.21.** Only `block-cache-usage` and `block-cache-capacity` (occupancy, not hit rate). Hit/miss needs tickers |

Worth adding while we are here, all typed and cheap: `background-errors`, `compaction-pending`,
`num-running-compactions`, `estimate-num-keys`.

> **DECIDED — build the curated set on `property_int_value`, and treat block cache hit ratio as
> blocked rather than dropped.** Four of the five proposed statistics are cheap typed reads that do
> not require `enable_statistics()` at all, which removes the overhead concern that made this
> opt-in in the first place — so **the property-derived metrics can ship without a flag**. Hit ratio
> is the single most operationally useful of the five and the only one that cannot ship: it needs
> either a bump to `rocksdb 0.22` for `get_ticker_count()`, or parsing the `get_statistics()` text
> blob, which is a fragile thing to put on a timer. **Recommend the version bump**, alongside the
> OpenTelemetry version decision in
> [01 — Tracing](01-tracing.md#t2--the-host-side-and-a-conflict-with-facade-only), and ship the
> other four meanwhile.

##### redb — eight numbers, and reading them is a blocking full scan

redb 2.6.3 backs the **state store** (`components/state_stores/redb`) and the **WAL**
(`components/wals/redb`). It is **not** an index backend, so `drasi.index.*` is the wrong namespace
for it — its metrics belong under `drasi.state_store.*` / `drasi.wal.*`
([A.10](#a10-storage-index-state-store-and-wal)).

`DatabaseStats` (`src/transactions.rs:300-351`) exposes exactly eight values: `tree_height`,
`allocated_pages`, `leaf_pages`, `branch_pages`, `stored_bytes`, `metadata_bytes`,
`fragmented_bytes`, `page_size`. `TableStats` (`src/table.rs:29-55`) offers a per-table subset.
**There are no cache hit/miss counters, no operation counters and no latency histograms** — redb
reports *shape*, never *behaviour*.

The surface is not the problem. The access path is:

| Finding | Evidence |
|---|---|
| `stats()` exists **only on `WriteTransaction`** | `transactions.rs:2282`, inside `impl WriteTransaction` at `:797`. `ReadTransaction` has no `stats()` at all |
| redb permits **one writer at a time** | `Database::begin_write()` docs, `db.rs:1022`: *"Only a single write may be in progress at a time. If a write is in progress, this function will block until it completes."* |
| `stats()` is a **full traversal of every page** | `TableTree::stats()` iterates every table (`table_tree.rs:807`, `range::<RangeFull, &str>`) and `stats_helper` recurses through every leaf and branch page (`btree.rs:950`) |

Together those mean a metrics sampler calling `stats()` would **take the write lock the WAL needs
and hold it for an O(database size) scan that pulls the entire file through the page cache.** On the
WAL — the component on the write path — that is a latency injection dressed as observability.

> **DECIDED — never sample redb's `stats()` on a timer.**
>
> - **Size comes from the filesystem.** `drasi.wal.size_bytes` and the state store's size are
>   obtained by stat-ing the database file, which costs nothing and needs no transaction. This is
>   what [A.10](#a10-storage-index-state-store-and-wal) already specifies (*"from file sizes"*).
> - **Behaviour comes from Drasi's own instrumentation.** `drasi.wal.append_duration_seconds` and
>   `drasi.state_store.operation_duration_seconds` are interaction metrics that Drasi times itself
>   (§6.2) — they need nothing from redb, and they are what an operator actually acts on.
> - **`fragmented_bytes` is genuinely useful and still does not become a metric.** It is the
>   signal for "this database wants compacting", but it is only obtainable via the blocking scan.
>   It belongs in an **on-demand admin/diagnostic operation** an operator invokes deliberately,
>   never on an interval.
>
> The earlier guess — *"redb may offer very little, in which case it gets `disk_bytes` and nothing
> more"* — reached roughly the right conclusion for the wrong reason. redb offers eight numbers; the
> reason we decline almost all of them is the cost of reading them, not their absence.

##### What survives as shared policy

- **Backend-neutral names only where the semantics genuinely match.** A size-on-disk gauge is safe
  across every on-disk backend. Anything engine-specific keeps an engine-specific name
  (`drasi.index.rocksdb.*`) rather than being forced into a shared name that means something
  slightly different per backend.
- **Read on a timer, not per-operation**, on a cadence decoupled from the export interval per §5.3
  — and only for reads that are *actually* cheap, which after this survey means RocksDB properties
  and `stat()`, not redb's `stats()`.
- **Verbatim pass-through stays available behind a flag** as an escape hatch for deep debugging.

### 7. Future: Internal Telemetry as a Drasi Source

Not phase 1, recorded for direction. Internal Drasi metrics could be exposed through a generic
telemetry/OTel **Source**, allowing continuous queries over operational data — e.g. reacting when
load crosses a threshold or processing deteriorates. A local/flagged mode would read local
telemetry directly rather than looping back over the network. A telemetry-source proposal already
exists in the mobility demo proposal and should be cross-referenced rather than duplicated. This
depends on the collection architecture chosen in §2, and would require the throttling described in
§5 so Drasi cannot overload itself with its own metrics.

### 8. Plugin Metrics

**DECIDED — telemetry is a standard capability of every plugin kind, structured as three tiers: a
universal baseline every plugin gets, a per-kind standard set, and whatever the plugin author adds
on top. All three land in the same recorder.**

The FFI transport these tiers ride on is described in
[00 — Overview](00-observability-overview.md#plugin-telemetry-across-ffi); this section defines what
is actually emitted.

#### 8.1 The plugin kinds

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
[§8.6](#86-how-tiers-2b-and-3-reach-the-recorder).

#### 8.2 The three tiers

Tiers 1 and 2 are **guaranteed** — they exist for a plugin whose author wrote no instrumentation at
all. Tier 3 is the author's own, and is optional.

| Tier | What | Who emits it | Applies to |
|---|---|---|---|
| **1 — Universal baseline** | Same handful of metrics for every plugin, whatever its kind | drasi-lib, automatically | all 8 kinds |
| **2 — Per-kind standard set** | Metrics meaningful for *that* kind: source metrics, reaction metrics, bootstrapper metrics, … | drasi-lib where derivable; plugin where not | all 8 kinds |
| **3 — Plugin-author metrics** | Whatever the author considers useful about their own internals | the plugin, opt-in | any kind |

**All three tiers land in the same recorder.** For a statically linked plugin the `metrics` macros
resolve to the host's global recorder directly; for a cdylib plugin they resolve to the plugin's
own recorder, which is an FFI bridge that forwards to the host's. The destination is identical
either way, so a dashboard cannot tell how a plugin was linked — which is the point.

#### 8.3 Tier 1 — the universal baseline

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
> dynamic plugins only — and **index, state-store and WAL plugins have no FFI path at all**
> (`lib`-only crates, no `export_plugin!`), so they would silently report nothing, permanently. That
> reason does not expire with the move to dynamic-only plugins the way "most components ship
> statically today" would.

#### 8.4 Tier 2 — the per-kind standard set

Tier 1 is deliberately semantic-free: it knows a call happened, not what it meant. Tier 2 adds the
metrics that are meaningful for a specific kind, so that every source reports the same things as
every other source and a Postgres source can be compared against a Kafka one.

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
[§6.2](#62-storage-backend-statistics) argues only Drasi can produce, and
[A.10](#a10-storage-index-state-store-and-wal) catalogues. They are not a separate mechanism —
storage backends are plugins, and the decorator is where their interaction metrics come from.

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
`ReactionMetrics` — so the author fills in values rather than inventing names.

#### 8.5 Tier 3 — plugin-author metrics

Anything else the author wants to measure about their own internals — WAL parse time, change-feed
decoding, retry loops, cache hits. The author uses the standard `metrics` macros and the SDK routes
them to the same recorder as tiers 1 and 2.

#### 8.6 How tiers 2b and 3 reach the recorder

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
the host. This is why plugin metrics do **not** flow through the recorder stack an embedder
installs in the host: the plugin links its own copy of the `metrics` crate and has its own global
slot (§2).

Had this been hung off `FfiRuntimeContext` instead — the natural-looking place, since the
per-component log callback lives there — tiers 2b and 3 would have been available to sources and
reactions only.

**ABI rule.** New fields append to the end of `FfiPluginRegistration` and the host gates access on
the plugin's reported `sdk_version`, exactly as `identity_provider_plugins` and `set_log_level`
already do — reading a trailing field from a plugin that allocated the older, smaller struct is
undefined behaviour. `validate_plugin_metadata` additionally requires an exact `major.minor` match,
so an SDK-version bump rejects stale plugins outright.

#### 8.7 Attribution: what plugin-emitted metrics cannot label

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

#### 8.8 Namespace governance

Tier 3 is open-ended, so it is the one place a third-party plugin could collide with a Drasi metric
name or squat on `drasi.source.*`. Two rules:

- **The SDK issues pre-labelled handles.** A plugin obtains its tier 2b and tier 3 handles from an
  SDK-provided emitter that has already captured the plugin kind and (where available) the
  component id. This solves naming and attribution together, and is why a plugin author does not
  hand-write labels.
- **The bridge enforces the prefix.** `FfiMetricsRecorder` prefixes anything a plugin emits that is
  not a declared tier 2b name, so tier 3 metrics land under `drasi.plugin.<plugin_kind>.*` and
  cannot shadow a first-party name.

> **DECIDED — the bridge is the enforcement point, and it is on every path that can carry a
> third-party plugin.** Prefix enforcement lives in `FfiMetricsRecorder` and therefore applies to
> cdylib plugins only. That is sufficient because sources, reactions, bootstrappers, identity
> providers and secret stores are moving to **dynamic-only**, so the bridge is unavoidable for them.
>
> ⚠️ **Indexes, state stores and WALs are the permanent exception** — they declare no `cdylib`
> crate-type and no `export_plugin!`, so they have no FFI path for a bridge to occupy. Their tier 1
> metrics are still fully attributed, because those come from the drasi-lib decorator above rather
> than from the plugin itself; only a tier 3 metric such a plugin emits for itself is ungoverned.
> Acceptable while those remain first-party (Garnet, RocksDB, redb). See
> [Open Issue 3](#open-issues) for the reopening condition.

> **Consequence for naming.** The convention `drasi.<component_type>.<plugin_kind>.<metric>`
> assumes a pipeline component type, which identity providers and secret stores do not have. Tier 1
> therefore uses a single `drasi.plugin.*` family with `plugin_kind` and `component_kind` as
> **labels**, consistent with the `drasi.queue.*` decision in §4.3; tier 2 uses a per-kind prefix
> (`drasi.source.*`, `drasi.bootstrap.*`, …).

#### 8.9 Enablement

Emission is always optional for the plugin author — a plugin that instruments nothing is valid, and
tiers 1 and 2a still report it. `set_log_level` is the precedent for host-controlled filtering: the
host reports its effective level and the plugin drops records *before formatting or forwarding
them*. A metrics equivalent should follow, so that a disabled metric costs a filter check inside
the plugin rather than an FFI crossing.

#### 8.10 Worked example: adding metrics to a source plugin

Suppose someone is writing a source that subscribes to a remote change feed and reacts to frames as
the upstream system pushes them. Before they write a single line of instrumentation they already
get tier 1 and tier 2a, because drasi-lib emits those from the decorator wrapping their plugin.
What follows is only what they add on top.

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
    decode_duration_seconds: Histogram,
    heartbeats: Counter,
}
```

**Step 2 — obtain them from the runtime context.** `context.metrics()` returns an emitter that has
already captured `plugin_kind` and `component_id`, so the author never writes a label. This is also
what keeps tier 3 names inside the plugin's own namespace.

```rust
async fn initialize(&self, context: SourceRuntimeContext) {
    let m = context.metrics();
    let _ = self.metrics.set(ChangeFeedMetrics {
        std:                     m.source(),                    // drasi.source.*
        frames_received:         m.counter("frames_received"),  // drasi.plugin.changefeed.frames_received
        decode_duration_seconds: m.histogram("decode_duration_seconds"),
        heartbeats:              m.counter("heartbeats"),
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
        m.std.upstream_lag_seconds.set(frame.age().as_secs_f64());   // tier 2b

        let started = Instant::now();
        let changes = self.decode(frame)?;
        m.decode_duration_seconds.record(started.elapsed().as_secs_f64());
        m.std.batch_size.record(changes.len() as f64);   // tier 2b

        for change in changes {
            self.dispatch_change(change).await?;
        }
    }
    Ok(())
}
```

Three things worth noting about what the author did *not* write:

- **No labels.** `source_id` / `plugin_kind` are attached by the emitter, so they cannot be
  forgotten, misspelled, or made unbounded.
- **No error counter for the interrupted stream.** The tier 1 decorator already recorded
  `drasi.plugin.errors{operation="start"}` if the call returns `Err`. A plugin should add an error
  counter only for failures it *handles internally* and never surfaces to the host — which is
  exactly the case above, where the loop resubscribes and continues.
- **No exporter, no recorder, no configuration.** Whether these land in Prometheus or OTLP, and
  whether this plugin is statically linked or loaded from a `.so`, is decided by the host.

**For plugin kinds with no runtime context** — bootstrap, identity, secret store — there is no
`context.metrics()`, so the SDK exposes a library-scoped emitter instead. It carries `plugin_kind`
but not `component_id`, per §8.7:

```rust
let m = drasi_plugin_sdk::metrics::plugin_metrics();  // no component_id available
let cache_hits = m.counter("cache_hits");             // drasi.plugin.vault.cache_hits
```

**The raw macros still work.** `metrics::counter!("anything")` reaches the same recorder — the
emitter is a convenience and a governance mechanism, not a gate.

### 9. Naming and Namespacing Conventions

**DECIDED — dot-namespaced, lowercase, with the unit and the counter suffix written into the leaf.
Service identity lives in a resource attribute, never in the metric name.**

The two conventions worth following disagree with each other, so the choice turns on one
implementation fact about our stack rather than on taste.

#### 9.1 What the established conventions actually say

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

#### 9.2 The fact that decides it

Under the OpenTelemetry SDK the disagreement is a non-issue: you write the OTel name, set the unit
in metadata, and the Prometheus exporter mechanically produces the Prometheus name — replacing `.`
with `_`, appending the unit word, and appending `_total` to monotonic sums. Both conventions are
satisfied because the exporter translates between them.

**Drasi does not emit through the OpenTelemetry SDK.** It emits through the `metrics` facade (§2),
and `metrics-exporter-prometheus` performs *no* such translation. Its documented name handling is
limited to replacing invalid characters with `_`; its `formatting` module offers only
`sanitize_metric_name`, `write_help_line`, `write_type_line` and `write_metric_line`. There is no
unit suffixing and no `_total` appending. `metrics::Unit` exists and `describe_histogram!` records
it, but the Prometheus exporter does not use it to build the name.

So **the name we write is, after `.` → `_` substitution, the name that ships**. Nothing downstream
will add what we leave out. If we follow OTel's "no unit in the name" rule, Prometheus receives
`drasi_query_engine_duration` — no unit, no type — which is precisely the ambiguity Prometheus
warns about, and the `Unit::Seconds` we carefully declared is silently discarded.

#### 9.3 The convention

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

#### 9.4 Namespace layout

| Namespace | Owner |
|---|---|
| `drasi.source.*`, `drasi.query.*`, `drasi.reaction.*` | pipeline stages |
| `drasi.pipeline.*` | metrics spanning the whole pipeline |
| `drasi.queue.*` | backpressure, labelled by `component_kind` |
| `drasi.component.*` | lifecycle, any component type |
| `drasi.plugin.*` | the universal plugin baseline (tier 1, §8.3) |
| `drasi.plugin.<plugin_kind>.*` | plugin-author metrics (tier 3), prefix enforced by the bridge |
| `drasi.index.*`, `drasi.state_store.*`, `drasi.wal.*` | storage interaction |
| `drasi.index.rocksdb.*` | engine-native stats, kept engine-specific by design |
| `process_*` | **exception** — ecosystem-standard, no `drasi` prefix |

Engine-specific statistics keep an engine-specific namespace rather than being forced into a shared
name, following OTel's own reasoning for preferring `jvm.gc.*` over `gc.*`: implementations differ
enough that a shared name invites false comparison.

#### 9.5 Service identity is a resource attribute, not a prefix

**`drasi-lib` and `drasi-server` are NOT differentiated in the metric name.** They are distinguished
by the OTel `service.name` resource attribute, or an equivalent global label on the Prometheus
exporter — which is what those mechanisms exist for.

**Specifically, there is no `drasi.lib.*` namespace.** That was considered and rejected: it names
metrics by *the crate that compiled them* rather than by *what they measure*, and both variants fail
differently.

| Variant | What breaks |
|---|---|
| Flat `drasi.lib.*` | Pipeline latency, checkpoint/reliability and component lifecycle all collapse into one namespace, losing the grouping that makes the set browsable |
| Per-module `drasi.lib.<module>.*` | The opposite failure: intervals A–G are **one ordered flow** that an operator reads as a single row, but they would scatter across `drasi.lib.sources.*`, `.channels.*` and `.queries.*` |

The domain layout above avoids both — `drasi.pipeline.*` keeps the end-to-end flow together while
`drasi.source.*` / `drasi.query.*` / `drasi.reaction.*` group by stage.

**Two further reasons, and the second is the decisive one:**

1. **Comparability.** `drasi.query.events_processed_total` should mean the same thing, and be
   chartable on the same panel, whether the query ran inside Drasi Server or a user's own binary.
   Encoding the host in the name makes that impossible and doubles the number of names for no gain.
2. **"lib" is only ever a crate.** `drasi.server.*` is legitimate because *server* is a **functional
   domain** as well as a crate — API traffic, config persistence and uptime are things only a server
   has, and they have no pipeline equivalent. There is no corresponding domain called "lib": the
   library *is* the pipeline, and the pipeline already has proper domain namespaces. The name would
   also be meaningless to an embedder reaching Drasi through a non-Rust binding, for whom no crate
   called `drasi-lib` is a visible concept.

If lib-vs-server provenance is genuinely wanted, it belongs in a label or the OTel instrumentation
scope — not in the name. **Spans follow a different convention**, deliberately; see
[01 — Tracing](01-tracing.md#span-naming-and-namespacing).

#### 9.6 Consequences

This supersedes the `_ns` convention inventoried in §3.7. The existing `ProfilingMetadata` fields
stay in nanoseconds internally — only the *exported* metric converts, via `as_secs_f64()`. Note it
settles the **unit** only: bucket *boundaries* remain the embedder's, since they are recorder
configuration that [Requirement 1](00-observability-overview.md#requirements) puts out of
drasi-lib's reach — see [Open Issue 2](#open-issues), including why the Prometheus defaults are the
wrong shape for Drasi's intervals.

### Alternatives Considered

#### 1. Embed Metrics in ComponentLogLayer

Extend the existing `ComponentLogLayer` to also track counters and histograms internally rather than adding the `metrics` crate.

**Rejected because**: `ComponentLogLayer` is a log routing mechanism, not a metrics system. The `metrics` crate provides the standard Rust interface for counters/histograms/gauges with ecosystem support for exporters. Mixing concerns in `ComponentLogLayer` would make it harder to maintain.

See also [00 — Overview](00-observability-overview.md#requirements) — Requirement 1's facade-only rule is what rejects instrumenting against the OpenTelemetry SDK directly, and it applies to metrics as well as tracing.

#### 2. Exporter-Only ("Option A")

Install a single exporter recorder — `metrics-exporter-prometheus` or an OTLP recorder — and treat
export as the recorder's only job, with no Drasi-owned store.

**Rejected because**: values then live inside the exporter and can only be read back as rendered
text (`PrometheusHandle::render()` returns a `String`). Serving `/metrics` from the existing
management API, driving UI counters, or feeding a telemetry Source (§7) would all mean parsing our
own output. It also leaves stale-series eviction with no home, so a deleted query keeps reporting
`drasi.component.up = 1` indefinitely (§2.4).

#### 3. Internal Channel into a Drasi Registry ("Option B" as originally framed)

Metric records flow over an in-process channel into a Drasi-owned registry, mirroring how
`ComponentLogLayer` feeds `ComponentLogRegistry` for logs.

**Rejected because**: the channel is the wrong primitive. Logs need one because a log event is a
discrete record that must be delivered somewhere; a metric is a *current value* held in shared,
lock-free, read-optimised state, so there is nothing to send. `metrics_util::registry::Registry`
provides that state directly. Keeping the registry while dropping the channel is exactly what §2.2
adopts — which is why the outcome is neither A nor B.

## Supportability

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Metric recording | Unit | Install `metrics-util::debugging::DebuggingRecorder`, process events, assert counter/histogram values |
| Recorder installed after components are built | Unit | Build a pipeline, install a recorder *afterwards*, then process events; assert the metrics are empty and that the documented startup warning was emitted. This pins the ordering constraint in §2.1 so it cannot regress silently |
| Queue promotion fidelity | Unit | With `DebuggingRecorder`, drive a `PriorityQueue` to capacity; assert `drasi.queue.depth`, `drasi.queue.depth_max`, `drasi.queue.drops_total` and `drasi.queue.blocked_enqueues_total` match the existing `PriorityQueueMetrics::snapshot()` values exactly. The promoted metric and the existing struct must never disagree |
| Latency histograms from profiling stamps | Unit | Push events through a mock pipeline; assert `drasi.query.engine_duration_seconds` and `drasi.pipeline.end_to_end_duration_seconds` record non-zero samples derived from the stamped `ProfilingMetadata` timestamps |
| Monotonic clock | Unit | Step the wall clock backwards mid-run; assert no histogram records a negative-turned-huge value (§5.6) |
| No `SystemTime` subtraction | Unit / lint | Assert every interval A–G is computed from an `Instant` pair. A grep-level check that no duration is derived by subtracting two `_ns` wall-clock fields is enough to pin the §5.6 rule |
| Histogram single-drainer | Unit | Record samples, then read the published snapshot twice and from two independent consumers; assert both reads return identical data and that no consumer's read empties the bucket. Assert `clear_with()` has exactly one call site (§2.5) |
| `phase` separation | Unit | Push bootstrap and steady-state events through the same query; assert `drasi.pipeline.end_to_end_duration_seconds` produces two distinct label sets and that the bootstrap samples do not appear in the steady-state distribution (§4.1) |
| `end_to_end` label set | Unit | Assert `drasi.pipeline.end_to_end_duration_seconds` carries only `query_id` and `phase` — no `source_id`, no `reaction_id`. This pins the §5.4 cardinality decision against a well-meaning future addition |
| `ProfilingConfig` removed | Compile | Assert `ProfilingConfig` and `should_profile()` no longer exist and that stamping remains unconditional (§5.6) |
| redb `stats()` is never called on a timer | Unit / lint | Assert `WriteTransaction::stats()` has no call site in any sampling or export path. It takes the single write lock and scans every page, so a periodic caller would inject latency into the WAL write path |
| RocksDB properties need no `enable_statistics()` | Unit | Open a RocksDB index *without* calling `enable_statistics()`; assert `total-sst-files-size`, `cur-size-all-mem-tables`, `estimate-pending-compaction-bytes` and `actual-delayed-write-rate` all return values via `property_int_value` |
| No sampling | Unit | Drive N events through the pipeline; assert every counter reports exactly N. Counters must never be approximate (§5.1) |
| Inspection API unchanged | Integration | `get_query_output_metrics` / `get_reaction_metrics` / `get_lifecycle_metrics` return identical values before and after the facade is added (§3.6 says emit alongside, not replace) |
| Label cardinality | Unit | Assert `error_kind` values come from a closed enum; fail the test if a formatted string reaches a label |
| Naming convention | Unit | Render the full metric set through `metrics-exporter-prometheus` and assert every name matches `^drasi_[a-z0-9_]+$`, that every monotonic counter ends in `_total`, that no name contains `_ns`/`_ms`, and that no two metrics share a name. The exporter performs no name translation, so the emitted name is the shipped name |

## Open Issues

1. ~~**Opting out of per-component labels**~~ — **RESOLVED: no escape hatch, until someone needs one.** §5.4 sizes Phase 0 at roughly 1,700 series for a mid-sized deployment and fixes the one multiplicative metric. Scaling that example twentyfold — 50 sources, 500 queries, 100 reactions — lands near 31,000 series, which a single Prometheus absorbs without noticing; the threshold where this would genuinely bite is in the thousands of queries per process. Adding a switch that drops `query_id` / `source_id` / `reaction_id` would trade away the whole diagnostic point of the metrics ("some query is slow" instead of "`orders-join` is slow") to solve a problem no known deployment has. **Reopen if a concrete deployment hits it**, with its component counts, rather than designing the hatch speculatively.
2. ~~**Histogram bucket boundaries**~~ — **RESOLVED: the embedder owns them, and drasi-lib publishes no recommended set.** Buckets are *recorder* configuration, and [Requirement 1](00-observability-overview.md#requirements) forbids drasi-lib from installing or configuring a recorder — so this is not a preference, it is the only option consistent with facade-only. `PrometheusBuilder` supports per-metric overrides via `Matcher`, so an embedder wanting different boundaries has one builder call to make.
   > ⚠️ **The default buckets are wrong for Drasi, and this document must say so even though it recommends none.** Prometheus defaults start at **5ms**, while `drasi.query.engine_duration_seconds` measures work in the **microsecond** range. Under the defaults essentially every sample lands in the first bucket and every percentile reads "under 5ms" — true, and useless: a 10µs query and a 4ms query become indistinguishable, a 400× difference. Declining to *publish* a bucket set is not the same as declining to *warn*.
   >
   > This also makes the designated-drainer decision (§2) matter more than it first appeared: keeping raw samples means an embedder who chose badly can re-bucket later, where a quantile sketch would have baked the mistake in permanently.

3. ~~**Plugin metrics naming governance**~~ — **RESOLVED for the kinds that can be dynamic; a narrow, permanent gap remains elsewhere.** Tier 1 uses `drasi.plugin.*`, tier 2 a per-kind prefix, and tier 3 is prefixed to `drasi.plugin.<plugin_kind>.*` by `FfiMetricsRecorder` ([§8.8](#88-namespace-governance)). Since the plugin model is moving to **dynamic-only** for sources, reactions, bootstrappers, identity providers and secret stores, the bridge is on every emission path for those kinds and prefix enforcement is a real guarantee rather than a convention.
   > ⚠️ **Three plugin kinds cannot go dynamic, and this is structural rather than a roadmap item.** `components/indexes/*`, `components/state_stores/*` and `components/wals/*` declare **no `cdylib` crate-type and contain no `export_plugin!`** — they are `lib`-only and have no FFI path at all, so no bridge can sit in front of them. Their **tier 1** metrics are unaffected, because those come from drasi-lib's own decorator, which sits above the FFI distinction and is fully attributed ([§8.8](#88-namespace-governance)). What stays ungoverned is a **tier 3** metric such a plugin emits for itself.
   >
   > That is acceptable **only while those plugins are first-party** — today they are Garnet, RocksDB and redb ×2. The risk B5 was about is a *third-party* plugin squatting a Drasi name, and no third-party index, state store or WAL can exist without this being revisited. **Reopen the moment one does.**
   >
   > The span-side fix from [01 — Tracing](01-tracing.md#build-mode-identity-converges-scope-does-not) does not transfer and cannot be used here: spans nest, so a host span lends `plugin_kind` to anything emitted inside it, but a metric emission has no enclosing scope to inherit from.

4. ~~**Two stores for the same numbers**~~ — **DEFERRED, deliberately.** §3.6 keeps the existing `lib/src/metrics/` atomic structs for the synchronous inspection API while also emitting through the facade into the registry (§2). That is the Phase 0 decision and it stands: it keeps the inspection API bit-for-bit unchanged at the moment of highest churn. Collapsing onto the registry later would remove the duplication but would change the inspection API's **failure modes and timing** — registry reads behave differently from an atomic load, and histograms become subject to the drainer snapshot (§2). Not a Phase 0 question, and not worth pre-deciding: **revisit when the inspection API is next touched for its own reasons.** The drift this risks is already guarded by the queue-promotion fidelity test, which asserts the promoted metric and the existing struct never disagree.

## Appendix A — Future Metrics (P1–P3)

**Not part of the Phase 0 commitment.** This appendix records the remaining 61 candidate metrics so
the Phase 0 cut can be judged against the full picture, and so later phases have a starting point.
Names, labels and phase assignments here are indicative and not agreed.

> **Names here predate the naming convention.** Duration metrics have been renamed to `_seconds`,
> but the `_total` suffix has **not** been applied to the counters below. Each name must be
> normalised against
> [00 — Overview](00-observability-overview.md#naming-and-namespacing-conventions) at the point it
> is promoted out of the appendix — which for counters means adding `_total`.

| Category | P1 | P2 | P3 | Total |
|---|---|---|---|---|
| A.1 Pipeline latency | 6 | — | — | 6 |
| A.2 Pipeline throughput | 2 | — | — | 2 |
| A.3 Queue and backpressure | 3 | 1 | — | 4 |
| A.4 Query engine and results | 4 | 2 | 1 | 7 |
| A.5 Reaction delivery and recovery | 4 | 3 | — | 7 |
| A.6 Bootstrap | 4 | — | 1 | 5 |
| A.7 Component lifecycle | 1 | 4 | — | 5 |
| A.8 Source plugins | 3 | 5 | — | 8 |
| A.9 Reaction plugins | 4 | 2 | — | 6 |
| A.10 Storage | — | 7 | 2 | 9 |
| A.11 Tokio runtime | — | — | 2 | 2 |
| **Total** | **31** | **24** | **6** | **61** |

Broadly: **P1** completes the categories Phase 0 samples and is where the plugin SDK gains a
telemetry surface. **P2** adds the categories needing new collection mechanisms — chiefly storage
backend statistics. **P3** is directional.

### A.1 Pipeline Latency

The remaining intervals from the pipeline model in
[00 — Overview](00-observability-overview.md#pipeline-model-and-instrumentation-points). All
seconds, all derived from existing `ProfilingMetadata` stamps (which remain nanoseconds internally,
per §3.7).

| Metric | Type | Labels | Interval | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.source.dispatch_duration_seconds` | histogram | `source_id` | A | derive | P1 |
| `drasi.query.ingest_wait_seconds` | histogram | `query_id`, `source_id` | B | derive | P1 |
| `drasi.query.queue_wait_seconds` | histogram | `query_id` | C | derive | P1 |
| `drasi.query.dispatch_duration_seconds` | histogram | `query_id` | E | derive | P1 |
| `drasi.reaction.dispatch_wait_seconds` | histogram | `reaction_id`, `query_id` | F | derive | P1 |
| `drasi.reaction.enqueue_duration_seconds` | histogram | `reaction_id`, `query_id` | G (partial) | derive | P1 |

### A.2 Pipeline Throughput

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.query.events_received` | counter | `query_id`, `source_id`, `phase` | Events received by the query forwarder | new | P1 |
| `drasi.reaction.results_enqueued` | counter | `reaction_id`, `query_id` | Results enqueued to a reaction | new | P1 |

### A.3 Queue Depth and Backpressure

Completes the promotion of `PriorityQueueMetrics` (§3.2).

| Metric | Type | Labels | Existing field | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.queue.enqueued_total` | counter | `component_kind`, `component_id` | `total_enqueued` | promote | P1 |
| `drasi.queue.dequeued_total` | counter | `component_kind`, `component_id` | `total_dequeued` | promote | P1 |
| `drasi.queue.capacity` | gauge | `component_kind`, `component_id` | configured max | promote | P1 |
| `drasi.queue.utilization` | gauge | `component_kind`, `component_id` | depth ÷ capacity | new | P2 |

> `drasi.queue.depth`, `drasi.queue.depth_max`, `drasi.queue.drops_total` and
> `drasi.queue.blocked_enqueues_total` are **already in Phase 0** (§4.2) and are not repeated here.
> `drasi.queue.capacity` is what makes `utilization` derivable downstream without a second metric.

### A.4 Query Engine and Result State

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.query.live_results` | gauge | `query_id` | Live (non-deleted) results tracked | promote (§3.1) | P1 |
| `drasi.query.outbox_size` | gauge | `query_id` | Outbox ring-buffer occupancy | promote (§3.1) | P1 |
| `drasi.query.outbox_sequence` | gauge | `query_id` | Latest outbox sequence | promote (§3.1) | P1 |
| `drasi.query.transaction_duration_seconds` | histogram | `query_id` | Outer-transaction duration | promote (§3.1) | P1 |
| `drasi.query.snapshot_fetches` | counter | `query_id` | Snapshot fetches served to reactions | promote (§3.1) | P2 |
| `drasi.query.index_operation_duration_seconds` | histogram | `query_id`, `index_kind`, `operation` | Time in element / result index calls | new | P2 |
| `drasi.query.solution_cardinality` | histogram | `query_id` | Intermediate solution set size per event | new | P3 |

`outer_transaction_duration_ns_last` / `_max` become a proper histogram rather than a last-and-max
pair, which is strictly more informative and removes the compare-exchange loop.
`drasi.query.index_operation_duration_seconds` is the missing link between interval D and storage
performance: today a slow query cannot be attributed to the engine versus the index backend.

### A.5 Reaction Delivery, Checkpointing and Recovery

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.reaction.checkpoint_sequence` | gauge | `reaction_id`, `query_id` | Latest persisted checkpoint sequence | promote (§3.1) | P1 |
| `drasi.reaction.dedup_skips` | counter | `reaction_id`, `query_id` | Events skipped as already processed | promote (§3.1) | P1 |
| `drasi.reaction.sequence_gaps` | counter | `reaction_id`, `query_id` | Gaps detected in the broadcast stream | promote (§3.1) | P1 |
| `drasi.reaction.recoveries` | counter | `reaction_id`, `policy` | Recovery triggered, labelled `strict`\|`auto_reset`\|`auto_skip_gap` | promote (§3.1) | P1 |
| `drasi.reaction.snapshot_fetches` | counter | `reaction_id`, `query_id` | Full snapshots fetched | promote (§3.1) | P2 |
| `drasi.reaction.outbox_fetches` | counter | `reaction_id`, `query_id` | Outbox catch-up fetches | promote (§3.1) | P2 |
| `drasi.reaction.checkpoint_write_duration_seconds` | histogram | `reaction_id` | Time to persist a checkpoint | new | P2 |

The three `recovery_*_count` fields collapse into one counter with a `policy` label — an
illustration of the general rule that **enum-valued names become labels**.

### A.6 Bootstrap

Bootstrap is a distinct operational phase with distinct failure modes, and nothing measures it
today. Depends on the `phase` marker from §4.1 being threaded through the bootstrap path.

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.bootstrap.duration_seconds` | histogram | `source_id`, `query_id` | Wall time of a bootstrap run | new | P1 |
| `drasi.bootstrap.elements_loaded` | counter | `source_id`, `query_id`, `element_kind` | Nodes / relations loaded | new | P1 |
| `drasi.bootstrap.in_progress` | gauge | `query_id` | Bootstraps currently running | new | P1 |
| `drasi.bootstrap.failures` | counter | `source_id`, `query_id`, `error_kind` | Failed bootstrap attempts | new | P1 |
| `drasi.bootstrap.rate_elements_per_sec` | gauge | `source_id`, `query_id` | Current load rate | new | P3 |

### A.7 Component Lifecycle

`ComponentStatus` (`lib/src/channels/events.rs`) already models `Added`, `Starting`, `Running`,
`Stopping`, `Stopped`, `Removed`, `Reconfiguring`, `Error`, but no metric reflects it beyond the
Phase 0 `drasi.component.up`.

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.component.startup_rejections` | counter | `reason` | Startup rejections | promote (§3.1) | P1 |
| `drasi.component.status_transitions` | counter | `component_kind`, `component_id`, `to_status` | Transitions into each state | new | P2 |
| `drasi.component.restarts` | counter | `component_kind`, `component_id` | Restarts since process start | new | P2 |
| `drasi.component.auto_reset_completions` | counter | `reaction_id` | AutoReset full re-bootstraps | promote (§3.1) | P2 |
| `drasi.component.config_hash_mismatches` | counter | `component_kind`, `component_id` | Config hash mismatches at startup | promote (§3.1) | P2 |

The four `startup_rejection_*` fields collapse into one counter with a `reason` label, matching the
existing `StartupRejectionReason` enum.

### A.8 Source Plugin Metrics

No source plugin has any instrumentation today (§3.5). Plugin metrics are structured as three tiers,
settled in [00 — Overview](00-observability-overview.md#which-plugin-types-get-telemetry): a
universal baseline every plugin gets, a per-kind standard set, and optional author-defined metrics.
All three reach the same recorder.

1. **Auto-instrumented at the boundary** (tiers 1 and 2a) — drasi-lib wraps every registered plugin
   in a decorator at `DrasiLibBuilder`, so these come for free and are uniform across all 22
   sources.
2. **Plugin-supplied** (tiers 2b and 3) — only the plugin knows its own semantics.

> **The two `auto` rows below are superseded.** The universal baseline is emitted once as a single
> `drasi.plugin.*` family labelled by `plugin_kind` and `component_kind`, covering all eight plugin
> kinds rather than one family per component type. They are listed here per-kind only to show what
> a source operator sees. The same applies to [A.9](#a9-reaction-plugin-metrics).

| Metric | Type | Labels | Mechanism | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.source.plugin.call_duration_seconds` | histogram | `source_id`, `plugin_kind`, `operation` | auto | new | P1 |
| `drasi.source.plugin.errors` | counter | `source_id`, `plugin_kind`, `error_kind` | auto | new | P1 |
| `drasi.source.connected` | gauge | `source_id`, `plugin_kind` | plugin-supplied | new | P1 |
| `drasi.source.reconnects` | counter | `source_id`, `plugin_kind` | plugin-supplied | new | P2 |
| `drasi.source.fetch_duration_seconds` | histogram | `source_id`, `plugin_kind` | plugin-supplied — time to retrieve a batch from upstream | new | P2 |
| `drasi.source.batch_size` | histogram | `source_id`, `plugin_kind` | plugin-supplied | new | P2 |
| `drasi.source.upstream_lag_seconds` | gauge | `source_id`, `plugin_kind` | plugin-supplied — event time vs. now | new | P2 |
| `drasi.source.replication_lag` | gauge | `source_id`, `plugin_kind` | plugin-supplied — CDC/WAL position lag | new | P2 |

`drasi.source.replication_lag` already has a concrete implementation waiting: the Postgres source
maintains `read_lsn` and `flush_fence_lsn` atomics (§3.5), and their difference is exactly this
metric.

### A.9 Reaction Plugin Metrics

| Metric | Type | Labels | Mechanism | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.reaction.plugin.call_duration_seconds` | histogram | `reaction_id`, `plugin_kind`, `operation` | auto | new | P1 |
| `drasi.reaction.plugin.errors` | counter | `reaction_id`, `plugin_kind`, `error_kind` | auto | new | P1 |
| `drasi.reaction.delivery_duration_seconds` | histogram | `reaction_id`, `plugin_kind` | plugin-supplied — time in the external call | new | P1 |
| `drasi.reaction.delivery_attempts` | counter | `reaction_id`, `plugin_kind`, `outcome` | plugin-supplied — `success` \| `retry` \| `failure` | new | P1 |
| `drasi.reaction.connected` | gauge | `reaction_id`, `plugin_kind` | plugin-supplied | new | P2 |
| `drasi.reaction.batch_size` | histogram | `reaction_id`, `plugin_kind` | plugin-supplied | new | P2 |

Interval G is only partially covered by `drasi.reaction.enqueue_duration_seconds`, which measures
the enqueue rather than delivery to the external system.
`drasi.reaction.delivery_duration_seconds` closes that gap and is usually where real latency lives.

As in A.8, the two `auto` rows are subsumed by the single `drasi.plugin.*` baseline family.

> **Not covered by either table: index, state store and WAL plugins.** Those three kinds never
> cross the FFI boundary, so plugin-supplied metrics can never reach them — the builder-level
> decorator is the only way they are ever instrumented. Their interaction metrics are in
> [A.10](#a10-storage-index-state-store-and-wal).

### A.10 Storage: Index, State Store and WAL

Nothing is collected today, and the RocksDB `Statistics` API is never enabled (§3.5).

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.index.operation_duration_seconds` | histogram | `query_id`, `index_kind`, `operation` | Index read/write latency | new | P2 |
| `drasi.index.operations` | counter | `query_id`, `index_kind`, `operation` | Index operation count | new | P2 |
| `drasi.index.errors` | counter | `query_id`, `index_kind`, `error_kind` | Index errors | new | P2 |
| `drasi.index.disk_bytes` | gauge | `query_id`, `index_kind` | On-disk size, from file sizes | new | P2 |
| `drasi.index.entries` | gauge | `query_id`, `index_kind` | Entry count where cheaply available | new | P3 |
| `drasi.state_store.operation_duration_seconds` | histogram | `store_kind`, `operation` | State-store latency | new | P2 |
| `drasi.wal.append_duration_seconds` | histogram | `wal_kind` | WAL append latency | new | P2 |
| `drasi.wal.size_bytes` | gauge | `wal_kind` | WAL size | new | P2 |
| `drasi.index.rocksdb.*` | mixed | `query_id` | Engine-native stats (block cache hit ratio, compaction, level sizes, memtable) — opt-in, embedded engines only | new | P3 |

> **Survey outcome**, from [Exposing embedded-engine statistics](#exposing-embedded-engine-statistics).
> The RocksDB rows are cheaper than assumed: four of the five curated statistics come from
> `property_int_value` and need no `enable_statistics()`, so they need no opt-in flag either — only
> block cache hit ratio is blocked, on the pinned `rocksdb 0.21`. The redb side is the opposite —
> **no engine-native redb metrics are collected at all**, because `stats()` requires the single
> write lock and scans every page. `drasi.wal.size_bytes` therefore comes from the filesystem, and
> the `*_duration_seconds` rows are Drasi's own timings rather than anything redb reports.

§6.2 settles the ownership question these depend on: **interaction metrics** (the `drasi.index.*`,
`drasi.state_store.*` and `drasi.wal.*` rows above) always flow through Drasi for every backend,
because only Drasi can attribute an operation to a `query_id`. **Engine-internal** stats flow
through Drasi only for in-process backends (RocksDB, redb); for the networked Garnet backend they
are scraped directly from the server and Drasi does not proxy them. Every row above additionally
carries a `backend_instance` label so a dashboard can join Drasi's view against the backend's own
exporter.

### A.11 Tokio Runtime

Process-level metrics (memory, CPU, threads, file descriptors) are **not listed here** — they belong
to whichever component owns the process, and are designed in
[Drasi Server observability §4](../../drasi-server/tracing-logging/00-observability-integration.md#4-process-resource-metrics).
See §6.1 for why drasi-lib does not emit them.

What remains is genuinely library-level:

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.runtime.tokio_tasks_alive` | gauge | — | Live tokio tasks | new | P3 |
| `drasi.runtime.tokio_worker_busy_ratio` | gauge | `worker` | Tokio worker utilization | new | P3 |

> **Both rows require `--cfg tokio_unstable`**, which is not set anywhere in drasi-core or
> drasi-server today. They are gated on a build-configuration decision, not just effort (§6.1).

## References

- [`metrics` crate](https://crates.io/crates/metrics) — Metrics facade for Rust
- [`metrics-util`](https://docs.rs/metrics-util) — `layers::{Stack, Fanout, Filter}` and
  `registry::Registry`, the building blocks of §2
- [`metrics-exporter-prometheus`](https://docs.rs/metrics-exporter-prometheus) — the exporter whose
  passthrough name handling drives the naming convention
- [OpenTelemetry semantic conventions — naming](https://opentelemetry.io/docs/specs/semconv/general/naming/)
- [Prometheus — metric and label naming](https://prometheus.io/docs/practices/naming/)
