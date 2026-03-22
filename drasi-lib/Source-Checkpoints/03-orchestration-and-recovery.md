# Orchestration and Recovery

> **Parent**: [Checkpoint-Based Recovery Overview](./00-checkpoint-recovery-overview.md)

---

## 1. Overview

This document specifies the orchestration-layer changes required for checkpoint-based recovery. It covers:

- Compatibility validation at startup (the enforcement layer for the compatibility matrix).
- The startup decision flow - how the system chooses between resume-from-checkpoint and fresh bootstrap.
- Recovery policies.
- Configuration validation and cross-component warnings.
- Multi-source coordination edge cases.

The orchestration layer lives primarily in `QueryManager` and `DrasiLib`, with validation hooks during `add_source()`, `add_query()`, and `start_query()`.

---

## 2. Compatibility Validation

### The Matrix

The system should enforce compatibility between query persistence and source durability:

| Query Index | Source Durability | Allowed | Delivery Guarantee |
|-------------|------------------|---------|-------------------|
| Volatile (Memory) | Any | ✅ | At-Most-Once |
| Persistent (RocksDB/Garnet) | Log-Tailing (Postgres, Kafka, MSSQL) | ✅ | At-Least-Once |
| Persistent (RocksDB/Garnet) | Transient + Durable WAL | ✅ | At-Least-Once |
| Persistent (RocksDB/Garnet) | Transient + No WAL | ❌ Error | Startup failure |

### Rationale for the Rejection

A persistent query against a volatile source creates an **unrecoverable inconsistency** on crash:

1. Query commits events to its persistent index.
2. Process crashes.
3. Query restarts, opens its persistent index (stale state from before the crash).
4. Query has a checkpoint for the volatile source, sends `resume_from`.
5. Volatile source has no replay capability and it cannot honor `resume_from`.
6. The query's index permanently diverges from the source's current state.

There is no automatic recovery path. Even `auto_reset` (wipe and re-bootstrap) doesn't fully solve this, because the events previously ACK'd by the volatile source to the upstream system have been lost.

### Mixed-Source Queries

If a persistent query subscribes to multiple sources and **any** source is volatile (no WAL, not log-tailing), the entire subscription is rejected. Partial recoverability is worse than no recoverability. If we were to allow this, the persistent index would contain replayed data from durable sources mixed with gaps from the volatile source, producing silently incorrect results.

### Enforcement Point

We can add validation for this in `QueryManager::start_query()`, during the subscription phase. After the query is built but before any source subscriptions are created. This check will run every time the query starts as a source's durability status could change between restarts.

---

## 3. Startup Decision Flowchart

When a query starts, the `QueryManager` follows this decision tree for each source subscription:

```
start_query(id)
  │
  ├─ Is index volatile?
  │    YES → resume_from: None
  │    │      enable_bootstrap: (from config)
  │    │      request_position_handle: false
  │    │      → FRESH START (current behavior)
  │    │
  │    NO ↓
  │
  ├─ Compute Config Hash (query + sources + joins)
  │    │
  │    ├─ Matches stored hash?
  │    │    NO  → FRESH START (Wipe index, Full bootstrap)
  │    │    │
  │    │    YES ↓
  │    │
  │    ├─ Read checkpoint for this source from persistent index
  │    │    │
  │    │    ├─ Checkpoint found (sequence = N)?
  │    │    YES ↓
  │    │    │
  │    │    ├─ Subscribe with resume_from: Some(N)
  │    │    │   request_position_handle: true
  │    │    │    │
  │    │    │    ├─ Source accepts?
  │    │    │    │    YES → RESUME FROM CHECKPOINT
  │    │    │    │           Skip bootstrap.
  │    │    │    │           Dedup events ≤ N.
  │    │    │    │
  │    │    │    │    NO (PositionUnavailable) ↓
  │    │    │    │
  │    │    │    ├─ Consult recovery_policy:
  │    │    │    │    │
  │    │    │    │    ├─ strict → FAIL STARTUP
  │    │    │    │    │    Log error with gap details.
  │    │    │    │    │    Require manual intervention.
  │    │    │    │    │
  │    │    │    │    ├─ auto_reset → AUTO RESET
  │    │    │    │         Clear all indexes and checkpoints.
  │    │    │    │         Subscribe with resume_from: None
  │    │    │    │         Full bootstrap.
  │    │    │    │         Handover protocol sets new checkpoints.
  │    │
  │    ├─ No checkpoint found?
  │         → FRESH START
  │           resume_from: None
  │           enable_bootstrap: (from config)
  │           request_position_handle: true
  │           Full bootstrap if enabled.
  │           Handover protocol sets initial checkpoints.
```

### Sources subscribed are not independent

A persistent query may subscribe to more than one source. If a checkpoint is found for one or more sources, but not for others - the query must then go into a full bootstrap. This is because it cannot re-use its index from past run as new sources may have been added.

Even if checkpoints are found for all sources, if any one returns `PositionUnavailable` and recovery policy is `auto_reset`, the entire index is wiped. This is necessary because the index may contain cross-source correlations (joins) that would be inconsistent if only one source's data was reset.

---

## 4. Recovery Policies

### Configuration

Recovery policy should be configured per-query, with an optional global default:

```rust
// Per-query in YAML
queries:
  - id: my_query
    recovery_policy: strict  // or auto_reset

// Global default via builder API
DrasiLib::builder()
    .with_default_recovery_policy(RecoveryPolicy::Strict)
    .build()
    .await?;
```

Per-query configuration overrides the global default. If neither is set, the default is `Strict`.

### `Strict` Policy

When a gap is detected (source returns `PositionUnavailable`):

1. The query transitions to `ComponentStatus::Error`.
2. A component event is emitted with details:
   ```
   "Gap detected for source 'orders_db': requested resume from sequence 5000,
    but earliest available is 7500 (gap: 2500 events).
    Recovery policy is 'strict' - manual intervention required.
    Options: (1) Change recovery_policy to 'auto_reset' and restart.
    (2) Manually clear the query's index and restart for a fresh bootstrap."
   ```
3. The query does not start.
4. Other queries are unaffected.

### `AutoReset` Policy

When a gap is detected:

1. Log a warning with gap details.
2. Clear the query's persistent index:
   - Call `clear()` on ElementIndex, ResultIndex, FutureQueue.
   - Call `clear_checkpoints()` on the `CheckpointWriter`.
3. Re-subscribe with `resume_from: None` for all sources (not just the failed one).
4. Full bootstrap proceeds.
5. Handover protocol sets new checkpoints.
6. Query transitions to `Running` normally.

---

## 5. Lifecycle Operations

### `stop_query`

When a query is stopped:

1. Shutdown signal sent to processor task.
2. Processor task exits the main loop.
3. Subscription forwarder tasks are aborted.
4. Position handles are dropped (query's `Arc` refcount decreases).
5. Source detects handle drop during next periodic scan, removes from tracking list.
6. Checkpoints remain in the persistent index (for resume on next start).

The checkpoints are **not** cleared on stop. This allows `start_query` to resume from where it left off.

### `delete_query`

When a query is deleted:

1. Stop the query (steps above).
2. If `cleanup: true`:
   - Clear the persistent index (all CFs).
   - Clear all checkpoints.
   - The query's storage is fully released.
3. If `cleanup: false`:
   - Index and checkpoints remain on disk.
   - A future query with the same ID could potentially resume (uncommon).

### `update_query` (Reconfiguration)

When a query is reconfigured with a new definition:

1. Stop the old query.
2. Replace the query configuration.
3. Start the newly configured query.

All logic regarding whether to reuse the index or wipe it is handled automatically during `start_query` by the **Config Hashing** mechanism (see doc `02`). If the AST, sources, or joins changed, the hash mismatch will automatically trigger an index wipe and full bootstrap. Operational parameters like `priority_queue_capacity` or `recovery_policy` do not factor into the hash, allowing seamless resumption.

---

## 6. Error Reporting

### Error Types

```rust
/// Errors specific to checkpoint-based recovery.
pub enum RecoveryError {
    /// A persistent query requires all sources to support replay,
    /// but one or more do not.
    IncompatibleSource {
        query_id: String,
        source_id: String,
        reason: String,
    },

    /// Auto-reset was triggered. The index has been cleared and the
    /// query will re-bootstrap.
    AutoResetTriggered {
        query_id: String,
        reason: String,
    },
}
