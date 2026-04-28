# Separate Event Sequence from Source Position

* Project Drasi - April 2026 - Daniel Gerlag (@danielgerlag)

**Status:** Draft
**Related:** [#379 — MSSQL Source Replay Support](https://github.com/drasi-project/drasi-core/issues/379)
**Parent:** [Checkpoint-Based Recovery Overview](./00-checkpoint-based-recovery.md)

## Overview

Drasi's source framework needs to track positions in external change streams so that, after a restart, sources can resume from where they left off rather than replaying from the beginning. The current checkpoint design (docs 00–03) conflates two distinct concerns into a single `u64` type:

1. **Event ordering** — the framework needs a lightweight, comparable value to order events, detect gaps, and compute watermarks across subscribers.
2. **Stream resumption** — each source needs its own native position token to seek back into its change stream on restart.

The `u64` approach works for Postgres (whose WAL LSN is a u64), but most databases don't have u64-sized positions. A single type cannot serve both purposes well across all source types. This document proposes splitting the current design into two types: a framework-owned `u64` sequence for ordering, and a source-owned opaque byte buffer for stream resumption.

## Terms and definitions

| Term | Definition |
|------|------------|
| EventSequence | A `u64` value assigned by the framework for event ordering, gap detection, watermarks, and dedup. Monotonic per source. |
| SourcePosition | An opaque byte buffer (`Bytes`) provided by the source plugin containing its native position token. Only the source can interpret these bytes. |
| Checkpoint | The persisted pair of `(EventSequence, SourcePosition)` for a given source, stored in the result index. |
| Confirmed Position Watermark | The minimum sequence across all active subscribers, used to determine the safe source position for upstream advancement. |
| Resume Token | A source-specific byte payload used to seek back into a change stream on restart (e.g., Postgres WAL LSN, MSSQL `start_lsn ∥ seqval`, MongoDB resume token). |

## Objectives

### User scenarios

1. **Source plugin author** building an MSSQL source needs to attach a 20-byte CDC LSN as the resume token. Under the current `u64`-only design, this is not possible without lossy encoding.
2. **Source plugin author** building a MongoDB source needs to pass an opaque ~80-byte resume token back to the driver on restart. The token cannot be compared or ordered — it is only meaningful to the MongoDB driver.
3. **Framework maintainer** needs to order events, detect gaps, and compute watermarks across all subscriber queries using a fast, comparable value.
4. **Operator** deploying Drasi with persistent queries expects that after a restart, queries resume from where they left off with no data loss and no extra I/O operations.

### Goals

- Cleanly separate what the framework needs (a `u64` for ordering) from what sources need (their native position bytes for resumption).
- Impose no size limits on source position tokens.
- Add no new I/O operations — position bytes piggyback on existing result index writes.
- Keep source plugin authoring simple — attach bytes, interpret on restart.
- Support all current and foreseeable source types (see compatibility table below).

### Non-Goals

- Redesigning the checkpoint persistence layer. The result index write path is extended, not replaced.
- Multi-source checkpoint coordination changes. The per-source keying model from doc 02 applies unchanged.
- Changes to the core query engine (`ContinuousQuery`, `process_source_change`). The core remains untouched.

## Design requirements

### Requirements

| Source | Native Position | Size | Comparable? |
|--------|----------------|------|-------------|
| Postgres | WAL LSN | 8 bytes | ✅ Numeric u64 |
| Oracle | SCN | 8 bytes | ✅ Numeric u64 |
| MSSQL | `start_lsn ∥ seqval` | 20 bytes | ✅ Lexicographic |
| MySQL | GTID set | 40–80+ bytes | ⚠️ Partial order only |
| MongoDB | Resume token | 40–80 bytes | ❌ Opaque to client |
| Cosmos DB | Continuation token | 100+ bytes | ❌ Opaque |
| DynamoDB | Shard iterator | 200+ bytes | ❌ Opaque |
| Kafka | `topic:partition:offset` | Composite | ⚠️ Per-partition only |

The design must accommodate all of these without forcing native positions into a fixed-size comparable buffer.

### Dependencies

- **Checkpoint-Based Recovery (docs 00–03):** This document is an amendment to the existing checkpoint design. The nested transaction model, `CheckpointWriter` trait, position handles, and recovery policies from docs 00–03 remain in effect. This document modifies the types that flow through those mechanisms.
- **Result Index (drasi-core):** The `ResultSequenceCounter` trait and its RocksDB/Garnet implementations are extended to carry source position bytes.
- **Source SDKs (Rust, Java, .NET):** The `SourceEventWrapper` type and subscription protocol are modified. SDK updates are required for source plugins to adopt the new position field.
- **FFI boundary:** The `FfiSourceEvent` type must be extended with an optional byte buffer for `source_position` (see Open Issues).

### Out of scope

- **Local WAL for transient sources:** The redb-backed WAL design from doc 01 is unaffected. Transient sources with WAL will encode their monotonic counter as the `SourcePosition` bytes.
- **Reaction checkpoints:** The `ResultSequenceCounter` is extended but its role in reaction-facing sequence numbers is not changed.

## Design

### High-level design

The core insight is that event ordering and stream resumption are different concerns. The framework should own one; sources should own the other.

**Two new types replace the single `u64` sequence:**

```rust
// In the framework — lightweight, universal ordering.
// Assigned by the framework, not the source.
// Used for: event ordering, gap detection, watermarks, dedup.
type EventSequence = u64;

// In the source — opaque native position for resume/seek.
// Provided by the source, never interpreted by the framework.
// Used for: seeking back into the change stream on restart.
struct SourcePosition(Bytes);  // or SmallVec<[u8; 24]>
```

### Architecture Diagram

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                         SOURCE PLUGIN                            │
 │                                                                  │
 │  CDC poll loop produces:                                         │
 │    SourceEventWrapper {                                          │
 │      event: SourceChange { ... },                                │
 │      source_position: Some(vec![...20 bytes...]),  ← MSSQL LSN  │
 │      sequence: None,  ← source does NOT set this                 │
 │    }                                                             │
 └──────────┬───────────────────────────────────────────────────────┘
            │
            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                      FRAMEWORK (SourceBase)                      │
 │                                                                  │
 │  On dispatch, framework assigns:                                 │
 │    event.sequence = self.next_sequence();  // monotonic u64      │
 │                                                                  │
 │  Dispatches event to subscriber queries.                         │
 │  The (sequence, source_position) pair is the "checkpoint."       │
 └──────────┬───────────────────────────────────────────────────────┘
            │
            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                      QUERY MANAGER                               │
 │                                                                  │
 │  Processes the SourceChange through the continuous query engine.  │
 │  After processing, persists the checkpoint:                      │
 │    result_index.apply_checkpoint(sequence, &source_position);    │
 │                                                                  │
 │  This piggybacks on the EXISTING result index write —            │
 │  no additional I/O operation.                                    │
 └──────────┬───────────────────────────────────────────────────────┘
            │
            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                      ON RESTART                                  │
 │                                                                  │
 │  1. Query manager reads result_index.get_checkpoint()            │
 │     → returns (last_sequence: u64, source_position: Vec<u8>)     │
 │                                                                  │
 │  2. Passes source_position bytes to source.subscribe(            │
 │       resume_from: source_position_bytes,                        │
 │     )                                                            │
 │                                                                  │
 │  3. Source uses its own bytes to seek:                            │
 │     - Postgres: u64 from bytes → WAL LSN                         │
 │     - MSSQL: 20 bytes → start_lsn + seqval                      │
 │     - MongoDB: N bytes → resume token (passed directly to driver)│
 │     - MySQL: N bytes → GTID set string                           │
 │                                                                  │
 │  4. Framework resets its sequence counter from last_sequence.     │
 │     Stream events with sequence > last_sequence are new;         │
 │     events ≤ last_sequence are deduped.                          │
 └──────────────────────────────────────────────────────────────────┘
```

### Detail design

#### Event structure

The `SourceEventWrapper` is modified to carry both a framework-assigned sequence and a source-provided position:

```rust
pub struct SourceEventWrapper {
    pub source_id: String,
    pub event: SourceEvent,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub profiling: Option<ProfilingMetadata>,

    // Framework-assigned. Monotonic per source. Used for ordering,
    // watermarks, and dedup. Source plugins do NOT set this.
    pub sequence: u64,

    // Source-provided. Opaque bytes that only the source can interpret.
    // Framework persists them alongside the sequence and returns them
    // on restart via subscribe(resume_from: ...).
    // None for volatile sources that don't support replay.
    pub source_position: Option<Bytes>,
}
```

Key changes from the current design (doc 01):
- `sequence` is no longer `Option<u64>` set by the source — it is always present and assigned by the framework.
- `source_position` is a new field: opaque bytes that only the source can interpret.

#### Framework sequence assignment

The framework stamps every event with a monotonic sequence in `SourceBase::dispatch()`:

```rust
fn dispatch(&self, mut event: SourceEventWrapper) {
    event.sequence = self.next_sequence.fetch_add(1, Ordering::Relaxed);
    // source_position was set by the source plugin; framework just carries it
    self.dispatchers.send(event);
}
```

The source plugin does not set `sequence`. It only provides `source_position`.

#### Persistence — no extra I/O

The result index already writes to persistent storage (RocksDB or Garnet) after processing each source change. Today it persists `ResultSequence { sequence, source_change_id }`. The proposal extends this to:

```rust
pub struct ResultCheckpoint {
    pub sequence: u64,
    pub source_change_id: Arc<str>,
    pub source_position: Option<Vec<u8>>,  // NEW: carried alongside existing data
}
```

This is persisted in the **same write** that already stores `ResultSequence`. The bytes are additional payload in the same key-value write. There is no separate database, no separate write, no separate fsync. The position bytes piggyback on an operation that already happens on every event.

On restart, `get_checkpoint()` reads back the full `ResultCheckpoint`, and the `source_position` bytes are passed to the source.

**Honest caveat**: The result index write itself is I/O. It already exists today. What this proposal does NOT add is a *new* I/O operation. The source position bytes increase the size of the existing write by however many bytes the source position is (8 for Postgres, 20 for MSSQL, ~80 for MongoDB), which is negligible relative to the accumulator and result data already being written.

#### Confirmed position watermark

The confirmed position watermark (minimum position across all subscribers) is essential for sources that gate upstream retention — e.g., Postgres advancing its WAL flush LSN, or Kafka committing offsets.

With the two-level design:

1. Each subscriber (query) reports the latest `u64 sequence` it has durably processed, via the existing position handle mechanism.
2. The framework computes `min(sequence)` across all subscribers — this is the confirmed sequence.
3. The framework looks up the `source_position` bytes associated with the confirmed sequence from the result index.
4. It invokes a callback on the source: `source.advance_position(confirmed_position_bytes)`.
5. The source uses those bytes to advance its upstream cursor (flush WAL slot, commit Kafka offset, etc.)

The `u64` watermark computation is trivial and fast. The `source_position` lookup is a single read from the result index (which is already in memory or cached).

#### Source plugin author experience

A source plugin that wants replay support needs to:

1. Return `supports_replay() → true`
2. Attach position bytes to events: `event.set_source_position(my_native_lsn_bytes)`
3. In `subscribe(resume_from)`, interpret the bytes to seek: `let lsn = MyLsn::from_bytes(resume_from)`

That's it. The source doesn't need to:

- Manage a side table
- Assign sequence numbers
- Understand watermarks
- Handle dedup
- Know about other subscribers

#### What the framework handles

- Assigning monotonic sequences
- Persisting `(sequence, position)` — piggybacked on existing writes
- Restoring on restart
- Computing confirmed position across subscribers
- Dedup of replayed events via sequence comparison

#### Advantages of this design

- **Universal ordering**: `u64` comparison for all framework operations is fast and uniform.
- **No size limits**: Source positions can be arbitrarily large byte buffers.
- **Clean separation of concerns**: The framework never interprets source position bytes.
- **Zero new I/O**: Position bytes are piggy-backed on existing result index writes.
- **Simple plugin authoring**: Source authors only attach bytes and interpret them on restart.
- **Straightforward migration**: The existing `u64 sequence` in doc 01 splits into framework-assigned `sequence` + source-provided `source_position`.

#### Disadvantages

- **Memory overhead**: Each in-flight event carries an additional `Option<Bytes>` allocation. For Postgres (8 bytes) this is negligible; for Cosmos DB (100+ bytes) it is still small relative to the event payload.
- **Confirmed position lookup**: Computing the min-watermark is cheap (`u64` comparison), but looking up the corresponding `source_position` bytes requires a result index read (see Open Issues).
- **Protocol change**: The FFI boundary (`FfiSourceEvent`) needs extending with an optional byte buffer.

### API Design

#### Extended `ResultSequenceCounter` trait

```rust
pub trait ResultSequenceCounter: Send + Sync {
    async fn apply_checkpoint(
        &self,
        sequence: u64,
        source_change_id: &str,
        source_position: Option<&[u8]>,
    ) -> Result<(), IndexError>;

    async fn get_checkpoint(&self) -> Result<ResultCheckpoint, IndexError>;
}
```

#### Extended subscription settings

The `resume_from` field changes from `Option<u64>` to `Option<Vec<u8>>`:

```rust
pub struct SourceSubscriptionSettings {
    pub source_id: String,
    pub enable_bootstrap: bool,
    pub query_id: String,
    pub nodes: HashSet<String>,
    pub relations: HashSet<String>,
    pub resume_from: Option<Vec<u8>>,  // CHANGED: was Option<u64>
    pub request_position_handle: bool,
}
```

Sources receive the raw bytes and interpret them according to their own format.

#### Source plugin example (MSSQL)

```rust
async fn subscribe(
    &self,
    settings: SourceSubscriptionSettings,
) -> Result<SubscriptionResponse> {
    if let Some(pos_bytes) = settings.resume_from {
        let seek_lsn = position_to_seek_lsn(&pos_bytes)?;
        self.start_cdc_from(seek_lsn).await?;
    }
    // ...
}
```

#### Source plugin example (Postgres)

```rust
// Postgres encodes its WAL LSN as 8 bytes
fn set_position(&self, event: &mut SourceEventWrapper, end_lsn: u64) {
    event.source_position = Some(Bytes::copy_from_slice(&end_lsn.to_le_bytes()));
}

fn decode_position(bytes: &[u8]) -> u64 {
    u64::from_le_bytes(bytes.try_into().expect("WAL LSN must be 8 bytes"))
}
```

### Alternatives Considered

#### Keep u64-only with source-side encoding

Sources could encode their native positions into a u64 (e.g., by hashing or truncating). This was rejected because:

- Opaque tokens (MongoDB resume tokens, Cosmos continuation tokens) cannot be meaningfully encoded into 8 bytes.
- Truncation or hashing loses the ability to seek — the whole point of the position token.
- It limits which sources can be supported.

#### Variable-length position as the sole type (no separate sequence)

Use the source position bytes for both ordering and resumption. This was rejected because:

- Many source positions are not comparable (MongoDB, Cosmos, DynamoDB).
- Even when comparable (MSSQL), lexicographic comparison of variable-length byte arrays is slower than `u64` comparison.
- The framework's watermark and dedup logic becomes source-specific.

#### Framework-assigned sequence only (no source position)

Have the framework assign sequences and maintain a mapping table from sequence to source position. This was rejected because:

- It requires a separate side table that must be persisted and managed.
- It adds I/O operations that don't exist today.
- The mapping table grows unboundedly until pruned.

## Compatibility impact

This proposal modifies the `SourceEventWrapper` structure and `SourceSubscriptionSettings`, which are used across the source SDKs (Rust, Java, .NET). Changes required:

- **Rust SDK**: Direct type change. `sequence` becomes framework-assigned (sources no longer set it). New `source_position: Option<Bytes>` field added.
- **Java/.NET SDKs**: `resume_from` changes from `long?` to `byte[]?`. Source plugins must be updated to provide and interpret position bytes.
- **FFI boundary**: `FfiSourceEvent` needs an optional byte buffer field for `source_position`.

Existing source plugins that currently set `sequence` will need migration:
- Remove sequence assignment logic.
- Add `source_position` bytes encoding from their native position.
- Update `subscribe()` to interpret `resume_from` as bytes instead of `u64`.

## Supportability

### Telemetry

- Log the size of `source_position` bytes per source type at startup (helps diagnose unexpectedly large positions).
- Metric: `source_position_bytes` histogram per source type.
- Existing sequence-based metrics (watermark lag, dedup count) continue to work unchanged on the `u64` sequence.

### Verification

- **Unit tests**: Verify round-trip encoding/decoding of position bytes for each source type (Postgres u64, MSSQL 20-byte, mock opaque token).
- **Integration tests**: Verify that a query can restart and resume from a persisted `ResultCheckpoint` containing position bytes.
- **Multi-source tests**: Verify that checkpoints are correctly keyed by `(query_id, source_id)` and that position bytes are not mixed between sources.
- **Migration tests**: Verify that upgrading from the u64-only design to the two-level design handles the transition correctly (existing u64 checkpoints are interpretable as 8-byte position buffers).

## Development Plan

### Phase 1: Extend the event and result index types

Modify `SourceEventWrapper` to add `source_position: Option<Bytes>` and change `sequence` to be framework-assigned. Extend `ResultCheckpoint` and `ResultSequenceCounter` trait to carry position bytes.

### Phase 2: Framework assigns sequence, carries position

Modify `SourceBase::dispatch()` to stamp the sequence. Source plugins stop setting sequence and start providing `source_position` bytes.

### Phase 3: Query manager persists checkpoint (no new I/O)

Integrate position bytes into the existing result index write path. The `apply_checkpoint` call carries `source_position` alongside sequence and source_change_id.

### Phase 4: Restart reads checkpoint, passes position to source

Modify startup flow to read `ResultCheckpoint`, extract `source_position` bytes, and pass them to `source.subscribe(resume_from: position_bytes)`.

### Phase 5: Source interprets its own bytes

Update source plugins (starting with MSSQL, then Postgres) to encode their native positions as bytes and decode them in `subscribe()`.

## Open issues

1. **Memory representation for `source_position`**: `Vec<u8>` is simplest but allocates per event. `Bytes` (from the `bytes` crate) is reference-counted and cheaper to clone. `SmallVec<[u8; 24]>` inlines small positions (Postgres, MSSQL) and only heap-allocates for larger ones. Recommendation: start with `Bytes` — zero-copy cloning fits the broadcast dispatch pattern well.

2. **Multi-source queries**: A query subscribed to multiple sources will have one checkpoint per source. The result index needs to key checkpoints by `(query_id, source_id)`, not just globally. This is a straightforward extension of the `stream_state` CF layout from doc 02.

3. **Confirmed position lookup**: Computing the min-watermark is cheap (`u64` comparison). But looking up the `source_position` bytes for the confirmed sequence requires a read from the result index. This could be cached in memory to avoid the lookup. Alternatively, sources that need upstream cursor advancement (Postgres, Kafka) could cache the mapping themselves — they already see every event.

4. **FFI sources (plugin SDK)**: The current `FfiSourceEvent` doesn't carry a position field. This would need to be extended with an optional byte buffer for `source_position`. This is a protocol change for the plugin FFI boundary.

## References

- [Checkpoint-Based Recovery Overview](./00-checkpoint-based-recovery.md)
- [Source Sequencing and Replay](./01-source-sequencing-and-replay.md)
- [Query Checkpointing](./02-query-checkpointing.md)
- [Orchestration and Recovery](./03-orchestration-and-recovery.md)
- [#379 — MSSQL Source Replay Support](https://github.com/drasi-project/drasi-core/issues/379)
