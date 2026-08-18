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

Two ways to resolve it, both of which must be decided before the registry serves histograms to more
than one consumer:

| Option | Consequence |
|---|---|
| **`metrics_util::storage::Summary`** — a quantile sketch with relative-error guarantees | Bounded memory, non-destructive reads, safe for any number of consumers. Raw samples are lost, so exact max and arbitrary re-aggregation are unavailable |
| **One designated drainer** that `clear_with()`s on the export interval and publishes an immutable snapshot everyone else reads | Keeps raw samples and exact quantiles; adds a snapshot-publishing step and makes every consumer's view as stale as the last drain. This is what `metrics-exporter-prometheus` does internally |

Recommendation: the designated drainer, on the §5.2 export interval. It keeps the exported and
in-process views identical by construction, which matters because a `/metrics` endpoint and a
dashboard disagreeing about p99 would be worse than either being slightly stale.


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
`source_ns`, `reactivator_start_ns`, `reactivator_end_ns` (supplied by the external source);
`source_receive_ns`, `source_send_ns`, `query_receive_ns`, `query_core_call_ns`,
`query_core_return_ns`, `query_send_ns`, `reaction_receive_ns`, `reaction_complete_ns`.

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
| `ProfilingConfig` / `should_profile()` | **Defect — decide in §5.6.** Zero callers; the configuration implies an opt-in and a sampling rate that do not exist. Either delete it or wire it up, but do not place the exported histograms behind it (§5.5). |
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

> **`phase` is not optional.** `ProfilingConfig` has an `include_bootstrap` flag, but it is dead
> code (§3.3) and it gated profiling rather than labelling it — so bootstrap latency and
> steady-state latency currently mix into the same distribution. Bootstrap is bulk load and will
> dominate the tail. Every latency and throughput metric must be separable by `phase`, and that
> requires a bootstrap marker on the event envelope.

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
| 9 | `drasi.reaction.checkpoint_lag` | gauge | `reaction_id`, `query_id` | promote (§3.1) | Is it keeping up? |
| 10 | `drasi.queue.drops_total` | counter | `component_kind`, `component_id` | promote (§3.2) | Is it losing data? |
| 11 | `drasi.errors_total` | counter | `component_kind`, `component_id`, `error_kind` | new | Is it failing? |
| 12 | `drasi.query.engine_duration_seconds` | histogram | `query_id` | derive (interval D) | How slow is it? |
| 13 | `drasi.pipeline.end_to_end_duration_seconds` | histogram | `source_id`, `query_id`, `reaction_id` | derive (A→G) | How slow is it? |

> **`drasi.queue.depth_max` is reset on read.** Each export reports the high-water mark *for that
> interval*, not for all time — an all-time maximum stops being informative after the first burst.
> The underlying `max_depth_seen` field (§3.2) is monotonic, so the exported gauge is the difference
> since the previous read, and the reset must be atomic with it. See §5.3 for why this metric is in
> Phase 0 at all.

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
- **`drasi.reaction.checkpoint_lag`** is the single best consumer-health signal Drasi already
  computes and never exports.
- **`drasi.component.up`** is deliberately trivial — it is the metric every dashboard and alert
  rule starts from.
- **The two histograms** derive from `ProfilingMetadata` stamps already written on every event
  (§3.3), so they add no hot-path timing call. Their one prerequisite is the monotonic-clock fix
  in §5.6. See §5.4 for a recommended change to the `end_to_end` label set.

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
plugin boundary ([00 — Overview](00-observability-overview.md#tier-1--the-universal-baseline)),
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
| **Enablement / profiles** | Which metrics exist at all | [00 — Overview](00-observability-overview.md) |
| **Observation cadence** | Cost of *reading* expensive gauges | §5.3 |

Sampling appears in none of them.

**Sampling also breaks the metrics that matter most.** `drasi.queue.drops_total` and
`drasi.errors_total` are
counters of rare, important events. Sampling at 1% means a single dropped event is 99% likely to be
invisible, and the reported count is an estimate of a number that must be exact. A metric whose
entire purpose is to detect silent data loss cannot itself lose data. The same argument applies to
`drasi.reaction.checkpoint_lag` and `drasi.component.up`, where a sampled reading is simply a stale
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

**DECIDED — `drasi.queue.depth_max` ships in Phase 0** as the thirteenth metric (§4.2), with
reset-on-read semantics so each export reports the high-water mark *for that interval* rather than
for all time. The value is already maintained as `max_depth_seen` (§3.2), so the metric costs
nothing to produce. The two alternatives were considered and rejected as insufficient on their own:

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
deployment of 5 sources, 20 queries and 10 reactions, the Phase 0 set produces roughly 1,200 time
series — comfortable for any backend.

With one exception. **`drasi.pipeline.end_to_end_duration_seconds` as currently specified is
multiplicative**: labelling it `source_id` × `query_id` × `reaction_id` yields 5 × 20 × 10 = 1,000
label combinations, and a histogram multiplies that by its bucket count — roughly 12,000 series
from a single metric, ten times the rest of Phase 0 combined. Worse, it grows as the product of
deployment size rather than the sum.

> **Recommendation:** label `drasi.pipeline.end_to_end_duration_seconds` with `query_id` and `phase`
> only. End-to-end latency is charted per query in practice; attribution to a specific
> source–reaction pair is a tracing question, and [01 — Tracing](01-tracing.md) already answers it
> exactly. This reduces the metric to ~240 series.

#### 5.5 What Existing Instrumentation Should Do

| Existing mechanism | Sampling disposition |
|---|---|
| `lib/src/metrics/` atomic structs (§3.1) | Raw. Already exact, already cheap, no change |
| `PriorityQueueMetrics` (§3.2) | Raw. These are the loss-detection metrics — sampling them would defeat their purpose |
| `ProfilingMetadata` stamping (§3.3) | Unconditional, as it already is. The per-event cost is clock reads, addressed in §5.6 |
| Profiler Reaction `sampling_rate` | **Stays a Profiler-only control.** It samples which events get statistically summarised in the Profiler's own sliding window. It must not become a metrics control, and the exported histograms must not be placed behind it |

#### 5.6 Clock Source for Latency Histograms

The latency histograms in §4.2 have **no gating dependency** — the stamps they derive from are
already written on every event, because `ProfilingConfig` is never consulted (§3.3). What they do
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

Recommendation: the third. Intervals A–G are all intra-process, so they should be measured with
`Instant`. `SystemTime` is still needed for `source_ns` / `reactivator_*_ns`, which arrive from
outside the process and can only be wall-clock. The existing `Option<u64>` fields stay as they are;
the change is that durations stop being computed by subtracting two wall-clock stamps.

> **Also to settle here:** whether `ProfilingConfig` should be deleted or actually wired up.
> Deleting it makes the current behaviour honest. Wiring it up would reintroduce a gating problem
> for the exported histograms, so if it is wired up it must remain a Profiler-only control (§5.5).

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

For RocksDB and redb, where Drasi is the only possible exporter:

- **Opt-in, always.** Enabling RocksDB's `Statistics` has measurable overhead, and it offers levels
  (`kExceptDetailedTimers`, `kExceptTimeForMutex`, `kAll`). The choice of level should be
  configuration, not a constant.
- **Curated by default, verbatim behind a flag.** The curated set covers what a Drasi operator acts
  on — block cache hit ratio, memtable size, SST bytes on disk, pending compaction bytes, write
  stall time. Verbatim pass-through stays available as an escape hatch for deep debugging.
- **Backend-neutral names only where the semantics genuinely match.** `drasi.index.disk_bytes` is
  safe across every on-disk backend. Anything engine-specific keeps an engine-specific name
  (`drasi.index.rocksdb.*`) rather than being forced into a shared name that means something
  slightly different per backend.
- **Read on a timer, not per-operation**, on a cadence decoupled from the export interval per §5.3.

> **OPEN — the curated list itself.** The five curated RocksDB statistics above are a proposal, not
> an agreed set, and **redb's exposed surface has not been surveyed** — it may offer very little,
> in which case redb gets `disk_bytes` and nothing more. Both need confirming before A.10 moves out
> of P2.

### 7. Future: Internal Telemetry as a Drasi Source

Not phase 1, recorded for direction. Internal Drasi metrics could be exposed through a generic
telemetry/OTel **Source**, allowing continuous queries over operational data — e.g. reacting when
load crosses a threshold or processing deteriorates. A local/flagged mode would read local
telemetry directly rather than looping back over the network. A telemetry-source proposal already
exists in the mobility demo proposal and should be cross-referenced rather than duplicated. This
depends on the collection architecture chosen in §2, and would require the throttling described in
§5 so Drasi cannot overload itself with its own metrics.

### Alternatives Considered

#### 1. Embed Metrics in ComponentLogLayer

Extend the existing `ComponentLogLayer` to also track counters and histograms internally rather than adding the `metrics` crate.

**Rejected because**: `ComponentLogLayer` is a log routing mechanism, not a metrics system. The `metrics` crate provides the standard Rust interface for counters/histograms/gauges with ecosystem support for exporters. Mixing concerns in `ComponentLogLayer` would make it harder to maintain.

See also [00 — Overview](00-observability-overview.md#alternatives-considered) for the facade-vs-OpenTelemetry-SDK decision, which applies to metrics as well as tracing.

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
| No sampling | Unit | Drive N events through the pipeline; assert every counter reports exactly N. Counters must never be approximate (§5.1) |
| Inspection API unchanged | Integration | `get_query_output_metrics` / `get_reaction_metrics` / `get_lifecycle_metrics` return identical values before and after the facade is added (§3.6 says emit alongside, not replace) |
| Label cardinality | Unit | Assert `error_kind` values come from a closed enum; fail the test if a formatted string reaches a label |
| Naming convention | Unit | Render the full metric set through `metrics-exporter-prometheus` and assert every name matches `^drasi_[a-z0-9_]+$`, that every monotonic counter ends in `_total`, that no name contains `_ns`/`_ms`, and that no two metrics share a name. The exporter performs no name translation, so the emitted name is the shipped name |

## Open Issues

1. **Opting out of per-component labels**: §5.4 sizes Phase 0 at roughly 1,200 series for a mid-sized deployment and fixes the one multiplicative metric, so cardinality is bounded in the cases we expect. What is *not* settled is the escape hatch: at very large component counts, should drasi-lib offer a configuration that drops `query_id` / `source_id` / `reaction_id` from the label set and reports only aggregates? Doing so makes the metrics cheap but useless for per-component diagnosis, so it needs a deliberate default. This interacts with the filtering and profile model in [00 — Overview](00-observability-overview.md).
2. **Histogram bucket boundaries**: the naming convention settles the *unit* — duration histograms record **seconds** as `f64` — but not the bucket boundaries themselves, which the `metrics` crate leaves to the recorder. Drasi's pipeline intervals span roughly a microsecond to a few seconds, so the default Prometheus buckets (5ms–10s) resolve almost nothing at the fast end. Should drasi-lib publish a recommended bucket set for `drasi.query.engine_duration_seconds` and `drasi.pipeline.end_to_end_duration_seconds`, or leave it entirely to the embedder? `PrometheusBuilder` supports per-metric overrides via `Matcher`, so a recommendation costs the embedder one builder call.

3. **Plugin metrics naming governance**: settled in principle — tier 1 uses `drasi.plugin.*`, tier 2 a per-kind prefix, and tier 3 is prefixed to `drasi.plugin.<plugin_kind>.*` by `FfiMetricsRecorder` ([00 — Overview](00-observability-overview.md#namespace-governance)). **What remains open is enforcement for statically linked plugins**, which bypass the bridge entirely and so reach the global recorder with whatever name they choose. Either the SDK emitter becomes the only supported emission path, or static plugins are governed by convention alone.

4. **Two stores for the same numbers**: §3.6 keeps the existing `lib/src/metrics/` atomic structs for the synchronous inspection API while also emitting through the facade into the registry (§2). That is deliberate for Phase 0 — it keeps the inspection API bit-for-bit unchanged — but it means two sources of truth that can drift. Should a later phase back `get_query_output_metrics()` and friends with the registry and delete the bespoke structs? Doing so would remove the duplication but changes the inspection API's failure modes and timing.

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
