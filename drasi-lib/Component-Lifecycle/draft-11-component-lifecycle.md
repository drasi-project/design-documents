# Component Lifecycle in DrasiLib — current implementation

* Project Drasi - 2026-08-15 - _(author)_
* Draft 11 (resolved inline edit requests from draft 10)
* Code: `drasi-core/lib/` (graph under `component_graph/`)

## Components

There are currently nine kinds of Components in drasi-lib. The following table describes them:

| Component kind | Description |
|---|---|
| Source | Ingests change data from an external system and makes it available to queries. |
| Query | Continuously evaluates a continuous query over one or more sources and produces result updates. |
| Reaction | Subscribes to query results and acts on them (notify, write outward, invoke other systems). |
| Bootstrap provider | Supplies initial/historical snapshot data so a query can build starting state before live changes. |
| Identity provider | Resolves credentials and identity for components that talk to external systems. |
| Index backend | Provides the storage implementation queries use for intermediate and result indexes. |
| Secret store | Resolves secret references in configuration before components are constructed. |
| State store | Durable key-value style state for components that need to persist progress or checkpoints. |
| WAL provider | Write-ahead log storage for durable source change streams and replay. |

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


---

## Changes to make

Actionable follow-ups from this review. Not a design of the end state — a backlog of
concrete fixes and renames. Ordered roughly by cost vs confusion removed.

### Naming / API clarity

1. **Rename `ComponentGraph.runtimes` → `instances` (or `component_instances`).**  
   The field holds component instances (`Arc<dyn Source|Query|Reaction>`). The comment at
   introduction already said "runtime instances"; the shortened field name `runtimes` collides
   with process/Tokio "runtime" and fights the type/instance vocabulary.  
   Rename with it: `set_runtime` → `set_instance`, `get_runtime` → `get_instance`,
   `take_runtime` → `take_instance`, `has_runtime` → `has_instance`.  
   Introduced in `de6064a7` (Runtime plugins #349, 2026-04-14).

2. **Rename `ComponentKind::Instance` (graph root).**  
   It is the DrasiLib instance as a node, not a component instance. Prefer something like
   `DrasiLib` / `Root` / `Container` so it does not collide with "component instance."

3. **Stop using "runtime" in public docs/comments for component instances.**  
   Prefer *component instance*, *component type*, *interface*, *kind* as defined under Components / kind–type–instance vocabulary.
   Keep "runtime" only where it means execution environment or an existing type name that
   cannot move yet (e.g. `QueryRuntimeContext` — separate rename pass).

### Graph model honesty

4. **Decide the role of topology-only nodes (bootstrap, identity).**  
   Today they look like component instances on the graph but have no `runtimes` entry, no
   status transitions, and (for identity) stale edges. Either:
   - make them real lifecycle-managed or at least consistently updated instances, or
   - stop presenting them as peer nodes and record them only as metadata/edges on the
     components that use them.

5. **Align bootstrap with Drasi Server.**  
   Server: top-level named bootstrap config. Lib: attached to source, optional shadow node.
   Pick one model and implement it on both sides.

6. **Keep identity edges current — or drop them.**  
   `Authenticates` is drawn once at build; live-added sources/reactions never get edges.
   Either maintain edges on add/remove, or remove unused topology until lifecycle uses it.

7. **Put provider dependencies in the dependency story.**  
   Index backend, state store, WAL (and secret store at resolve time) are required for
   correct operation but are invisible to `get_dependencies` / `can_remove` / `snapshot`
   topology. At minimum document the gap in API terms; longer term consider explicit links
   or a non-graph registry the control plane can query.

### Lifecycle / status

8. **Split or qualify `Error`.**  
   Misconfiguration vs disconnect vs dependency loss are different outcomes; today they share
   one state and a free-text message. Matters for terminal vs non-terminal reporting and for
   auto-recovery policy.

9. **Define recovery policy for `Error`.**  
   `Error → Starting` is legal but never automatic. Reactions detect subscription loss only.
   Decide what leaves `Error` and what remains operator-driven.

10. **Unify invalid-transition handling.**  
    `validate_and_transition` returns `Err`; `apply_update` warns and ignores. Same state
    machine should not have two silent/loud policies without an explicit reason.

11. **Revisit "query start failure is non-fatal, source/reaction is fatal" at DrasiLib
    instance start.**  
    Today severity is by interface kind, not by failure cause. Align with whatever
    admit/activate success means for multi-component load (config OK vs external connect OK).

12. **Validate what can be validated at register/create, not only at start.**  
    Query compile-at-start and reaction durable-store checks mean "created" ≠ "known valid."
    Move static checks earlier where possible; name the residual class of start-only failures.

### Transactions / multi-component load

13. **Extend transactional semantics past graph nodes/edges.**  
    `GraphTransaction` does not cover the instances map or initialize/start. Multi-component
    load (solutions, clone) needs a defined success boundary and compensating actions —
    registry-first alone is not atomic deploy.

14. **Include component instances in any export/clone story deliberately.**  
    `snapshot()` is topology only (nodes, edges, metadata). Config for host-built instances
    still depends on `properties()` of live instances. Define what clone/solution capture
    must include (types, plugin ids, configs, autoStart) vs graph snapshot alone.

### Extension / types

15. **Represent component type identity more than `metadata.kind`.**  
    Graph has instances and kinds; component types and plugins are not nodes. For export,
    reload, and "same type" reasoning, define where type id + plugin id/version live
    (structured fields vs opaque metadata) and require them for descriptor-backed instances.

---

*End of draft 06.*
