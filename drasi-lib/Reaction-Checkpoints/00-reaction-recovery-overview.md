# Reaction Recovery Overview

## 1. Problem Statement

Drasi Lib currently provides **at-most-once delivery** between queries and reactions. The channel connecting them is an in-memory Tokio `mpsc` or `broadcast` channel. If the process crashes, all in-flight results are lost permanently because:
1. The query commits the source checkpoint *before* delivering results to the channel.
2. An un-delivered `QueryResult` simply evaporates.
3. On restart, the query skips re-evaluating the event, leaving the side-effect unexecuted.

Additionally, reactions have **no bootstrap story**. New reactions joining a running query miss all past state.

## 2. Design Goals

1. **At-least-once delivery** for persistent queries and durable reactions.
2. **O(1) Live Result Set Mutations** (replace `O(N)` `Vec` scans with a HashMap/KV store).
3. **Strict Decoupling** - Queries use fixed-size outboxes to bound disk growth, rejecting the "min-watermark" / position handle approach.
4. **Zero overhead for volatile queries** - In-memory indexes operate exactly as today with no I/O.
5. **Reaction authors control policy** - The framework provides the outbox and dedup; reactions drive checkpointing and recovery logic.

## 3. Architecture Overview

### Target Flow

```text
Query Processor                                    Reaction
  │                                                  │
  │  outer begin()                                   │
  │    process_source_change() → diffs               │
  │    stage_source_checkpoint(src, seq)             │
  │    result_seq = ++last_result_seq                │
  │    outbox.append(result_seq, QueryResult)        │
  │    update_live_results(diffs)                    │
  │  outer commit() [atomic]                         │
  │                                                  │
  │  dispatch_query_result(QueryResult { sequence }) │
  ├──── mpsc/broadcast channel ─────────────────────►│ priority queue
  │                                                  │ framework dedup (skip if seq ≤ checkpoint)
  │                                                  │ execute side-effect
  │                                                  │ write_checkpoint(query_id, seq)
  │                                                  │
  │◄── fetch_snapshot() ─────────────────────────────│ (on subscribe, optional)
  │◄── fetch_outbox(after_seq) ──────────────────────│ (on subscribe, optional)
```

## 4. Key Design Decisions

### Ring buffer outbox over position-handles
We use a fixed-size, self-pruning ring buffer (`outbox` namespace) rather than tracking per-reaction position handles. The query has no knowledge of downstream consumer state. This avoids unbounded disk growth from stalling consumers and simplifies the Query hot-path. 

### Dual bootstrap APIs (`fetch_snapshot` & `fetch_outbox`)
Reactions read missed data explicitly on start-up rather than having the query dump history into the live channel. This separates the replay path from the live data path. 

### Memory-Served Reads for Consistency
Both `fetch_snapshot` and `fetch_outbox` read strictly from an in-memory `RwLock<QueryOutputState>`, never from the persistent index during normal operation.
- **Consistency**: Guarantees readers don't see partially committed state.
- **Backend-Agnostic**: Avoids needing backend-specific read-snapshots. Storage namespaces are write-only during normal operation and read only on crash-recovery startup.

### Uniform APIs & State
The in-memory `QueryOutputState` and sequence tracking run for all queries. The only thing conditional on the index backend is the actual `write` to the persistent storage. This unifies the code paths, gives volatile queries `O(1)` diff application performance, and also opens the possibility for volatile reactions to detect broadcast lag via sequence numbers.

## 5. Delivery Guarantees & Compatibility Matrix

| Query Index | Reaction Type | Allowed | Guarantee |
|-------------|---------------|---------|-----------|
| Volatile (Memory) | Volatile | ✅ | At-Most-Once |
| Volatile (Memory) | Durable | ❌ Error | Rejected at startup - no outbox exists for replay |
| Persistent (RocksDB/Garnet) | Volatile | ✅ | At-Least-Once delivery, At-Most-Once processing |
| Persistent (RocksDB/Garnet) | Durable | ✅ | At-Least-Once delivery and processing |

## 6. Recovery Flows

- **New Volatile Reaction**: `fetch_snapshot()` → apply state → open live gate.
- **New Durable Reaction**: `fetch_snapshot()` → apply state → save checkpoint → open live gate.
- **Durable Reaction Restart (No Gap)**: read state checkpoint → `fetch_outbox(checkpoint)` → process entries → open live gate.
- **Durable Reaction Restart (Gap)**: Reaction detects checkpoint is older than the oldest outbox entry. Executes its `recovery_policy` (`strict`, `auto_reset`, or `auto_skip_gap`).
