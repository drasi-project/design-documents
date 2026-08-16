# Component Lifecycle in DrasiLib — current implementation

* Project Drasi - 2026-08-15 - Allen Jones
* Draft 19 (owner-relative cardinality)
* Code: `drasi-core/lib/` (graph under `component_graph/`)

## Components

There are currently nine kinds of Components defined in drasi-lib. The following table describes them:

| Component Kind | Description |
|---|---|
| Bootstrap provider | Supplies initial/historical snapshot data so a query can build starting state before live changes. |
| Identity provider | Resolves credentials and identity for components that talk to external systems. |
| Index backend | Provides the storage implementation queries use for intermediate and result indexes. |
| Query | Continuously evaluates a continuous query over one or more sources and produces result updates. |
| Reaction | Subscribes to query results and acts on them (notify, write outward, invoke other systems). |
| Secret store | Resolves secret references in configuration before components are constructed. |
| Source | Ingests change data from an external system and makes it available to queries. |
| State store | Durable key-value style state for components that need to persist progress or checkpoints. |
| WAL provider | Write-ahead log storage for durable source change streams and replay. |

These Component Kinds vary in terms of:
- what trait defines them
- what creates them
- what they are at runtime
- their cardinality
- their lifecycle

| Component kind | Trait | Created By | Cardinality | Runtime representation | Lifecycle |
|---|---|---|---|---|---|
| Bootstrap provider | `BootstrapProvider` trait | Host | 0 or 1 per Source | Passive instance owned by the source; optional metadata-only graph node; no instance in `runtimes` | No independent lifecycle; invoked as part of source/query bootstrap |
| Identity provider | `IdentityProvider` trait | Host (optional) | 0 or 1 per DrasiLib | Shared singleton passed to sources and reactions; metadata-only graph node when configured | No independent lifecycle; node remains `Added` |
| Index backend | `IndexBackendPlugin` trait | DrasiLib default or Host | 0 or more per DrasiLib; 0 or 1 is the default | Shared named provider plus index instances owned by individual queries; no graph node | No graph lifecycle; provider lives with DrasiLib and indexes are created during query provisioning |
| Query | `Query` trait | DrasiLib | 0 or more per DrasiLib | Named active instance stored in `runtimes` with a graph node and source dependencies | Full graph lifecycle: add, start, run, stop, reconfigure, error, remove |
| Reaction | `Reaction` trait | Host | 0 or more per DrasiLib | Named active instance stored in `runtimes` with a graph node and query dependencies | Full graph lifecycle: add, start, run, stop, reconfigure, error, remove |
| Secret store | `SecretStoreProvider` trait | Host (optional) | 0 or 1 per DrasiLib | Shared singleton used while resolving configuration; no graph node | No graph lifecycle; lifetime follows the DrasiLib instance |
| Source | `Source` trait | Host | 0 or more per DrasiLib | Named active instance stored in `runtimes` with a graph node | Full graph lifecycle: add, start, run, stop, reconfigure, error, remove |
| State store | `StateStoreProvider` trait | Host or DrasiLib default | 1 per DrasiLib | Shared singleton passed to sources and reactions; no graph node | No graph lifecycle; lifetime follows the DrasiLib instance |
| WAL provider | `WalProvider` trait | Host | 0 or 1 per DrasiLib | Shared singleton used by sources, with per-source registration; no graph node | No graph lifecycle; lifetime follows the DrasiLib instance |

## ComponentGraph

The `ComponentGraph` is the shared registry for a `DrasiLib` instance. It is intended to be the source of truth
for which components exist, how they relate, and what state they are in. Beyond the registry it also provides:

- **Dependency tracking** — edges, plus `get_dependents` / `can_remove`
- **Status validation** — every node carries a `ComponentStatus`; transitions are checked
- **Events** — `broadcast::Sender` fans out `ComponentEvent`s on add, remove, status change
- **Status ingestion** — components report over a shared `mpsc`, drained by an external
  update loop calling `apply_update`. Managers mutate commanded transitions directly under
  the write lock. These are the two write paths.
- **Ordering** — `topological_order` for lifecycle operations
- **Event history** — recent events and last error, queryable per component

### Nodes

ComponentGraph Nodes have the following structure:

```text
ComponentNode {
  id: String,
  kind: ComponentKind,
  status: ComponentStatus,
  metadata: HashMap<String, String>,   // opaque to the graph
}
```

Node `kind` is stored as the `ComponentKind` enum:

| Value | Description |
|---|---|
| `Instance` | Graph root representing the DrasiLib instance itself. Created in `ComponentGraph::new`. Status fixed at `Running`. Cannot be removed. Not a component instance. |
| `Source` | A source component node. |
| `Query` | A query component node. |
| `Reaction` | A reaction component node. |
| `BootstrapProvider` | A bootstrap provider node (topology; no instance in `runtimes` today). |
| `IdentityProvider` | An identity provider node (topology; no instance in `runtimes` today). |

`Instance` is the ComponentGraph root node and exist only for the purpose of having a central Node for ComponentNodes to be connected to.
**Note:** Index backend, secret store, state store, and WAL have no `ComponentKind` variant and never appear as nodes.

ComponentNode `status` is stored as the `ComponentStatus` enum:

| Value | Description |
|---|---|
| `Added` | Registered on the graph, not yet started. Set when the node is created. |
| `Starting` | Start has been commanded or is in progress. |
| `Running` | Operating normally (for sources/queries/reactions that use the full machine). |
| `Stopping` | Stop has been commanded or is in progress. |
| `Stopped` | Stopped; may be started again. |
| `Reconfiguring` | Being updated / replaced. |
| `Error` | Failed; message may be in event history. Recovery is not automatic. |
| `Removed` | Terminal structural state used when emitting the remove event; the node is gone after `remove_component`. |

Allowed transitions and the two write paths are under **Status** below.

### Metadata

Each node has `metadata: HashMap<String, String>`. The graph does not interpret these keys;
it only stores and returns them (including in `snapshot()`).

The keys below are entries in that map, written by DrasiLib (or the builder) when
registering the node, before/at `register_*`:

| Key | Set for | Meaning |
|---|---|---|
| `kind` | source, query, reaction, bootstrap, identity | Component **type** hint as a string (e.g. `"http"`, `"identity_provider"`), not `ComponentKind`. |
| `autoStart` | source, query, reaction | Whether the instance should auto-start (`"true"` / `"false"`). |
| `query` | query | Query text from `QueryConfig`. |

Callers may pass additional pairs (e.g. Drasi Server’s `pluginId`, `pluginVersion`) via
`extra_metadata`; those are merged into the same map. Bootstrap registration on the builder
can also copy config property keys into metadata as JSON strings.

### Relationships

Relationships are the connections between nodes. They are typed by the `RelationshipKind`
enum (the schema). Each logical link is stored as a **bidirectional pair** of edges: a
forward kind and its reverse (`RelationshipKind::reverse()`). Adding a pair that already
exists is a no-op. Edges carry no status and have no lifecycle of their own.

| Forward | Reverse | Meaning | Allowed endpoints (`is_valid_relationship`) |
|---|---|---|---|
| `Owns` | `OwnedBy` | Graph root owns this component node | `Instance` → any |
| `Feeds` | `SubscribesTo` | Data flows producer → consumer | `Source` → `Query`, `Query` → `Reaction` |
| `Bootstraps` | `BootstrappedBy` | Bootstrap provider serves a source | `BootstrapProvider` → `Source` |
| `Authenticates` | `AuthenticatedBy` | Identity provider authenticates a component | `IdentityProvider` → any |

Validation checks the forward kind against endpoint kinds when edges are added. Reverse
edges are always the mate of the forward kind.

## Component lifecycle

Creation, graph registration, and start are separate operations. For sources, queries, and
reactions, DrasiLib first registers the node, then initializes and stores the component
instance. Starting happens later, either explicitly or because `autoStart` is enabled.

### Creation and registration

| Component kind | Where the instance is created | How it is represented in `ComponentGraph` |
|---|---|---|
| Source | Outside DrasiLib, normally by the host from a source plugin; passed through the builder or `add_source` | `register_source` creates a node. DrasiLib then calls `initialize` and stores the instance in `runtimes`. |
| Query | Inside DrasiLib from `QueryConfig` | `register_query` first verifies that referenced sources exist and creates `Feeds` relationships. DrasiLib then constructs, initializes, and stores the query instance. |
| Reaction | Outside DrasiLib, normally by the host from a reaction plugin; passed through the builder or `add_reaction` | `register_reaction` first verifies that referenced queries exist and creates `Feeds` relationships. DrasiLib then calls `initialize` and stores the instance in `runtimes`. |
| Bootstrap provider | Outside DrasiLib and attached to a source | The instance is not stored in the graph. The builder can create a separate metadata-only node with `register_bootstrap_provider`. |
| Identity provider | Outside DrasiLib and supplied to the builder | If configured, the builder creates one node named `identity-provider` and connects it to sources and reactions present at build time. The instance is not stored in the graph; it is passed to sources and reactions in their contexts. |
| Index backend | Named plugin providers are created outside DrasiLib and supplied to the builder. DrasiLib creates in-memory indexes internally when configured. | No node. |
| Secret store | Outside DrasiLib and supplied to the builder | No node. |
| State store | Outside DrasiLib and supplied to the builder, or DrasiLib creates the default in-memory provider | No node. |
| WAL provider | Outside DrasiLib and supplied to the builder | No node. |

Registration of a source, query, or reaction creates the node at `Added` before its instance
is initialized or stored. If initialization or storage fails, the caller removes the node as
a compensating action. `GraphTransaction` does not include instance initialization or the
`runtimes` map.

### State machine

Only sources, queries, and reactions use the lifecycle state machine.

| From | Allowed next states |
|---|---|
| `Added` | `Starting`, `Stopped`, `Reconfiguring` |
| `Starting` | `Running`, `Stopping`, `Stopped`, `Error` |
| `Running` | `Stopping`, `Stopped`, `Reconfiguring`, `Error` |
| `Stopping` | `Stopped`, `Error` |
| `Stopped` | `Starting`, `Reconfiguring` |
| `Reconfiguring` | `Starting`, `Stopped`, `Error` |
| `Error` | `Starting`, `Stopped`, `Reconfiguring` |

`Added` and `Removed` are structural states set when nodes are added and removed; they are
not transition targets.

| Component kind | Lifecycle behavior |
|---|---|
| Source, Query, Reaction | Full state machine. They can be started, stopped, reconfigured, and removed while the DrasiLib instance is running. |
| Bootstrap provider, Identity provider | Their nodes remain at `Added`. They have no graph-managed start, stop, reconfigure, or remove lifecycle. |
| Index backend, Secret store, State store, WAL provider | No node and no `ComponentStatus`. Their lifetime is controlled by the DrasiLib instance and its builder. |
| Graph root (`ComponentKind::Instance`) | Created at `Running` and never transitions. It is not a component instance. |

Lifecycle commands and outcomes come from different places:

- DrasiLib managers initiate `Starting`, `Stopping`, and `Reconfiguring` through
  `validate_and_transition`.
- Adding and removing nodes produces `Added` and `Removed`.
- The component instance reports `Running`, `Stopped`, and operational `Error` outcomes.
- Managers also set `Error` when a start, stop, or reconfiguration call returns an error.

For sources and reactions, the implementation is responsible for reporting successful
`Running` and `Stopped` outcomes. Queries are implemented inside DrasiLib and report these
outcomes themselves. Returning successfully from `start()` does not itself change the graph
to `Running`; the component must report that state.

### Status reporting

During `initialize`, a source, query, or reaction receives a context containing the graph's
component-update sender. Its `ComponentStatusHandle` is wired to that sender.

When `ComponentStatusHandle::set_status` is called:

1. it updates the component instance's local status;
2. it notifies local `watch` subscribers;
3. it sends a `ComponentUpdate::Status` to the graph update channel;
4. the graph update task applies the update under the graph write lock;
5. the graph updates the node, emits a `ComponentEvent`, records event history, and wakes
   status waiters.

The channel path validates the transition again. An invalid reported transition is logged
and ignored. Manager-initiated transitions use the same transition rules but return an
error to the caller when invalid.

---

## Problems and beneficial changes

| Problem | Consequence | Beneficial change |
|---|---|---|
| A node does not have one consistent meaning. Source/query/reaction nodes represent stored instances; bootstrap/identity nodes are metadata-only; `Instance` is the graph root. | Code cannot treat all nodes uniformly as component instances. | Define separate node roles, or restrict component nodes to actual component instances. Rename `ComponentKind::Instance` to `Root` or `DrasiLib`. |
| The graph is not the source of truth for all nine component kinds. Four provider kinds are absent, and their dependencies are not represented. | `snapshot`, dependency inspection, removal checks, and clone/export do not describe the complete DrasiLib instance. | Either include providers and their dependencies, or narrow the graph's stated responsibility and add a separate component/provider registry. |
| Bootstrap and identity nodes do not track their actual instances. Identity relationships are created only for components present during build. | The graph can contain incomplete or stale topology. | Choose one canonical model: manage these instances in the graph, or remove their shadow nodes. Keep relationships updated for live add/remove. |
| Lifecycle state exists in both the graph node and `ComponentStatusHandle`. Managers update the graph directly; components update local state and report asynchronously. | The two values can diverge. Invalid reported updates are logged and ignored. | Make one state store authoritative and route every transition through one lifecycle controller. Components should report outcomes/health, not independently own lifecycle state. |
| One `ComponentStatus` machine is attached to nodes that do not participate in it. Bootstrap and identity remain at `Added`; the root remains at `Running`. | `status` has different meaning depending on node kind. | Make lifecycle participation explicit. Do not attach lifecycle state to topology-only nodes, or implement their lifecycle. |
| `Error` covers configuration errors, connection failures, dependency loss, and operational failure. | Callers cannot determine whether creation failed, whether the component is usable but disconnected, or whether retry is appropriate. | Add structured failure reason, phase, and terminality. Separate configuration failure from transient/degraded operation. |
| Registration and instance creation are not one transaction. `GraphTransaction` covers nodes and edges, but not initialization or the `runtimes` map. | Failed creation requires compensating removal; multi-component load cannot provide a clear atomic success boundary. | Stage nodes, relationships, validated configuration, and initialized instances together; publish them only after the batch succeeds. Keep activation/start outside that creation boundary. |
| Component type, plugin identity, configuration, and `autoStart` are stored as optional string metadata or only on the live instance. | `snapshot()` is not sufficient for reliable clone, solution export, or reload. | Define structured fields for component type, plugin id/version, configuration, and lifecycle policy; make their presence part of registration validation. |
| `runtimes` is a map of component instances, not execution runtimes. | The name obscures the type/instance model and is easily confused with the process or Tokio runtime. | Rename it to `instances` or `component_instances`, with corresponding `set/get/take/has_instance` methods. |

---

*End of draft 19.*
