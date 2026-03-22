# Query Checkpointing

> **Parent**: [Checkpoint-Based Recovery Overview](./00-checkpoint-recovery-overview.md)

---

## 1. Overview

This document specifies the query-side changes required for checkpoint-based recovery in Drasi Lib. It covers:

- How queries atomically persist their last-processed sequence alongside index updates.
- How queries read checkpoints on startup to populate `resume_from`.
- How queries filter (dedup) already-processed events on replay.
- The bootstrap-to-streaming handover protocol.
- Conditional activation of the checkpoint machinery based on index volatility.

All changes are conditional: queries using in-memory indexes operate exactly as today with no overhead.

## 2. Atomic Checkpoint Persistence

### The Core Problem

The query's index state and its position/cursor in the source's stream must always be consistent. If they can diverge (index updated but position lost, or position advanced but index rolled back), crash recovery produces incorrect results.

To ensure consistency, we will write both in the **same database transaction**.

### Transaction Scope

`ContinuousQuery::process_source_change()` already wraps all index updates in a single session transaction via `SessionGuard`:

```rust
let guard = SessionGuard::begin(self.session_control.clone()).await?;
let changes = self.execute_source_middleware(change).await?;
let result = self.process_changes_inner(changes).await?;
guard.commit().await?;
```

This transaction atomically commits:

| Component | Column Families (RocksDB) |
|-----------|--------------------------|
| ElementIndex | `elements`, `slots`, `inbound`, `outbound`, `partial` |
| ResultIndex | `values`, `sets`, `metadata` |
| FutureQueue | `queue`, `findex` |

We add **one more write** to the `stream_state` CF in this same transaction:

```
Key:   "source_sequence:{source_id}"
Value: sequence (u64, little-endian)
```

If the process crashes before the transaction commits, both the index updates and the checkpoint roll back together. They are always consistent.

### Nested Transactions to avoid changes in Core Engine

The core engine's `process_source_change` owns the `SessionGuard` internally. Both `session_control` and `change_lock` are private fields on `ContinuousQuery` with no public accessors. Rather than modifying core's API, we will add **nested transaction support** in the index plugin implementations to write the checkpoint into the same transaction from the lib layer.

The lib layer wraps core's processing in an outer transaction:

```
Lib:    session_control.begin()              <- creates the real DB transaction (depth=1)
Core:     session_control.begin()            <- nested, no-op (depth=2)
Core:     ...index writes...
Core:     session_control.commit()           <- nested, no-op (depth=1)
Lib:    checkpoint_writer.stage(src, seq)    <- writes into the still-open transaction
Lib:    session_control.commit()             <- actual commit (depth=0): index + checkpoint
```

This is a well-established pattern (for example - SQL savepoints). Inner begin/commit calls are no-ops when an outer transaction is already active. Only the outermost commit does real work. This will need only need a small change in each index plugin to add a nesting depth counter to the `SessionControl` implementations, and a new `CheckpointWriter` trait will be defined in lib.

**The core remains untouched and does not need to know about source checkpoints.**

### The `CheckpointWriter` Trait

This trait will be added in lib to provide checkpoint read/write operations that work within the active database transaction:

```rust
/// Trait for checkpoint persistence, defined in lib.
/// Implemented by index plugin SessionControl types that support persistent storage.
pub trait CheckpointWriter: Send + Sync {
    fn stage_checkpoint(&self, source_id: &str, sequence: u64) -> Result<(), IndexError>;
    fn read_checkpoint(&self, source_id: &str) -> Result<Option<u64>, IndexError>;
    fn read_all_checkpoints(&self) -> Result<HashMap<String, u64>, IndexError>;
    fn clear_checkpoints(&self) -> Result<(), IndexError>;
    fn write_config_hash(&self, hash: u64) -> Result<(), IndexError>;
    fn read_config_hash(&self) -> Result<Option<u64>, IndexError>;
}
```

The `stage_checkpoint` call writes directly to the active database transaction via the shared session state. The read methods are called at query startup before any transaction begins.

Because `drasi-core` owns the `IndexBackendPlugin` interface but `CheckpointWriter` is an orchestrator pattern specific to `drasi-lib`, discovery of the checkpointing capability will happen inside `drasi-lib`'s `IndexFactory`. When `drasi-lib` initializes a storage plugin (like RocksDB), it natively knows the concrete type. The factory will create the shared session state, wrap it into the `IndexSet` to satisfy `drasi-core`'s requirements, and independently retain an `Arc<dyn CheckpointWriter>` reference to use within its orchestrator event loop for operations like `.stage_checkpoint()`.

This keeps `drasi-core` completely unaware of the checkpointing domain.

### Nesting Support in SessionControl Implementations

The `RocksDbSessionState` gains a nesting depth counter (`AtomicU32`). When `begin()` is called while a transaction is already active, it increments depth instead of erroring. When `commit()` is called at depth > 1, it decrements instead of actually committing.

The same pattern applies to `GarnetSessionControl` (nesting depth around `MULTI/EXEC`). `NoOpSessionControl` in core doesn't need nesting, as it's used for in-memory indexes where checkpointing is disabled.

### In-Memory (No-Op) Implementation

When the index is volatile (in-memory), **ideally we will not have checkpoint tracking.** The lib layer should not track sequences, not wrap transactions, and skips all checkpoint operations. No trait implementation is needed for `NoOpSessionControl`.

When an in-memory process restarts, the graph index is empty. Trying to resume streaming updates on an empty graph would corrupt outputs and throw errors. Therefore, an in-memory query's only valid recovery path is a **full start over**: it subscribes to sources with `resume_from: None`, triggering a fresh snapshot/bootstrap. By short-circuiting sequence tracking entirely, the in-memory pipeline remains frictionless without the overhead of sequence maps.

### Storage Layout & Future-Proofing

We explicitly isolate stream sequence tracking from core graph data. 
- In **RocksDB**, we define a dedicated **`stream_state`** Column Family instead of reusing `metadata`. 
- In **Garnet**, we use a dedicated **`query:{<query_id>}:stream_state`** hash representation.

This is because sequence numbers execute a very high-frequency *overwrite* pattern on every single event, while actual graph keys (`elements`, `results`) get inserted, updated, or deleted as the stream flows. Reusing existing CFs triggers unnecessary compactions of core graph state. 

This design also perfectly lays the foundation for future Query->Reaction pipeline persistence. When we introduce transactional outboxes and result sequence numbers, we will extend this storage separation:
- **`stream_state`** (RocksDB CF / Garnet Hash): Holds all scalar overwrite orchestrator state (`source_sequence:<id>`, `result_sequence`).
- **`outbox`** (RocksDB CF / Garnet Stream): Can holds a limited-size FIFO queue of emitted result diffs (high-frequency *appends* with range deletes/trimming).
- **`live_results`** (RocksDB CF / Garnet Hash): Holds the materialized snapshot of the query's current output state.

### Relationship to Existing `ResultSequenceCounter`

The existing `ResultSequenceCounter` trait (`apply_sequence`, `get_sequence`) stores a single `(sequence, source_change_id)` pair. It is **not** used for source checkpoint tracking - it appears to be designed for reaction-facing sequence numbers (tracking the order of result emissions).

The source checkpoint mechanism we're introducing here is to address a separate concern and it will not replace or interfere with `ResultSequenceCounter`.

---

## 3. Reading Checkpoints on Startup

### The "All or Nothing" Rule
It is not possible to safely resume a query such that some sources resume from a stream while bootstrapping others. Doing that evaluates partial stream changes against incomplete data, corrupting the results.

Therefore, Drasi lib enforces a strict **"All or Nothing"** startup rule. A query either perfectly resumes from checkpoints for all sources, or wipes its index entirely and performs a full bootstrap for all config-defined sources. 

### Flow (Config Hashing)

When a query with a persistent index (RocksDB/Garnet) starts, Drasi evaluates its configuration hash to determine whether to resume or rebuild:

```
1. Compute current_hash = Hash(QueryConfig.query + sources + joins)
2. Open persistent index (RocksDB/Garnet)
3. Read stored_hash = checkpoint_writer.read_config_hash()
4. If stored_hash == current_hash (Match):
   a. It is safe to resume. 
   b. Read all source_checkpoints
   c. Set resume_from = Some(seq) for ALL sources. Sources skip Bootstrap where possible.
5. If stored_hash != current_hash OR is None (Mismatch):
   a. The AST changed, a new source was added, or it is a fresh deployment!
   b. Truncate/Wipe the persistent index (Elements, Results, stream_state).
   c. Write current_hash to the empty stream_state CF.
   d. Set resume_from = None for ALL sources. Full Bootstrap.
6. Subscribe to each source with the populated resume_from.
```

### First Start vs. Restart

| Scenario | Hash Check | Behavior |
|----------|------------|----------|
| First start (empty index) | None found | Wipe index (safety). Write new hash. `resume_from: None` for all sources. **Full bootstrap.** |
| Crash Recovery / Graceful Restart | Matches | `resume_from: Some(seq)` for all sources. **Skip bootstrap.** |
| Update (Config/Sources changed) | Mismatch | Wipe index. Write new hash. `resume_from: None` for all sources. **Full bootstrap.** |
| Fallback (Source rejected position) | N/A | Wipe index. Write new hash. `resume_from: None` for all sources. **Full bootstrap.** |

### Interaction with `enable_bootstrap`

When `resume_from` is `Some(seq)`, bootstrap is skipped regardless of the `enable_bootstrap` configuration. The persistent index already has the base state, so bootstrapping would re-insert data that already exists.

When `resume_from` is `None`, the existing `enable_bootstrap` flag controls whether bootstrap is performed (current behavior, unchanged).

---

## 4. Dedup on Replay

### The Problem

Duplicate events can arrive at a query for two main reasons:

1. **Crash Recovery**: The source replays events from the last confirmed global watermark position. Some of these events may have already been processed and committed to the query's index before the crash.
2. **Source Rewinds**: When a new query subscribes to a running source, or an older query resumes, it may ask the source to rewind to an older `resume_from` sequence. Because the source broadcasts through a unified channel, sibling queries that are already up-to-date will receive events they have already seen.

Processing these duplicates again would cause incorrect results (duplicate inserts, double-counted aggregations, etc.).

### The Solution

Before processing each event, the query compares the event's sequence against the stored checkpoint for that source:

```rust
// In the query processor loop, after dequeuing from priority queue:
if let Some(sequence) = event.sequence {
    if let Some(checkpoint) = source_checkpoints.get(&event.source_id) {
        if sequence <= *checkpoint {
            // Already processed - skip
            debug!("Dedup: skipping event seq={sequence} from '{}' (checkpoint={checkpoint})",
                   event.source_id);
            continue;
        }
    }
}

// Process the event normally
let result = continuous_query.process_source_change(change, source_checkpoint).await;
```

**Note:** Dedup filtering happens **per-source**. Events from different sources have independent sequence spaces.

### In-Memory Checkpoint Cache

Reading the checkpoint from RocksDB/Garnet on every event would be expensive. Instead, the query processor maintains an in-memory `HashMap<String, u64>` cache of checkpoints:

- **Populated on startup**: from `read_all_checkpoints()`.
- **Updated after each commit**: when `process_source_change` succeeds, update the cache entry.
- **Used for dedup**: the comparison above reads from the cache, not from disk.

This cache is authoritative during the query's lifetime. On crash, it's lost, but the persistent checkpoints in RocksDB are the recovery source. The cache is rebuilt from them on restart.

### Transaction Dedup Behavior

Some sources have unique LSN for each event within transaction (for example in Postgres, the XLogData `end_lsn` is separate for each entry - and there might be multiple steps per transaction). The checkpoint in Drasi-Lib will advance per-event as each is processed. This means that partial transaction replay will be handled correctly, as already-processed events are filtered, remaining events are reprocessed.

The checkpoint granularity is per-event, not per-transaction.

---

## 5. Bootstrap-to-Streaming Handover

### The Problem

When a query starts fresh (no checkpoint) or falls back to full bootstrap, there is a critical transition point between bootstrap (snapshot) and streaming (live events). During bootstrap, live events are buffered in the priority queue. When bootstrap completes and the gate opens, the query must know:

1. What is the last-position covered by the bootstrap
2. Are the snapshot's positions compatible with the stream's positions?

Getting this wrong causes either data loss (gap between snapshot and stream) or duplication (overlap processed twice).

### The `BootstrapResult` Type

The `BootstrapProvider::bootstrap()` return type needs to be extended to include handover metadata:

```rust
async fn bootstrap(
    &self,
    request: BootstrapRequest,
    context: &BootstrapContext,
    tx: BootstrapEventSender,
    settings: Option<&SourceSubscriptionSettings>,
) -> Result<BootstrapResult>;   // NEW - Return Type changed from u64 (plain event count)
```

The `BootstrapResult` returned by the provider should contain metadata to coordinate the handover:
- `event_count`: Number of events sent.
- `last_sequence`: The sequence position of the snapshot (e.g., LSN), if applicable.
- `sequences_aligned`: A boolean indicating if the bootstrap and stream use the same sequence namespace.

The `sequences_aligned` and `last_sequence` should be made available as configurations on the bootstrap provider.

### The Four Handover Scenarios

| Scenario | Bootstrap | Stream | `BootstrapResult` | Query Action |
|----------|-----------|--------|--------------------|--------------|
| **Homogeneous** (Postgres/Postgres) | Snapshot at LSN 500 | Stream from LSN 0 | `last_sequence: Some(500), aligned: true` | Set checkpoint = 500. Dedup filters buffered events <= 500. |
| **Heterogeneous** (File/Postgres) | Static file, no LSN | Stream from LSN 500 | `last_sequence: None, aligned: false` | Set checkpoint = 0. Process all buffered events. |
| **Heterogeneous** (Postgres/HTTP-WAL) | Snapshot at LSN 10,000 | Counter starts at 1 | `last_sequence: Some(10000), aligned: false` | Set checkpoint = 0. Ignore bootstrap LSN. Process all buffered events. |
| **Heterogeneous** (File/HTTP-WAL) | Static file, no LSN | Counter starts at 1 | `last_sequence: None, aligned: false` | Set checkpoint = 0. Process all buffered events. |

### Handover Processing in the Query Manager

When all bootstrap tasks complete, the query manager collects the `BootstrapResult` from each source. For each source:
- If sequences are aligned and a `last_sequence` is provided, that sequence becomes the source's new checkpoint, and is persisted.
- Otherwise, the checkpoint is set to `0`, ensuring all buffered stream events are processed.

After checkpoints are updated, the query manager opens the bootstrap gate, and the processor begins draining the priority queue (subject to standard checkpoint-based deduplication).

### Carrying Handover Metadata Through Bootstrap State

When a bootstrap task finishes, it writes its `BootstrapResult` into the shared state map. The supervisor task reads the results from all sources and applies the handover logic above.

### Default Alignment Rules

Optionally, Bootstrap providers could have default `sequences_aligned` for some common cases:

| Provider Type | Source Type | Default `sequences_aligned` |
|---------------|-----------|---------------------------|
| Postgres bootstrapper | Postgres source | `true` |
| MSSQL bootstrapper | MSSQL source | `true` |
| File bootstrapper | Any source | `false` |
| Script bootstrapper | Any source | `false` |
| Noop bootstrapper | Any source | `false` |
| Application bootstrapper | Application source | Configurable |

Users can override via configuration for custom alignment scenarios (e.g., a purpose-built file that they know is aligned with a specific source position).

This can be a future improvement.

### Re-Bootstrap on Recovery Fallback

If a query's `resume_from` is rejected by the source (gap detected, `PositionUnavailable`), and the recovery policy is `auto_reset`, the query clears its index and performs a full bootstrap. In this case:

1. All checkpoints are cleared (`clear_checkpoints()`).
2. The query subscribes with `resume_from: None`.
3. Bootstrap runs normally.
4. The handover protocol applies, setting initial checkpoints from the `BootstrapResult`.

This is the same path as a fresh start. The handover protocol works identically regardless of whether it's the first start or a recovery fallback.

---

## 6. Conditional Activation

### Decision Logic

The checkpoint machinery activates based on the query's storage backend. The decision is made **once at query startup** and determines behavior for the query's entire lifetime:

```rust
let checkpointing_enabled = !index_factory.is_volatile(&storage_backend);
```

| Storage Backend | `is_volatile()` | Checkpointing | Position Handle | `resume_from` |
|-----------------|-----------------|---------------|-----------------|---------------|
| None (default)  | `true`          | Disabled      | Not requested   | Always `None` |
| Memory          | `true`          | Disabled      | Not requested   | Always `None` |
| RocksDB         | `false`         | Enabled       | Requested       | Read from index |
| Garnet          | `false`         | Enabled       | Requested       | Read from index |

### What "Disabled" Means

When checkpointing is disabled, we should aim to have the same behavior we have today (wherever possible):

- Lib should not wrap `process_source_change` in an outer transaction.
- No checkpoint is staged. The `CheckpointWriter` should not be invoked.
- No checkpoint is read on startup. `resume_from` should always be sent to Sources as `None`.
- No position handle is requested from the source. The query should excluded from the source's min-watermark calculation. The source should not hold back upstream advancement for this query.
- No dedup filtering. All events should be processed unconditionally.
- Bootstrap should always runs (if `enable_bootstrap` is configured). 

### What "Enabled" Means

When checkpointing is enabled:

- **Lib wraps `process_source_change` in an outer transaction** (nested begin/commit). After core returns, lib stages the checkpoint and commits the outer transaction.
- **Checkpoints are read on startup** via `checkpoint_writer.read_all_checkpoints()` and used to populate `resume_from`.
- **A position handle is created** by the source and returned in `SubscriptionResponse`. The query writes to it after each successful outer commit.
- **Dedup filtering is active**. Events with `sequence ≤ checkpoint` are skipped.
- **Bootstrap may be skipped** if checkpoints exist (resume from checkpoint instead).

### Position Handle Interaction

The position handle (`Arc<AtomicU64>` from doc 01) is only meaningful when checkpointing is active. The query manager communicates this to the source via the subscription:

```rust
pub struct SourceSubscriptionSettings {
    // ... existing fields ...
    pub resume_from: Option<u64>,
    pub request_position_handle: bool,  // NEW - will be true only for persistent queries
}
```

When `request_position_handle` is `false`, the source should not create a handle, and not add it to its tracking list. This means that this query will not be included in the min-watermark calculation. This helps volatile query to continue to work as they do today.

---

## 7. Query Processor Integration

### Complete Processing Loop

The query processor loop integrates nested transactions, checkpointing, deduplication, and position handle updates without modifying `drasi-core`'s evaluation interfaces.

When checkpointing is enabled:
1. **Startup**: The checkpoint cache will be populated by reading all sequence markers from the storage volume.
2. **Event Parsing**: Streaming events pop from the `priority_queue` containing the corresponding source's ID and `sequence`.
3. **Deduplication Check**: If `sequence <= checkpoint_cache(source_id)`, the event will be dropped.
4. **Outer Transaction Processing**: If the event passes dedup, `drasi-lib` will explicitl calls `session_control.begin()`.
5. **Core Evaluation**: The processor will call `continuous_query.process_source_change(source_change)`. Core will evaluate the inner transaction and drop into state execution just like it does today.
6. **Inner Commit and State Checkpoint Stage**: On a successful `Ok(results)` return from evaluate, `checkpoint_writer.stage_checkpoint(source_id, seq)` is called and the outer transaction calls completion.
7. **Position Update**: The memory cache updates and signals the `position_handles` cache updating the current cursor mark for stream min-watermark propagation.

### Key Observations

- No changes to `process_source_change` in the core library. It remains unaware of source checkpointing.
- The outer `begin()`/`commit()` wrapping is only active when checkpointing is enabled for persistent index. For volatile indexes, core's internal transaction management is used directly as today.
- `process_due_futures()` is NOT wrapped in an outer transaction. Future-due events are internally generated and don't have a source sequence. Core manages its own transaction for these.
- The checkpoint cache is updated only after the outer commit succeeds. If `stage_checkpoint` or `commit` fails, the cache retains the old value.
- **`change_lock` Gap Constraint**: Core releases its internal `change_lock` mutex when `process_source_change` completes, but **before** the outer transaction commits. **Because `drasi-lib` processes events sequentially in a single-threaded loop, this brief unlocked window poses no concurrency risk. However, direct users of the `ContinuousQuery` API must be mindful of this if sharing evaluation across threads.**

### Error Handling

When `process_source_change` fails:

- Core's `SessionGuard` rolls back its inner transaction. But with nesting, this decrements depth (or sets it to 0 and triggers actual rollback if core panics).
- The lib layer detects the error and calls `session_control.rollback()` on the outer transaction, ensuring the real database transaction is rolled back.
- The checkpoint is **not** staged (the `stage_checkpoint` call was never reached).
- The checkpoint cache is **not** updated.
- The position handle is **not** updated.
- The event is lost from the priority queue.

**With checkpoint recovery**: On restart, the checkpoint still points to the pre-failure position. The source replays from there, and the failed event is redelivered. At-least-once is maintained.

**Without restart** (current behavior): The event is lost for the remainder of this run. Adding in-loop retry for transient failures (e.g., temporary Garnet timeout) can be taken up as a future enhancement work item.