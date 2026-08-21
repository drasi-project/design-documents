# Metrics for drasi-lib

* Project Drasi - Ruokun Niu (@ruokun-niu)
* Last edited on August 20th, 2026

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
what drasi-lib already measures (§3), then defines the metrics it should emit (§4), how each build
mode collects them (§5), and how their volume is controlled (§6).

## Design

### 1. New Dependency: `metrics` Crate

Add `metrics = "0.24"` to `lib/Cargo.toml`. This is the facade crate only — no exporter.

```toml
# lib/Cargo.toml
[dependencies]
metrics = "0.24"
# tracing, tracing-subscriber, tracing-log already present
```

**This is the only dependency drasi-lib takes.** Recorder and exporter crates belong to the
embedding application, which decides whether and where metrics are collected.

### 2. Collection Architecture

**drasi-lib emits metrics but does not choose where they go.** The `metrics` crate allows one
global recorder per process. Because the embedding application owns that process, it must choose
and install the recorder—for example, a Prometheus exporter. drasi-lib then emits into the same
recorder as the application's own metrics.

So the contract drasi-lib imposes on an embedder is one line long:

| Responsibility | Owner |
|---|---|
| Emit via `counter!` / `gauge!` / `histogram!` | drasi-lib, plugins |
| Install a recorder — *if any metrics are wanted at all* | The embedding application |

If the host application already installs a recorder, Drasi's metrics flow into it automatically. If it installs none, every call is a near-no-op and there is no
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

#### 3.2 `PriorityQueueMetrics` — tracked internally

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


Seven intervals are derived from them: `elapsed_source_to_query`, `elapsed_query_processing`,
`elapsed_query_to_reaction`, `elapsed_reaction_processing`, `elapsed_total`,
`elapsed_source_internal`, `elapsed_query_internal`, plus `elapsed_summary_ms()`.

#### 3.4 Profiler Reaction — aggregation that exists but only logs

`components/reactions/profiler/src/profiler.rs` maintains a sliding window over the five derived
intervals and computes `MetricStats { count, mean, variance, std_dev, min, max, p50, p95, p99 }`. Output is a periodic `info!()` line per interval, on a
`report_interval_secs` timer. There is no file, endpoint, or structured export. It is an opt-in
user-configured reaction, not built-in behaviour.


#### 3.5 Plan for Existing Values

| Existing value | Action |
|---|---|
| `lib/src/metrics/` three structs | **Keep and additionally emit.** They serve a different purpose (synchronous in-process inspection for the API/UI); the `metrics` facade adds export. Emit alongside rather than replacing, so the inspection API is unchanged. |
| `PriorityQueueMetrics` | **Promote.** `current_depth` → `drasi.queue.depth` gauge, `drops_due_to_capacity` → `drasi.queue.drops`, `blocked_enqueue_count` → `drasi.queue.blocked_enqueues`. |
| `ProfilingMetadata` stamps | **Reuse as the source of the latency histograms** in §4.2. Already written unconditionally, so no gating change is needed — but the clock source must be fixed first (§6.6). |
| `ProfilingConfig` / `should_profile()` | **Delete (§6.6).** Zero callers; the configuration implies an opt-in and a sampling rate that do not exist. Removing it makes the current unconditional behaviour honest, and avoids reintroducing a gate the exported histograms must not sit behind (§6.5). |
| Profiler Reaction statistics | **Keep unchanged.** Aggregation moves to the recorder for exported metrics; the Profiler stays as a standalone opt-in tool. |
| `opentelemetry = "0.20"` in `core/Cargo.toml` | **Remove.** Dead dependency, zero references. |
| Throttled queue `debug!` lines | **Keep** as logs; they become redundant once the gauges are exported, but they are cheap and aid local debugging. |

### 4. Proposed Metrics
This design plans to implement thirteen metrics, chosen to be
the smallest surface that proves the architecture end to end — facade → recorder → exporter →
dashboard.

A further 61 candidate metrics across eleven categories are recorded in
[Appendix A](#appendix-a--future-metrics-p1p3) for future work (open to discussion)

Each row below carries:

- **Type** — `counter` (monotonic), `gauge` (up/down), `histogram` (distribution). These map onto
  OTel Sum / UpDownCounter / Histogram respectively.
- **Origin** — `new` (no equivalent exists), `promote` (a value already computed today, per §3, that
  only needs to be emitted through the facade), or `derive` (computable from `ProfilingMetadata`
  stamps that already exist).

#### 4.1 Standard Labels

Defined once, applied consistently across all phases.

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

#### 4.2 The Phase 0 Metric Set

Thirteen metrics, chosen to answer five operational questions:

| # | Metric | Type | Labels | Origin | Question answered |
|---|---|---|---|---|---|
| 1 | `drasi.component.up` | gauge | `component_kind`, `component_id` | new | Is it up? |
| 2 | `drasi.source.events_dispatched` | counter | `source_id`, `phase` | new | Is it doing work? |
| 3 | `drasi.query.events_processed` | counter | `query_id`, `phase` | new | Is it doing work? |
| 4 | `drasi.query.results_emitted` | counter | `query_id`, `change_kind` | new | Is it doing work? |
| 5 | `drasi.reaction.results_processed` | counter | `reaction_id`, `query_id` | new | Is it doing work? |
| 6 | `drasi.queue.depth` | gauge | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 7 | `drasi.queue.depth_max` | gauge | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 8 | `drasi.queue.blocked_enqueues` | counter | `component_kind`, `component_id` | promote (§3.2) | Is it keeping up? |
| 9 | `drasi.reaction.checkpoint_lag_events` | gauge | `reaction_id`, `query_id` | promote (§3.1) | Is it keeping up? |
| 10 | `drasi.queue.drops` | counter | `component_kind`, `component_id` | promote (§3.2) | Is it losing data? |
| 11 | `drasi.errors` | counter | `component_kind`, `component_id`, `error_kind` | new | Is it failing? |
| 12 | `drasi.query.engine.duration` | histogram | `query_id` | derive (interval D) | How slow is it? |
| 13 | `drasi.pipeline.end_to_end.duration` | histogram | `query_id`, `phase` | derive (A→G) | How slow is it? |

Intervals D and A→G are defined in the pipeline model in
[00 — Overview](00-observability-overview.md#pipeline-model-and-instrumentation-points). Names
follow the convention in
[§9](#9-naming-and-namespacing-conventions): dot-namespaced OpenTelemetry names, with instrument
type and unit recorded as metadata rather than encoded in the name.

Notes:

- **`drasi.queue.drops` is the most important metric in this document.** Dropping events at
  capacity is silent data loss, and today it is visible only as a throttled `debug!` line (§3.2).
  Together with `depth`, `depth_max` and `blocked_enqueues` it promotes values that
  `lib/src/channels/priority_queue.rs` already maintains and that nothing currently reads.
- **`drasi.queue.depth_max` earns its place because `depth` alone cannot see a burst.** A queue that
  fills and drains between two exports is invisible to an instantaneous gauge, and that burst is
  exactly what an operator needs (§6.3). The value already exists, so the metric is free.
- **`drasi.query.results_emitted`**, labelled by `change_kind`, answers "is this query actually
  producing output, and of what shape". It has no equivalent today and is the cheapest way to
  distinguish a silent query from an idle source.
- **`drasi.reaction.checkpoint_lag_events`** is the single best consumer-health signal Drasi already
  computes and never exports.
- **`drasi.component.up`** is deliberately trivial — it is the metric every dashboard and alert
  rule starts from.
- **The two histograms** derive from `ProfilingMetadata` stamps already written on every event
  (§3.3), so they add no hot-path timing call. They carry two prerequisites: the monotonic-clock
  fix in §6.6, and — for `end_to_end` — a bootstrap marker on the event envelope to populate
  `phase` (§4.1). Neither exists today.
  > Their **bucket boundaries are the embedder's**, and the Prometheus defaults are the wrong shape
  > for Drasi's microsecond-scale intervals — see §6.4 before charting percentiles.


#### 4.3 Errors

Phase 0 ships **one rollup error counter** rather than a counter per subsystem:

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `drasi.errors` | counter | `component_kind`, `component_id`, `error_kind` | Any component error |

The rules that govern it, and every error counter that follows in later phases:

- `error_kind` is a **closed enum**, never a message or a formatted string. This is the single
  largest cardinality risk in the design and the reason Phase 0 includes an error metric at all —
  the convention needs to be established and tested before it is replicated.
- Error counters live next to the operation that failed, so an error rate can always be divided by
  the corresponding throughput counter.
- Errors are additionally emitted as `tracing` events: the metric gives the rate, the trace gives
  the instance. The two are not redundant.

Later phases replace this rollup with per-subsystem counters — `drasi.plugin.errors` for the
plugin boundary ([§8.2](#82-tier-1--the-universal-baseline)),
`drasi.index.errors` for storage, and so on — all carrying the same `error_kind` discipline.

### 5. Metrics by Build Mode

#### 5.1 Statically Linked Build

Statically linked plugins and drasi-lib share the embedding process's global `metrics::Recorder`.
Their counters, gauges, and histograms therefore use the same path:

```text
drasi-lib or plugin → metrics facade → process-global recorder
```

No plugin-specific recorder or transport is required. The embedder installs the recorder before
constructing `DrasiLib`; otherwise cached metric handles remain no-op handles. Plugin authors use
the standard `metrics` API and do not configure an exporter.

#### 5.2 Dynamically Loaded Build

A cdylib plugin has its own Rust global recorder slot and cannot access the host's recorder
directly. When loading the library, the host calls the library-level `set_metrics_recorder` setter.
The plugin SDK installs `FfiMetricsRecorder`, which converts every update into `FfiMetricEntry` and
sends it through `MetricsCallbackFn`:

```text
plugin metrics API → FfiMetricsRecorder → FfiMetricEntry → MetricsCallbackFn
  → host metrics facade → process-global recorder
```

The host re-emits the same counter, gauge, or histogram update. The FFI record preserves the metric
name, instrument type, value, attributes, description, and UCUM unit, so static and dynamic builds
produce the same instruments.

New callback fields are appended to `FfiPluginRegistration` and gated by `sdk_version`; older
plugins never access the appended fields.


### 6. Sampling, Aggregation and Reporting Intervals


The instinct to apply a sampling rate carries over from tracing, where it is essential. It does not
transfer, because the two signals have different cost models.

#### 6.1 Levers for Controlling Metrics Cost

| Lever | Owner and control |
|---|---|
| **Reporting interval** | Embedder/backend: Prometheus `scrape_interval` or OpenTelemetry `PeriodicReader::with_interval()` (§6.2) |
| **Label cardinality** | Fixed by the metric definitions; not runtime-configurable (§4.1, §6.4) |
| **Enablement / profiles** | Embedder installs the resolved metric filter; `DrasiLibBuilder` receives the resolved `CollectionFlags` |
| **Observation cadence** | `CollectionFlags` controls expensive drasi-lib gauges; backend-specific statistics use the provider's own configuration (§6.3) |

Profiles select which metrics are emitted; they do not change a sampling rate because metrics are
not sampled. Profile definitions remain in
[00 — Overview](00-observability-overview.md#enablement-filtering-and-observability-profiles).
`TelemetryProfile` ships in Phase 1; Phase 0 embedders apply the equivalent recorder filter and
collection flags directly.

The profile is resolved once and its two metrics outputs are applied on opposite sides of the
library boundary:

```rust
let profile = TelemetryProfile::Debug.resolve();

// Embedder-owned: install the recorder and suppress filtered metric names.
metrics_util::layers::Stack::new(
  metrics_exporter_prometheus::PrometheusBuilder::new().build_recorder(),
)
.push(metrics_util::layers::FilterLayer::from_patterns(
  &profile.metric_filters,
))
.install()?;

// drasi-lib-owned collection: enable only the work selected by the profile.
let drasi = DrasiLib::builder()
  .with_telemetry(profile.collection)
  .build()
  .await?;
```

The recorder must be installed first because components cache metric handles during construction.
Prometheus scrape cadence is configured in Prometheus, not on `DrasiLibBuilder`.

**Sampling also breaks the metrics that matter most.** `drasi.queue.drops` and
`drasi.errors` are
counters of rare, important events. Sampling at 1% means a single dropped event is 99% likely to be
invisible, and the reported count is an estimate of a number that must be exact. A metric whose
entire purpose is to detect silent data loss cannot itself lose data. The same argument applies to
`drasi.reaction.checkpoint_lag_events` and `drasi.component.up`, where a sampled reading is simply a stale
reading.

**Counters and histograms are already aggregations.** Incrementing a counter is one relaxed atomic
add. Recording a histogram sample is a bucket index plus an atomic add. Neither allocates and
neither grows with cardinality of the data. There is no accumulating cost for sampling to relieve.

#### 6.2 Reporting Interval — the Real Volume Control

drasi-lib has no reporting-interval setting because it does not own an exporter:

| Backend | Where cadence is configured |
|---|---|
| Prometheus | The scraper's `scrape_interval` |
| OTLP | The embedder's `PeriodicReader::with_interval()` |
| Other recorder | The recorder/exporter configuration defined by that backend |

For example, an OTLP embedder can configure periodic export independently of drasi-lib:

```rust
let reader = PeriodicReader::builder(exporter)
  .with_interval(Duration::from_secs(30))
  .build();
```

Shorter intervals increase downstream volume but do not change how often counters and histograms
are updated in-process. They are an exporter concern, not a `DrasiLibBuilder` option.

#### 6.3 Gauge Observation Cadence

Gauges are the one place where something *resembling* sampling is unavoidable: a gauge is read at
export time, so its value between exports is unobserved. That is an observation-cadence question,
not a statistical-sampling one, and it has a real consequence for Phase 0:

> **`drasi.queue.depth` read every 10s will miss short saturation bursts.** A queue that fills and
> drains within 200ms is invisible at a 10s cadence, yet it is exactly the event an operator needs
> to see. The instantaneous gauge alone is not sufficient evidence of backpressure.

The two alternatives were considered and rejected as insufficient on their own:

- `drasi.queue.blocked_enqueues` and `drasi.queue.drops` are counters and therefore lose
  nothing — but they only fire once the queue is *already* full, so they detect saturation rather
  than approach to it. They remain in Phase 0, but as a last line rather than an early warning.
- Shortening the export interval is expensive, applies globally, and still only narrows the window
  rather than closing it.

Phase 0 gauges are maintained as events occur and need no polling configuration. Future gauges that
are expensive to read — RocksDB properties and on-disk index sizes — use a separate cadence in
`CollectionFlags`, defaulting to 60 seconds. Backend statistics that must be enabled before the
provider is constructed remain part of that provider's configuration, not `DrasiLibBuilder`.

#### 6.4 Cardinality and Histogram Buckets

Metric definitions fix their label sets; runtime configuration cannot add or remove identity
labels. In particular, `drasi.pipeline.end_to_end.duration` carries only `query_id` and `phase`.
Adding `source_id` and `reaction_id` would make its series count grow multiplicatively; exact
source-to-reaction attribution belongs in tracing.

Histogram bucket boundaries are recorder configuration and therefore belong to the embedder.
drasi-lib does not publish or install a bucket set. Embedders must configure suitable boundaries
for Drasi latency histograms: Prometheus defaults begin at 5 ms, which is too coarse to distinguish
the microsecond-scale query durations Drasi commonly records.

#### 6.5 Existing Instrumentation Remains Unsampled

The existing atomic metric structs and priority-queue counters remain exact and are emitted without
sampling. `ProfilingMetadata` stamps also remain unconditional so exported histograms observe every
event. The Profiler Reaction may continue sampling its own sliding-window report, but that setting
does not gate exported metrics.

`ProfilingConfig` and `should_profile()` are removed because they have no callers and imply a
collection gate that does not exist.

#### 6.6 Clock Source for Latency Histograms

Durations must use a monotonic clock. The current nanosecond stamps come from `SystemTime`, which
can move backwards after an NTP or manual clock adjustment and corrupt an unsigned histogram
duration.

Each interval A–G is therefore measured with an `Instant` pair in one process and clock domain.
`SystemTime` remains only for absolute timestamps that must correlate with an external system.
`Instant` values never cross FFI; plugin-internal durations are measured inside the plugin and
reported as completed observations. The enforceable rule is: never subtract two `SystemTime`
stamps to produce a duration.


### 7. Environment and Storage Metrics

#### 7.1 Process and Runtime Metrics — Why Not in drasi-lib

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

#### 7.2 Storage Backend Metrics

Users receive RocksDB metrics through Drasi's normal metrics recorder.

RocksDB runs inside the Drasi process, so drasi-lib periodically reads a curated set of cheap
properties and emits them through the `metrics` facade. They appear in the same Prometheus or OTLP
output as the pipeline metrics; users do not install a separate RocksDB exporter.

When engine-specific storage metrics are added in P3, the initial set is:

| Metric | Type | Unit | RocksDB property | Meaning |
|---|---|---|---|---|
| `drasi.index.rocksdb.sst.size` | gauge | `By` | `total-sst-files-size` | Total size of all SST files |
| `drasi.index.rocksdb.memtable.size` | gauge | `By` | `cur-size-all-mem-tables` | Current size of all memtables |
| `drasi.index.rocksdb.pending_compaction.size` | gauge | `By` | `estimate-pending-compaction-bytes` | Estimated bytes awaiting compaction |
| `drasi.index.rocksdb.delayed_write.rate` | gauge | `By/s` | `actual-delayed-write-rate` | Current write rate while RocksDB is delaying writes |
| `drasi.index.rocksdb.background_errors` | counter | `1` | `background-errors` | Background errors since the database opened |
| `drasi.index.rocksdb.compactions.running` | gauge | `1` | `num-running-compactions` | Compactions currently running |
| `drasi.index.rocksdb.keys.estimated` | gauge | `1` | `estimate-num-keys` | Estimated number of keys |

drasi-lib reads these values with `property_int_value()`; they do not require RocksDB
`Statistics`. For `background_errors`, Drasi emits the increase since the previous read so the
instrument remains a monotonic counter. Block-cache hit ratio is deferred until Drasi upgrades
from `rocksdb 0.21` to a version with typed ticker access.


### 8. Plugin Metrics

Plugin metrics are Phase 1 work; they are not part of the thirteen-metric Phase 0 set in §4.2.

Telemetry is a standard capability of every plugin kind, structured as three tiers: a
universal baseline every plugin gets, a per-kind standard set, and whatever the plugin author adds
on top. All three land in the same recorder.

The FFI transport these tiers ride on is described in
[00 — Overview](00-observability-overview.md#plugin-telemetry-across-ffi); this section defines what
is actually emitted.

#### 8.1 The Three Tiers

Tiers 1 and 2 are **guaranteed** — they exist for a plugin whose author wrote no instrumentation at
all. Tier 3 is the author's own, and is optional.

| Tier | What | Who emits it | Applies to |
|---|---|---|---|
| **1 — Universal baseline** | Same handful of metrics for every plugin, whatever its kind | drasi-lib, automatically | all 8 kinds |
| **2 — Per-kind standard set** | Metrics meaningful for *that* kind: source metrics, reaction metrics, bootstrapper metrics, … | drasi-lib, automatically | all 8 kinds |
| **3 — Plugin-author metrics** | Whatever the author considers useful about their own internals | the plugin, opt-in | any kind |

Plugin developers do not implement a telemetry interface, create standard metric handles, add
labels, or configure a recorder for Tiers 1 and 2. Registering the plugin with drasi-lib is enough.
Only Tier 3 requires plugin code, and Tier 3 is optional.

All three tiers reach the same process-global recorder through the static or dynamic path defined
in [§5.1–§5.2](#51-statically-linked-build).

#### 8.2 Tier 1 — the universal baseline

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
| `drasi.plugin.call.duration` | histogram (`s`) | `plugin_kind`, `component_kind`, `component_id`, `operation` |
| `drasi.plugin.errors` | counter | `plugin_kind`, `component_kind`, `component_id`, `operation`, `error_kind` |

`operation` is the trait method — `start`, `stop`, `subscribe`, `enqueue_query_result`,
`get_credentials`, `get_secret`, `bootstrap`, `get`, `set`, `delete`, `append`, `read_from`. Four
metrics describe every plugin in the process, so one dashboard panel and one alert rule cover all
of them regardless of kind.

#### 8.3 Tier 2 — the per-kind standard set

Tier 1 is deliberately semantic-free: it knows a call happened, not what it meant. Tier 2 adds the
metrics that are meaningful for a specific kind, so that every source reports the same things as
every other source and a Postgres source can be compared against a Kafka one.

Tier 2 contains only values drasi-lib can derive from the calls and data already visible at the
wrapper or runtime boundary. drasi-lib records them automatically; the plugin API and plugin
implementation do not change.

| Kind | Tier 2 metrics |
|---|---|
| Source | `drasi.source.subscriptions`, `drasi.source.active_subscriptions` |
| Reaction | `drasi.reaction.results_processed`, `drasi.reaction.bootstraps` |
| Bootstrap provider | `drasi.bootstrap.runs`, `drasi.bootstrap.duration`, `drasi.bootstrap.elements_streamed` |
| Identity provider | `drasi.identity.credential_requests`, `drasi.identity.request.duration` |
| Secret store | `drasi.secret_store.requests`, `drasi.secret_store.request.duration` |
| Index backend | `drasi.index.operations`, `drasi.index.operation.duration` |
| State store | `drasi.state_store.operations`, `drasi.state_store.operation.duration` |
| WAL provider | `drasi.wal.appends`, `drasi.wal.append.duration` |

The last three rows are exactly the **interaction metrics** that
[§7.2](#72-storage-backend-metrics) assigns to Drasi, and
[A.10](#a10-storage-index-state-store-and-wal) catalogues. They are not a separate mechanism —
storage backends are plugins, and the decorator is where their interaction metrics come from.

Values that drasi-lib cannot observe, such as upstream connection state, replication lag, or an
internally handled retry, are not Tier 2 and are not part of the guaranteed standard set. A plugin
author may expose them as optional Tier 3 metrics, but no plugin is required to do so.

#### 8.4 Tier 3 — plugin-author metrics

Anything else the author wants to measure about their own internals — connection state,
replication lag, WAL parse time, change-feed decoding, retry loops, or cache hits. The author uses
the standard `metrics` macros and the SDK routes them to the same recorder as tiers 1 and 2.

Register handles once during initialization, cache them on the plugin, and record through those
handles on the hot path:

```rust
use metrics::{Counter, Histogram};

struct PluginMetrics {
  frames_received: Counter,
  decode_duration: Histogram,
}

let emitter = context.metrics();
let metrics = PluginMetrics {
  frames_received: emitter.counter("frames_received"),
  decode_duration: emitter.histogram("decode.duration"),
};

metrics.frames_received.increment(1);
metrics.decode_duration.record(started.elapsed().as_secs_f64());
```

The SDK adds `plugin_kind` and `component_id` and places these under the plugin's
`drasi.plugin.<plugin_kind>.*` namespace.

#### 8.5 Tier 3 Namespace

Plugin authors provide only a low-cardinality leaf name such as `cache_hits` or
`decode.duration`. The SDK publishes it as
`drasi.plugin.<plugin_kind>.<metric>` and adds `plugin_kind` and, when available, `component_id`.
Authors must not emit Tier 3 metrics under reserved first-party namespaces such as `drasi.source.*`.

#### 8.6 Worked example: adding metrics to a source plugin

Suppose someone is writing a source that subscribes to a remote change feed and reacts to frames as
the upstream system pushes them. Before they write a single line of instrumentation they already
get Tiers 1 and 2, because drasi-lib emits those from the decorator wrapping their plugin.
What follows is only what they add on top.

**Step 1 — declare the optional Tier 3 handles.** Handles are registered once and cached on the
struct, never created per event. This matters: the `metrics` macros build the metric `Key`
*before* consulting the recorder, so a per-event `counter!("…", "source_id" => id)` allocates on
every event even when nothing is collecting.

```rust
use drasi_plugin_sdk::prelude::*;
use metrics::{Counter, Histogram};

struct ChangeFeedMetrics {
    frames_received: Counter,
    decode_duration: Histogram,
}
```

**Step 2 — obtain them from the runtime context.** `context.metrics()` returns an emitter that has
already captured `plugin_kind` and `component_id`, so the author never writes a label. This is also
what keeps tier 3 names inside the plugin's own namespace.

```rust
async fn initialize(&self, context: SourceRuntimeContext) {
    let m = context.metrics();
    let _ = self.metrics.set(ChangeFeedMetrics {
    frames_received: m.counter("frames_received"), // drasi.plugin.changefeed.frames_received
    decode_duration: m.histogram("decode.duration"),
    });
}
```

**Step 3 — record the optional metrics.** Tiers 1 and 2 continue to be emitted by drasi-lib around
these calls; none of the code below is required for standard telemetry.

```rust
async fn start(&self) -> Result<()> {
    let m = self.metrics.get().expect("initialize runs first");
    let mut stream = self.subscribe_upstream().await?;

    while let Some(frame) = stream.next().await {
    let frame = frame?;

    m.frames_received.increment(1);

        let started = Instant::now();
        let changes = self.decode(frame)?;
    m.decode_duration.record(started.elapsed().as_secs_f64());

        for change in changes {
            self.dispatch_change(change).await?;
        }
    }
    Ok(())
}
```

Three things worth noting about what the author did *not* write:

- **No Tier 1 or Tier 2 instrumentation.** drasi-lib records the standard metrics outside the
  plugin implementation.
- **No labels.** `source_id` / `plugin_kind` are attached by the emitter, so they cannot be
  forgotten, misspelled, or made unbounded.
- **No exporter, no recorder, no configuration.** Whether these land in Prometheus or OTLP, and
  whether this plugin is statically linked or loaded from a `.so`, is decided by the host.

**For plugin kinds with no runtime context** — bootstrap, identity, secret store — there is no
`context.metrics()`, so the SDK exposes a library-scoped emitter instead. It carries `plugin_kind`
but not `component_id`, per §8.6:

```rust
let m = drasi_plugin_sdk::metrics::plugin_metrics();  // no component_id available
let cache_hits = m.counter("cache_hits");             // drasi.plugin.vault.cache_hits
```

**The raw macros still work.** `metrics::counter!("anything")` reaches the same recorder — the
emitter is a convenience and a governance mechanism, not a gate.

### 9. Naming and Namespacing Conventions

Drasi metric names follow OpenTelemetry conventions so the emitted API is independent of any
specific exporter:

1. Use lowercase dot-separated namespaces under `drasi`, with `snake_case` inside each segment:
  `drasi.query.engine.duration`.
2. Keep units out of names. Declare them as instrument metadata using UCUM units such as `s`,
  `By`, or `1`.
3. Do not add `_total` to counters. The instrument type already identifies a monotonic sum, and an
  exporter may translate the name for its backend.
4. Put dimensions such as `query_id`, `operation`, and `error_kind` in attributes, never in names.
5. Use domain namespaces such as `drasi.source.*`, `drasi.query.*`, and `drasi.plugin.*`; service
  identity belongs in the OpenTelemetry resource, not the metric name.

#### 9.1 Namespace Layout

| Namespace | Owner |
|---|---|
| `drasi.source.*`, `drasi.query.*`, `drasi.reaction.*` | pipeline stages |
| `drasi.pipeline.*` | metrics spanning the whole pipeline |
| `drasi.queue.*` | backpressure, labelled by `component_kind` |
| `drasi.component.*` | lifecycle, any component type |
| `drasi.plugin.*` | the universal plugin baseline (tier 1, §8.2) |
| `drasi.plugin.<plugin_kind>.*` | plugin-author metrics (tier 3), prefix enforced by the bridge |
| `drasi.index.*`, `drasi.state_store.*`, `drasi.wal.*` | storage interaction |
| `drasi.index.rocksdb.*` | engine properties, kept engine-specific by design |
| `process_*` | **exception** — ecosystem-standard, no `drasi` prefix |

Engine-specific statistics keep an engine-specific namespace rather than being forced into a shared
name, following OTel's own reasoning for preferring `jvm.gc.*` over `gc.*`: implementations differ
enough that a shared name invites false comparison.

#### 9.2 Why Not `drasi.lib.*`

`lib` is the name of the Rust crate, not the domain being measured. A `drasi.lib.*` prefix would
tie metric names to one implementation and make the same query or source use different names when
hosted by Drasi Server or another embedder. Metrics therefore use domain prefixes such as
`drasi.query.*`, `drasi.source.*`, and `drasi.pipeline.*`. The host application is identified by
the OpenTelemetry resource, such as `service.name`, rather than by the metric name.

#### 9.3 Consequences

The existing `ProfilingMetadata` wall-clock fields stay in nanoseconds internally; exported
durations are recorded as `f64` seconds. This settles the **unit** only: bucket boundaries remain
embedder-owned recorder configuration, as described in §6.4.

### Alternatives Considered

#### Embed Metrics in ComponentLogLayer

Extend the existing `ComponentLogLayer` to also track counters and histograms internally rather than adding the `metrics` crate.

**Rejected because**: `ComponentLogLayer` is a log routing mechanism, not a metrics system. The `metrics` crate provides the standard Rust interface for counters/histograms/gauges with ecosystem support for exporters. Mixing concerns in `ComponentLogLayer` would make it harder to maintain.

See also [00 — Overview](00-observability-overview.md#requirements) — Requirement 1's facade-only rule is what rejects instrumenting against the OpenTelemetry SDK directly, and it applies to metrics as well as tracing.

## Supportability

### Verification

| Test | Scope | Approach |
|------|-------|----------|
| Metric recording | Unit | Install `metrics-util::debugging::DebuggingRecorder`, process events, assert counter/histogram values |
| Recorder installed after components are built | Unit | Build a pipeline, install a recorder *afterwards*, then process events; assert the metrics are empty. This pins the ordering constraint in §2 so it cannot regress silently |
| Queue promotion fidelity | Unit | With `DebuggingRecorder`, drive a `PriorityQueue` to capacity; assert `drasi.queue.depth`, `drasi.queue.depth_max`, `drasi.queue.drops` and `drasi.queue.blocked_enqueues` match the existing `PriorityQueueMetrics::snapshot()` values exactly. The promoted metric and the existing struct must never disagree |
| Latency histograms from profiling stamps | Unit | Push events through a mock pipeline; assert `drasi.query.engine.duration` and `drasi.pipeline.end_to_end.duration` record non-zero samples derived from the stamped `ProfilingMetadata` timestamps |
| Monotonic clock | Unit | Step the wall clock backwards mid-run; assert no histogram records a negative-turned-huge value (§6.6) |
| No `SystemTime` subtraction | Unit / lint | Assert every interval A–G is computed from an `Instant` pair. A grep-level check that no duration is derived by subtracting two `_ns` wall-clock fields is enough to pin the §6.6 rule |
| `phase` separation | Unit | Push bootstrap and steady-state events through the same query; assert `drasi.pipeline.end_to_end.duration` produces two distinct attribute sets and that the bootstrap samples do not appear in the steady-state distribution (§4.1) |
| `end_to_end` attribute set | Unit | Assert `drasi.pipeline.end_to_end.duration` carries only `query_id` and `phase` — no `source_id`, no `reaction_id`. This pins the §6.4 cardinality decision against a well-meaning future addition |
| `ProfilingConfig` removed | Compile | Assert `ProfilingConfig` and `should_profile()` no longer exist and that stamping remains unconditional (§6.6) |
| redb `stats()` is never called on a timer | Unit / lint | Assert `WriteTransaction::stats()` has no call site in any sampling or export path. It takes the single write lock and scans every page, so a periodic caller would inject latency into the WAL write path |
| RocksDB properties need no `enable_statistics()` | Unit | Open a RocksDB index *without* calling `enable_statistics()`; assert `total-sst-files-size`, `cur-size-all-mem-tables`, `estimate-pending-compaction-bytes` and `actual-delayed-write-rate` all return values via `property_int_value` |
| No sampling | Unit | Drive N events through the pipeline; assert every counter reports exactly N. Counters must never be approximate (§6.1) |
| Inspection API unchanged | Integration | `get_query_output_metrics` / `get_reaction_metrics` / `get_lifecycle_metrics` return identical values before and after the facade is added (§3.5 says emit alongside, not replace) |
| Label cardinality | Unit | Assert `error_kind` values come from a closed enum; fail the test if a formatted string reaches a label |
| Naming convention | Unit | Inspect every registered instrument and assert names are lowercase and dot-separated under `drasi.`, contain no unit or `_total` suffix, use UCUM unit metadata, and do not collide |



## Appendix A — Future Metrics (P1–P3)

**Not part of the Phase 0 commitment.** This appendix records the remaining 61 candidate metrics so
the Phase 0 cut can be judged against the full picture, and so later phases have a starting point.
Names, labels and phase assignments here are indicative and not agreed.

> Names follow [§9](#9-naming-and-namespacing-conventions). Units are instrument metadata and are
> not encoded in the metric name.

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
seconds, all derived from existing `ProfilingMetadata` stamps (§3.3).

| Metric | Type | Labels | Interval | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.source.dispatch.duration` | histogram (`s`) | `source_id` | A | derive | P1 |
| `drasi.query.ingest_wait.duration` | histogram (`s`) | `query_id`, `source_id` | B | derive | P1 |
| `drasi.query.queue_wait.duration` | histogram (`s`) | `query_id` | C | derive | P1 |
| `drasi.query.dispatch.duration` | histogram (`s`) | `query_id` | E | derive | P1 |
| `drasi.reaction.dispatch_wait.duration` | histogram (`s`) | `reaction_id`, `query_id` | F | derive | P1 |
| `drasi.reaction.enqueue.duration` | histogram (`s`) | `reaction_id`, `query_id` | G (partial) | derive | P1 |

### A.2 Pipeline Throughput

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.query.events_received` | counter | `query_id`, `source_id`, `phase` | Events received by the query forwarder | new | P1 |
| `drasi.reaction.results_enqueued` | counter | `reaction_id`, `query_id` | Results enqueued to a reaction | new | P1 |

### A.3 Queue Depth and Backpressure

Completes the promotion of `PriorityQueueMetrics` (§3.2).

| Metric | Type | Labels | Existing field | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.queue.enqueued` | counter | `component_kind`, `component_id` | `total_enqueued` | promote | P1 |
| `drasi.queue.dequeued` | counter | `component_kind`, `component_id` | `total_dequeued` | promote | P1 |
| `drasi.queue.capacity` | gauge | `component_kind`, `component_id` | configured max | promote | P1 |
| `drasi.queue.utilization` | gauge | `component_kind`, `component_id` | depth ÷ capacity | new | P2 |

> `drasi.queue.depth`, `drasi.queue.depth_max`, `drasi.queue.drops` and
> `drasi.queue.blocked_enqueues` are **already in Phase 0** (§4.2) and are not repeated here.
> `drasi.queue.capacity` is what makes `utilization` derivable downstream without a second metric.

### A.4 Query Engine and Result State

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.query.live_results` | gauge | `query_id` | Live (non-deleted) results tracked | promote (§3.1) | P1 |
| `drasi.query.outbox_size` | gauge | `query_id` | Outbox ring-buffer occupancy | promote (§3.1) | P1 |
| `drasi.query.outbox_sequence` | gauge | `query_id` | Latest outbox sequence | promote (§3.1) | P1 |
| `drasi.query.transaction.duration` | histogram (`s`) | `query_id` | Outer-transaction duration | promote (§3.1) | P1 |
| `drasi.query.snapshot_fetches` | counter | `query_id` | Snapshot fetches served to reactions | promote (§3.1) | P2 |
| `drasi.query.index_operation.duration` | histogram (`s`) | `query_id`, `index_kind`, `operation` | Time in element / result index calls | new | P2 |
| `drasi.query.solution_cardinality` | histogram | `query_id` | Intermediate solution set size per event | new | P3 |

`outer_transaction_duration_ns_last` / `_max` become a proper histogram rather than a last-and-max
pair, which is strictly more informative and removes the compare-exchange loop.
`drasi.query.index_operation.duration` is the missing link between interval D and storage
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
| `drasi.reaction.checkpoint_write.duration` | histogram (`s`) | `reaction_id` | Time to persist a checkpoint | new | P2 |

The three `recovery_*_count` fields collapse into one counter with a `policy` label — an
illustration of the general rule that **enum-valued names become labels**.

### A.6 Bootstrap

Bootstrap is a distinct operational phase with distinct failure modes, and nothing measures it
today. Phase 0 introduces the `phase` marker for the end-to-end histogram; these P1 metrics reuse
that marker once they are added.

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.bootstrap.duration` | histogram (`s`) | `source_id`, `query_id` | Wall time of a bootstrap run | new | P1 |
| `drasi.bootstrap.elements_loaded` | counter | `source_id`, `query_id`, `element_kind` | Nodes / relations loaded | new | P1 |
| `drasi.bootstrap.in_progress` | gauge | `query_id` | Bootstraps currently running | new | P1 |
| `drasi.bootstrap.failures` | counter | `source_id`, `query_id`, `error_kind` | Failed bootstrap attempts | new | P1 |
| `drasi.bootstrap.elements.rate` | gauge (`{element}/s`) | `source_id`, `query_id` | Current load rate | new | P3 |

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

1. **Auto-instrumented at the boundary** (Tiers 1 and 2) — drasi-lib wraps every registered plugin
   in a decorator at `DrasiLibBuilder`, so these come for free and are uniform across all 22
   sources.
2. **Optional plugin internals** (Tier 3) — authors may add these when useful, but they are not
  required for standard telemetry.

> **The two `auto` rows below are superseded.** The universal baseline is emitted once as a single
> `drasi.plugin.*` family labelled by `plugin_kind` and `component_kind`, covering all eight plugin
> kinds rather than one family per component type. They are listed here per-kind only to show what
> a source operator sees. The same applies to [A.9](#a9-reaction-plugin-metrics).

| Metric | Type | Labels | Mechanism | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.plugin.call.duration` | histogram (`s`) | `component_id`, `plugin_kind`, `operation` | Tier 1, automatic | new | P1 |
| `drasi.plugin.errors` | counter | `component_id`, `plugin_kind`, `operation`, `error_kind` | Tier 1, automatic | new | P1 |
| `drasi.plugin.<plugin_kind>.connected` | gauge | `component_id` | Tier 3, optional | new | P1 |
| `drasi.plugin.<plugin_kind>.reconnects` | counter | `component_id` | Tier 3, optional | new | P2 |
| `drasi.plugin.<plugin_kind>.fetch.duration` | histogram (`s`) | `component_id` | Tier 3, optional — time to retrieve a batch from upstream | new | P2 |
| `drasi.plugin.<plugin_kind>.batch_size` | histogram | `component_id` | Tier 3, optional | new | P2 |
| `drasi.plugin.<plugin_kind>.upstream_lag` | gauge (`s`) | `component_id` | Tier 3, optional — event time vs. now | new | P2 |
| `drasi.plugin.<plugin_kind>.replication_lag` | gauge | `component_id` | Tier 3, optional — CDC/WAL position lag | new | P2 |

The Postgres source can optionally expose `drasi.plugin.postgres.replication_lag`: it already
maintains `read_lsn` and `flush_fence_lsn` atomics (§3.5), and their difference is exactly this
value.

### A.9 Reaction Plugin Metrics

| Metric | Type | Labels | Mechanism | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.plugin.call.duration` | histogram (`s`) | `component_id`, `plugin_kind`, `operation` | Tier 1, automatic | new | P1 |
| `drasi.plugin.errors` | counter | `component_id`, `plugin_kind`, `operation`, `error_kind` | Tier 1, automatic | new | P1 |
| `drasi.plugin.<plugin_kind>.delivery.duration` | histogram (`s`) | `component_id` | Tier 3, optional — time in the external call | new | P1 |
| `drasi.plugin.<plugin_kind>.delivery_attempts` | counter | `component_id`, `outcome` | Tier 3, optional — `success` \| `retry` \| `failure` | new | P1 |
| `drasi.plugin.<plugin_kind>.connected` | gauge | `component_id` | Tier 3, optional | new | P2 |
| `drasi.plugin.<plugin_kind>.batch_size` | histogram | `component_id` | Tier 3, optional | new | P2 |

Interval G is only partially covered by `drasi.reaction.enqueue.duration`, which measures
the enqueue rather than delivery to the external system.
The optional plugin delivery-duration metric closes that gap and is usually where real latency
lives.

As in A.8, the two `auto` rows are subsumed by the single `drasi.plugin.*` baseline family.

> **Not covered by either table: index, state store and WAL plugins.** Their standard interaction
> metrics are emitted automatically by the builder-level decorator, without plugin instrumentation.
> Those metrics are listed in
> [A.10](#a10-storage-index-state-store-and-wal).

### A.10 Storage: Index, State Store and WAL

Nothing is collected today, and the RocksDB `Statistics` API is never enabled (§3.5).

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.index.operation.duration` | histogram (`s`) | `query_id`, `index_kind`, `operation` | Index read/write latency | new | P2 |
| `drasi.index.operations` | counter | `query_id`, `index_kind`, `operation` | Index operation count | new | P2 |
| `drasi.index.errors` | counter | `query_id`, `index_kind`, `error_kind` | Index errors | new | P2 |
| `drasi.index.disk.size` | gauge (`By`) | `query_id`, `index_kind` | On-disk size, from file sizes | new | P2 |
| `drasi.index.entries` | gauge | `query_id`, `index_kind` | Entry count where cheaply available | new | P3 |
| `drasi.state_store.operation.duration` | histogram (`s`) | `store_kind`, `operation` | State-store latency | new | P2 |
| `drasi.wal.append.duration` | histogram (`s`) | `wal_kind` | WAL append latency | new | P2 |
| `drasi.wal.size` | gauge (`By`) | `wal_kind` | WAL size | new | P2 |
| `drasi.index.rocksdb.*` | mixed | `query_id` | Curated engine properties for compaction, level sizes, memtables, and estimated keys; block-cache hit ratio deferred | new | P3 |

> **Survey outcome**, from [§7.2](#72-storage-backend-metrics).
> The seven curated RocksDB properties in §7.2 come from `property_int_value` and need no
> `enable_statistics()`, so they need no opt-in flag either — only
> block cache hit ratio is blocked, on the pinned `rocksdb 0.21`. The redb side is the opposite —
> **no engine-native redb metrics are collected at all**, because `stats()` requires the single
> write lock and scans every page. `drasi.wal.size` therefore comes from the filesystem, and the
> duration rows are Drasi's own timings rather than anything redb reports.

§7.2 settles the ownership question these depend on: **interaction metrics** (the `drasi.index.*`,
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
See §7.1 for why drasi-lib does not emit them.

What remains is genuinely library-level:

| Metric | Type | Labels | Meaning | Origin | Phase |
|---|---|---|---|---|---|
| `drasi.runtime.tokio_tasks_alive` | gauge | — | Live tokio tasks | new | P3 |
| `drasi.runtime.tokio_worker.busy` | gauge (`1`) | `worker` | Tokio worker utilization | new | P3 |

> **Both rows require `--cfg tokio_unstable`**, which is not set anywhere in drasi-core or
> drasi-server today. They are gated on a build-configuration decision, not just effort (§7.1).

## Appendix B — Internal Telemetry as a Drasi Source

Internal Drasi metrics could be exposed through a telemetry/OTel **Source**, allowing continuous
queries over operational data. A local mode would read telemetry directly rather than loop back
over the network. This requires a separate local read-path design and throttling so Drasi cannot
overload itself with its own metrics. The existing mobility demo proposal should be referenced
rather than duplicated.

## References

- [`metrics` crate](https://crates.io/crates/metrics) — Metrics facade for Rust
- [`metrics-util`](https://docs.rs/metrics-util) — `DebuggingRecorder` used by verification tests
- [`metrics-exporter-prometheus`](https://docs.rs/metrics-exporter-prometheus) — optional Prometheus recorder
- [OpenTelemetry semantic conventions — naming](https://opentelemetry.io/docs/specs/semconv/general/naming/)
