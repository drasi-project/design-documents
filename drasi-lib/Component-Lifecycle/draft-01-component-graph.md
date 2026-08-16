# Component Graph — current implementation

* Project Drasi - 2026-08-15 - _(author)_
* Draft 01 (supersedes draft 00 for this topic)
* Code: `drasi-core/lib/src/component_graph/`

## Purpose

`ComponentGraph` is the shared registry for a `DrasiLib` instance. The three managers
(`SourceManager`, `QueryManager`, `ReactionManager`) share one
`Arc<RwLock<ComponentGraph>>` and do not keep separate component registries.

It records:

1. which components exist
2. how they relate
3. each component's `ComponentStatus`
4. the live runtime object for components that have one
5. recent lifecycle events

Source: module docs in `component_graph/mod.rs` and `graph.rs`.

## Layout

| File | Role |
|---|---|
| `mod.rs` | module docs, re-exports |
| `node.rs` | `ComponentNode`, `ComponentKind`, `RelationshipKind`, `ComponentUpdate`, `ComponentStatusHandle`, `GraphSnapshot` |
| `graph.rs` | `ComponentGraph` |
| `transaction.rs` | `GraphTransaction` |
| `wait.rs` | `wait_for_status` |

## Data held by `ComponentGraph`

| Field | Contents |
|---|---|
| `graph` | `petgraph::StableGraph<ComponentNode, RelationshipKind>` |
| `index` | `component_id → NodeIndex` |
| `instance_idx` | root node index |
| `runtimes` | `component_id → Box<dyn Any + Send + Sync>` |
| `event_tx` | `broadcast::Sender<ComponentEvent>` (capacity 1000) |
| `update_tx` | `mpsc::Sender<ComponentUpdate>` (capacity 1000) |
| `status_notify` | `Notify` woken on any status change |
| `event_history` | per-component recent events (up to 100 each) |

`ComponentGraph` itself is not `Send`/`Sync`. Callers always hold
`Arc<RwLock<ComponentGraph>>`.

`new(instance_id)` returns `(ComponentGraph, ComponentUpdateReceiver)`. The receiver
must be drained by an external update loop that calls `apply_update`.

## Nodes

```text
ComponentNode {
  id: String,
  kind: ComponentKind,
  status: ComponentStatus,
  metadata: HashMap<String, String>,   // opaque to the graph
}
```

`ComponentKind`:

| Kind | Role in the graph |
|---|---|
| `Instance` | Root node. Created in `new()`. Status fixed at `Running`. Cannot be removed. Emits no `ComponentEvent` (`to_component_type()` → `None`). Excluded from `topological_order`. |
| `Source` | Registered component |
| `Query` | Registered component |
| `Reaction` | Registered component |
| `BootstrapProvider` | Registered component |
| `IdentityProvider` | Registered component |

There is no plugin kind. Plugin identity, if recorded at all, is host-supplied metadata
on a component node (Drasi Server writes `pluginId` / `pluginVersion`).

`add_component` rejects duplicate IDs. Every non-root node automatically gets
`Instance --Owns--> component` and `component --OwnedBy--> Instance`.

### Metadata

The graph does not interpret metadata. Callers set it at registration.

DrasiLib registration paths currently set:

| Key | Set for |
|---|---|
| `kind` | source, query, reaction, bootstrap, identity |
| `autoStart` | source, query, reaction |
| `query` | query (query text) |

Anything else is host-defined and carried through `snapshot()`.

## Relationships

Edges are always written as a bidirectional pair. `RelationshipKind::reverse()` defines
the mate. Adding an existing pair is a no-op.

| Forward | Reverse | Intended endpoints | Validated by `is_valid_relationship` |
|---|---|---|---|
| `Owns` | `OwnedBy` | Instance → any | Instance → any with `Owns` |
| `Feeds` | `SubscribesTo` | Source → Query, Query → Reaction | only those two |
| `Bootstraps` | `BootstrappedBy` | BootstrapProvider → Source | only that |
| `Authenticates` | `AuthenticatedBy` | IdentityProvider → any | IdentityProvider → any with `Authenticates` |

Query helpers that matter:

| Method | Follows |
|---|---|
| `get_neighbors(id, kind)` | outgoing edges of that kind |
| `get_dependencies(id)` | outgoing `SubscribesTo` |
| `get_dependents(id)` | outgoing `Feeds` |
| `can_remove(id)` | errors if `get_dependents` is non-empty |
| `topological_order()` | toposort over **only** `Feeds` edges; excludes Instance |

`can_remove` / `deregister` only consider data-flow dependents (`Feeds`). Bootstrap and
auth edges do not block removal.

## Runtime store

Separate from the petgraph node data:

```text
set_runtime(id, Box<dyn Any + Send + Sync>)  // node must already exist
get_runtime::<T>(id) -> Option<&T>
take_runtime::<T>(id) -> Option<T>
has_runtime(id) -> bool
```

`set_runtime` fails if the node is missing. In debug builds it warns if the stored type
is not `Arc<dyn Source|Query|Reaction>` for those kinds.

`remove_component` also removes any runtime for that id.

The graph does not construct runtimes. It only stores what callers put there.
`GraphTransaction` does not cover the runtime map.

## Status

Every node has a `ComponentStatus`:

`Added` · `Starting` · `Running` · `Stopping` · `Stopped` · `Reconfiguring` · `Error` · `Removed`

### Allowed transitions (`is_valid_transition`)

```text
Added         → Starting | Stopped | Reconfiguring
Stopped       → Starting | Reconfiguring
Starting      → Running | Error | Stopped | Stopping
Running       → Stopping | Stopped | Error | Reconfiguring
Stopping      → Stopped | Error
Error         → Starting | Stopped | Reconfiguring
Reconfiguring → Stopped | Starting | Error
```

`Added` and `Removed` are structural. They are written by add/remove, not by
`validate_and_transition`.

Same-status updates are no-ops (no event).

### Two write paths

| Path | Used for | Invalid transition behavior |
|---|---|---|
| Manager holds write lock → `validate_and_transition` | commanded moves (`Starting`, `Stopping`, `Reconfiguring`) | **returns `Err`** |
| Component → `mpsc::ComponentUpdate` → update loop → `apply_update` | runtime reports (`Running`, `Stopped`, `Error`) | **warns and ignores** |

Components do not take the graph lock to report status. They use
`ComponentStatusHandle`, which updates a local `Arc<RwLock<ComponentStatus>>`, a local
`watch` channel, and (if wired) the graph mpsc sender.

`wait_for_status` waits on `status_notify` with a timeout.

## Registration helpers

These are the typed entry points managers use. Each starts a transaction, adds the node
at status `Added`, adds edges, commits (events emit on commit).

| Method | Creates | Also creates | Pre-checks |
|---|---|---|---|
| `register_source(id, metadata)` | Source node | ownership edges | id free |
| `register_query(id, metadata, source_ids)` | Query node | ownership + `Feeds` from each source | id free; every source exists |
| `register_reaction(id, metadata, query_ids)` | Reaction node | ownership + `Feeds` from each query | id free; every query exists |
| `register_bootstrap_provider(id, metadata, source_ids)` | BootstrapProvider node | ownership + `Bootstraps` to each source | id free; every source exists |
| `register_identity_provider(id, metadata, component_ids)` | IdentityProvider node | ownership + `Authenticates` to each target | id free; every target exists |
| `deregister(id)` | — | removes node, edges, runtime, history | `can_remove`; not Instance |

Registration does **not** store a runtime. Callers that need one call `set_runtime` later.
That is the "registry-first" rule: node before runtime storage.

## Transactions

`begin()` → `GraphTransaction`:

- `add_component(node)`
- `add_relationship(from, to, forward)`
- `commit()` — emits deferred add events and records history

Drop without commit removes added nodes and edges. Runtimes are outside the transaction.
If runtime setup fails after commit, callers must compensate with `deregister` /
`remove_component`.

## Events

On add, remove, and successful status change the graph:

1. broadcasts a `ComponentEvent` on `event_tx`
2. records it in `event_history` (except Instance, which has no event type)

| API | Behavior |
|---|---|
| `subscribe()` | broadcast receiver for all graph events |
| `get_events(id)` / `get_all_events()` | history queries |
| `get_last_error(id)` | last error message in history |
| `subscribe_events(id)` | history snapshot + per-component live receiver |

## Snapshot

`snapshot()` → `GraphSnapshot { instance_id, nodes, edges }`.

Includes every node (with status and metadata) and every edge. Excludes runtimes,
event history, and any config objects not stored on the node.

## Public surface (grouped)

| Group | Methods |
|---|---|
| Construct / wire | `new`, `instance_id`, `subscribe`, `event_sender`, `update_sender`, `status_notifier`, `apply_update` |
| Nodes | `add_component`, `remove_component`, `get_component`, `get_component_mut`, `contains`, `list_by_kind`, `node_count` |
| Runtimes | `set_runtime`, `get_runtime`, `take_runtime`, `has_runtime` |
| Edges | `add_relationship`, `remove_relationship`, `get_neighbors`, `get_dependencies`, `get_dependents`, `edge_count` |
| Lifecycle | `validate_and_transition`, `can_remove`, `topological_order` |
| Typed register | `register_source`, `register_query`, `register_reaction`, `register_bootstrap_provider`, `register_identity_provider`, `deregister` |
| Transaction | `begin` |
| History | `record_event`, `get_events`, `get_all_events`, `get_last_error`, `subscribe_events` |
| Export | `snapshot` |
| Wait (free function) | `wait_for_status` |

## What the graph does not model

These exist in DrasiLib but are not graph nodes and have no edges here:

- index backend providers
- state store provider
- WAL provider
- secret store provider
- plugins / plugin descriptors

Bootstrap and identity providers *can* be nodes, but the graph does not drive their
lifecycle: no runtime is stored for them by current callers, and nothing transitions
their status after `Added`.
