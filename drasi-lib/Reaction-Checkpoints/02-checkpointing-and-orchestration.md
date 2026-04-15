# Reaction Checkpointing and Orchestration

> **Parent**: [Reaction Recovery Overview](./00-reaction-recovery-overview.md)

---

## 1. Reaction Checkpoints

Because reaction components can multiplex over multiple queries tracking distinct sequences, checkpoints are tracked via a per-query map:

```rust
type ReactionCheckpoints = HashMap<String, u64>;  // query_id → last_processed_seq
```

Checkpoint storage will be opt-in via `ReactionBase` helpers (`read_checkpoints`, `write_checkpoint`, `clear_checkpoints`).

## 2. Framework-Level Dedup

The `ReactionBase` orchestrator can automatically deduplicates results prior to invoking plugin code:

```rust
let result = priority_queue.dequeue().await;
if let Some(checkpoint) = checkpoints.get(&result.query_id) {
    if result.sequence <= *checkpoint {
        continue; // Already processed
    }
}
// Pass to reaction processing
```

## 3. Reaction Trait Extensions

Reactions expose their state management properties via traits and config:

```rust
fn is_durable(&self) -> bool { false } // Declares if reaction persists checkpoints.
```

**Recovery Policies**: Configured by the reaction user to handle sequence gaps.
- `Strict`: Fail startup. Manual intervention required.
- `AutoReset`: Wipe reaction state, `fetch_snapshot()`, rebuild from scratch.
- `AutoSkipGap`: Note gap in logs, continue from oldest available outbox entry.

## 4. The Bootstrap Gate

A race condition exists between capturing initial query state (Snapshot/Outbox API) and polling the live channel where new events continuously buffer.

**Solution**: Handled by an `Arc<Notify>` (`bootstrap_gate`):
1. `ReactionManager::start_reaction` spawns the reaction's processing loop, which immediately `await`s the `bootstrap_gate` before dequeuing.
2. The manager connects live subscriptions (events queue up).
3. Manager performs per-query API calls (`fetch_snapshot` or `fetch_outbox`).
4. Manager sets the initial in-memory checkpoints based on API results.
5. `bootstrap_gate.notify_one()` fires.
6. Processing loop begins dequeuing buffered live events, deduplication filters overlapping sequences naturally.

## 5. Startup / Subscribe Flowchart

During startup, `ReactionManager` validates compatibility (error if `is_durable=true` and `is_volatile=true`). Then, for each query:

1. Check State Store for `last_processed_seq`.
2. **If None**: 
   - Execute `fetch_snapshot()`. Set checkpoint to snapshot `as_of_sequence`.
3. **If Sequence Found(=N)**:
   - Execute `fetch_outbox(after: N)`.
   - If oldest returned sequence ≤ N: Resume from outbox seamlessly.
   - If oldest returned sequence > N (Gap): Execute configuring `RecoveryPolicy`.

## 6. Broadcast Mode: Runtime Gaps

Using `broadcast` dispatch instead of `channel` guarantees non-blocking queries but risks buffer overflows under backpressure (`RecvError::Lagged`).

Sequence tracking allows Reactions to natively detect these dropped events:
```rust
if incoming_seq > last_processed_seq + 1 {
    on_gap_detected(query_id, last_processed_seq + 1, incoming_seq - 1);
}
```
Reaction plugins implement `on_gap_detected` dynamically. A view-heavy plugin will `fetch_snapshot()` and rebuild inline, whereas a fire-and-forget logging plugin may simply warn and continue. 

## 7. Query Lifecycle Protections

- **Update/Delete Query**: Strictly blocked if reactions are currently subscribed. Reconfiguring a query resets its internal DB/hash sequences to `0`. A lingering reaction checkpoint of `500` would incorrectly ignore all subsequent events. Users must consciously delete reactions before structural query updates.
- **Stop Query**: Safely auto-stops subscribed reactions. Restarts require manual reaction starts. Sequence spaces are preserved.
