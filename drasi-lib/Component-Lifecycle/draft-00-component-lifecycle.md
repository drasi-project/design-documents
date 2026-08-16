# Component Lifecycle in DrasiLib

* Project Drasi - 2026-08-14 - _(author)_

A component is an instance of one of these nine interfaces. This is the complete set.

| Interface | Defined in | Dynamically loadable via |
|---|---|---|
| `Source` | `lib/src/sources/traits.rs` | `SourcePluginDescriptor` |
| `Query` | `lib/src/queries/manager.rs` | — built inside DrasiLib |
| `Reaction` | `lib/src/reactions/traits.rs` | `ReactionPluginDescriptor` |
| `BootstrapProvider` | `lib/src/bootstrap/mod.rs` | `BootstrapPluginDescriptor` |
| `IdentityProvider` | `lib/src/identity/mod.rs` | `IdentityProviderPluginDescriptor` |
| `IndexBackendPlugin` | `core/src/interface/index_backend.rs` | `IndexBackendPluginDescriptor` |
| `SecretStoreProvider` | `lib/src/secret_store/mod.rs` | `SecretStorePluginDescriptor` |
| `StateStoreProvider` | `lib/src/state_store/mod.rs` | — static link only |
| `WalProvider` | `lib/src/wal/traits.rs` | — static link only |

## The component graph

`ComponentGraph` is DrasiLib's single source of truth for which components exist, how they
relate, and what state they are in. All three managers (`SourceManager`, `QueryManager`,
`ReactionManager`) share one `Arc<RwLock<ComponentGraph>>` and keep no registry of their own.

The graph stores two things per component: the node, and the live object. Registration is
**registry-first** — the node must exist before the runtime is stored against it, so
`provision_source` requires the caller to have called `register_source` first. This is
ordering inside the graph, not construction order: sources and reactions arrive already
constructed. Only `Source`, `Query` and `Reaction` ever get a runtime stored.
`BootstrapProvider` and `IdentityProvider` get a node and nothing else.

Beyond the registry it also owns:

- **Dependency tracking** — edges, and the `get_dependents`/`can_remove` checks over them.
- **Status** — every node carries a `ComponentStatus`; the graph validates each transition.
- **Events** — a `broadcast::Sender` fans out `ComponentEvent`s on add, remove, status change.
- **Status ingestion** — components report fire-and-forget over a shared `mpsc`, drained by an
  external update loop calling `apply_update`. Managers instead mutate directly under the
  write lock. These are the two write paths.
- **Ordering** — `topological_order` yields the sequence used for lifecycle operations.
- **Event history** — recent events and last error, queryable per component.

### Nodes and relationships

Six node kinds: `Instance` (root), `Source`, `Query`, `Reaction`, `BootstrapProvider`,
`IdentityProvider`. A node is `id`, `kind`, `status`, and a `metadata` string map. DrasiLib
sets `kind` and `autoStart` (plus `query` for queries) and treats the rest as opaque; the
host merges in whatever else it wants.

Plugins are not in the graph — there is no plugin node and no edge to one. A component's link
back to the plugin that created it is metadata only: Drasi Server writes `pluginId` and
`pluginVersion` into that map. So the association cannot be traversed, and statically-linked
components carry no plugin identity at all.

Four relationship kinds, each stored as a bidirectional pair. Edges are stateless and have no
lifecycle of their own.

| Pair | Between |
|---|---|
| `Owns` / `OwnedBy` | Instance → component |
| `Feeds` / `SubscribesTo` | Source → Query, Query → Reaction |
| `Bootstraps` / `BootstrappedBy` | BootstrapProvider → Source |
| `Authenticates` / `AuthenticatedBy` | IdentityProvider → Source/Reaction |

### API surface

| Group | Methods |
|---|---|
| Wiring | `new` · `subscribe` · `event_sender` · `update_sender` · `status_notifier` · `apply_update` · `instance_id` |
| Nodes | `add_component` · `remove_component` · `get_component` · `get_component_mut` · `contains` · `list_by_kind` · `node_count` |
| Runtimes | `set_runtime` · `get_runtime<T>` · `take_runtime<T>` · `has_runtime` |
| Edges | `add_relationship` · `remove_relationship` · `get_neighbors` · `get_dependencies` · `get_dependents` · `edge_count` |
| Lifecycle | `validate_and_transition` · `can_remove` · `topological_order` · `wait_for_status` |
| Typed registration | `register_source` · `register_query` · `register_reaction` · `register_bootstrap_provider` · `register_identity_provider` · `deregister` |
| Transaction | `begin` → `GraphTransaction::{add_component, add_relationship, commit}` |
| History | `record_event` · `get_events` · `get_all_events` · `get_last_error` · `subscribe_events` |
| Export | `snapshot` → `GraphSnapshot { instance_id, nodes, edges }` |

`GraphTransaction` covers nodes and edges but **not** runtimes, so runtime failures need a
compensating `deregister`. `snapshot` returns topology only — no runtime, no configuration.

## Component taxonomy

Three properties determine what lifecycle a type has.

**1. On the component graph?** Five kinds are graph nodes: `Source`, `Query`, `Reaction`,
`BootstrapProvider`, `IdentityProvider`. Index backends, state stores, WAL and secret store
are injected at build time and never appear — no graph identity, no status, no events.

**2. Constructed outside and injected, or created inside?** Nearly everything is built by the
host and handed over fully formed. **`Query` is the only type DrasiLib constructs itself**,
from a `QueryConfig`. Per-query index instances are also created inside, via a host-supplied
factory.

**3. Shared singleton or independent instance?** Sources, queries and reactions are
independently addressable and removable. Providers are shared — one object serves every
component that needs it.

| Type | On graph | Constructed | Multiplicity | Has runtime |
|---|---|---|---|---|
| Source | ✅ | Outside | Independent, named | ✅ |
| Query | ✅ | **Inside**, from `QueryConfig` | Independent, named | ✅ |
| Reaction | ✅ | Outside | Independent, named | ✅ |
| Bootstrap provider | ✅ | Outside, attached to a source | Per source, not addressable | ❌ |
| Identity provider | ✅ | Outside | **Singleton**, fixed id `identity-provider` | ❌ |
| Index backend | ❌ | Outside | **Named, many** | ❌ |
| State store | ❌ | Outside | Singleton | ❌ |
| WAL | ❌ | Outside | Singleton | ❌ |
| Secret store | ❌ | Outside | Singleton | ❌ |

`Source`, `Query` and `Reaction` are the only types with a runtime, the only types with
`add`/`remove`/`update` on a running instance, and the only types that use the status model.
The other two graph kinds — `BootstrapProvider` and `IdentityProvider` — hold a node that
never changes.

`IdentityProvider` is both a graph node and an injected singleton. One node under a
hard-coded ID, with `Authenticates` edges drawn once at build time against the components
existing then. Components added later are never linked, so the edges go stale.

## Where the DrasiLib boundary sits

*Host* here means the code that creates and owns the `DrasiLib` instance and hands components
to it — in Drasi Server, the server itself. It is the counterpart to DrasiLib in the Owner
column throughout. Where this document means the plugin-loading SDK — cdylib loader, FFI
proxies, callbacks, reverse vtables — it says `drasi-host-sdk` in full, never "host".

Everything except `Query` is constructed by the host and handed over. DrasiLib never
constructs or destroys those objects — it registers, provisions, starts, stops, releases.
Disposal is the host's again. Two consequences:

- A segment DrasiLib cannot observe: before hand-over and after release. A construction
  failure never reaches DrasiLib — the host fails first.
- The construction config lives outside DrasiLib, recoverable only by asking the live object
  via `properties()`. This is why configuration capture depends on running components.

`BootstrapProvider` is attached to a source *before* hand-over, so DrasiLib only ever sees it
as an attribute of a source.

DrasiLib holds the config for a `Query` and constructs the object, so a `Query` is the only
type whose configuration DrasiLib can reproduce without consulting a live object.

## The shared state model

Every graph node carries a `ComponentStatus`, whether or not its type uses it:

`Added` · `Starting` · `Running` · `Stopping` · `Stopped` · `Error` · `Reconfiguring` · `Removed`

Transitions are enforced centrally (`component_graph/graph.rs:1232`):

```
Added         → Starting | Stopped | Reconfiguring
Stopped       → Starting | Reconfiguring
Starting      → Running | Error | Stopped | Stopping
Running       → Stopping | Stopped | Error | Reconfiguring
Stopping      → Stopped | Error
Error         → Starting | Stopped
Reconfiguring → Stopped | Starting | Error
```

- `Added` and `Removed` are not valid transition targets. They are set only by
  structural graph operations.
- `Error → Starting` is legal, so recovery is permitted — but no code path in DrasiLib
  performs it automatically. It always requires an explicit caller.
- A component can move itself to `Error` with no caller involved, via the status channel.

---

## Source

**Lifecycle: full.**

| Stage | Owner | Notes |
|-------|-------|-------|
| Construct | **Host** | Built outside DrasiLib and handed in fully formed. |
| Register | DrasiLib | Node created with status `Added`, plus ownership edges. |
| Provision | DrasiLib | `initialize()` supplies the runtime context. No external I/O. |
| Start | DrasiLib | `Starting` → `start()` → `Running`. This is where the external system is contacted. Failure sets `Error`. |
| Stop | DrasiLib | `Stopping` → `stop()` → `Stopped`. A failed stop lands in `Error` rather than sticking at `Stopping`. |
| Teardown | DrasiLib | Refuses to delete while `Stopping` or `Reconfiguring`. A `cleanup` flag selects whether the provider is deprovisioned or merely unregistered. |
| Update | DrasiLib | Runtime replaced via `Reconfiguring`. |
| Dispose | **Host** | After release, the object's disposal is outside DrasiLib. |

- `auto_start` is a trait method on the source, defaulting to `true`.
- **Subscription is not part of start.** A source accepts subscriptions without being
  started and without reaching its external system, and it waits for subscribers before
  producing. Wiring and connecting are already independent for this type.
- After all auto-start queries have subscribed, DrasiLib signals every running source
  that the initial subscription window has closed.
- **No failure detection.** DrasiLib does not monitor a running source. If its
  connection drops, the source must report that itself.
- Uses every status.

---

## Bootstrap provider

**Lifecycle: none.**

A bootstrap provider gets a graph node with status `Added` and an edge to each source it
serves. That node never transitions again.

It has no runtime object, no `initialize()`/`start()`/`stop()`, no add, remove, or
update operation, no `auto_start`, and no status of its own.

It is **owned by its source**: constructed by the host and attached before the source is
handed to DrasiLib, and its work runs inside the source's bootstrap path. Its failures
surface as source or query failures. The node exists for topology visibility only.

> **Mismatch with Drasi Server.** Drasi Server now treats bootstrap providers as
> top-level configuration components with their own IDs and reference-by-name. DrasiLib
> still models them as source attributes with no independent identity or lifecycle.
> Making them first-class in the lib would be a change, not a clarification.

---

## Query

**Lifecycle: full, and entirely internal.**

| Stage | Owner | Notes |
|-------|-------|-------|
| Construct | **DrasiLib** | Built from a `QueryConfig`. Unlike other types, no host-supplied object is handed in. |
| Register | DrasiLib | **Validates that every referenced source exists**, then creates the node and dependency edges. |
| Provision | DrasiLib | `initialize()` supplies the runtime context. |
| Start | DrasiLib | The query is compiled, subscribes to its sources, and runs bootstrap. |
| Stop / Teardown / Update | DrasiLib | As for sources. |

- `auto_start` is a field on `QueryConfig`, not a trait method — unlike sources and
  reactions.
- **Bootstrap gates `Running`.** A query stays in `Starting` for the whole of bootstrap,
  which may be long. `Starting` means "not yet caught up", not "briefly initializing".
- **The query is not compiled until start.** An invalid query is added successfully and
  fails only when started, so a created query is not yet a known-valid query.
- Recovery from an unavailable source position is a single in-line retry governed by the
  configured policy; it either succeeds within `start()` or the query lands in `Error`.
- **Queries are the only type whose start failure is non-fatal to instance startup.**
  DrasiLib logs and continues when queries fail to start, while source and reaction
  failures abort the instance start.

---

## Reaction

**Lifecycle: full, and the only type with failure detection.**

| Stage | Owner | Notes |
|-------|-------|-------|
| Construct | **Host** | Built outside DrasiLib and handed in fully formed. |
| Register | DrasiLib | **Validates that every referenced query exists**, then creates the node and dependency edges. |
| Provision | DrasiLib | `initialize()` supplies the runtime context. No external I/O. |
| Start | DrasiLib | A configuration check runs first (below), then the reaction starts, subscribes to its queries, and bootstraps before processing begins. |
| Stop | DrasiLib | Moves to `Stopping` first, deliberately, so the supervisor does not mistake a deliberate stop for a failure. |
| Teardown | DrasiLib | Subscription and supervisor tasks are cancelled. |
| Update | DrasiLib | Runtime replaced via `Reconfiguring`. |
| Dispose | **Host** | After release, disposal is outside DrasiLib. |

- `auto_start` is a trait method, as for sources.
- **A configuration rule is enforced at start**: a reaction that requires durable state
  is rejected when no durable state store is configured. This is a configuration error,
  but it can only be evaluated against a constructed object, so it surfaces as a start
  failure.
- **Failure detection exists, but only detection.** A supervisor watches the reaction's
  subscriptions and moves it to `Error` if they are all lost while it is `Running`. It
  does not resubscribe, retry, or restart.

---

## Identity provider

**Lifecycle: none, though it is on the graph.**

When configured, a node is registered at build time under the hard-coded ID
`identity-provider`, with `Authenticates` edges to every source and reaction present at that
moment. The code calls registration "reserved for future use… not yet implemented in the
component lifecycle" — true of the lifecycle, but the node is created.

The node stays at `Added`. The provider itself works normally: it is supplied to sources and
reactions via their runtime contexts and can be overridden per component. No code reads the
node.

No add, remove or update; only one can exist. It is absent from configuration capture, which
is why Drasi Server's clone and export lose identity provider associations.

---

## Providers

Index backends, state stores, WAL and secret stores are supplied at build time: no node, no
status, no transitions, no runtime, no way to add, replace or remove one on a running
instance. Their lifecycle *is* the instance's. Identity is listed for comparison — delivered
the same way, but it also has a graph node.

| Provider | Multiplicity | Reaches components via | Per-component step |
|---|---|---|---|
| Index backend | Named, many; a query selects one by name | Internal query construction | `create_indexes(query_id)` per query |
| State store | Singleton | Source and reaction context | Partitioned by `store_id` |
| WAL | Singleton | Source context | `register(source_id, config)` before use |
| Identity | Singleton | Source and reaction context | — |
| Secret store | Singleton | Not passed to components at all | — |

Two carry a hidden per-component sub-lifecycle: the index backend creates a distinct index
set per query, and the WAL requires each source to register before appending. Neither is on
the graph or in the status model, so neither can fail as a visible lifecycle event.

The secret store is consumed while configuration is resolved — before components are
constructed — and never reaches one.

`Query` receives no providers at all: `QueryRuntimeContext` carries only instance ID, query
ID, and status handle, because DrasiLib wires query indexes directly.

---

## Summary — graph components

Providers are omitted; the answer is uniform for all of them and given in the table above.

| | Source | Bootstrap provider | Query | Reaction | Identity provider |
|---|---|---|---|---|---|
| Constructed | **Outside** | **Outside**, via its source | **Inside** | **Outside** | **Outside** |
| Graph node | ✅ | ✅ | ✅ | ✅ | ✅ (unused) |
| Runtime object | ✅ | ❌ | ✅ | ✅ | ❌ |
| `add`/`remove`/`update` | ✅ | ❌ | ✅ | ✅ | ❌ |
| Uses status transitions | ✅ all | ❌ pinned `Added` | ✅ all | ✅ all | ❌ |
| Dependencies validated at register | — | — | sources exist | queries exist | — |
| `auto_start` lives on | trait method | n/a | config field | trait method | n/a |
| Failure detection | ❌ | n/a | ❌ | ✅ detect only | n/a |
| Start failure at instance start | fatal | n/a | **non-fatal** | fatal | n/a |
| Captured in configuration snapshot | ✅ | ✅ via source | ✅ | ✅ | ❌ |

## Observations

Recorded as input to later design work, not as proposals.

1. **Only three of the nine component types are lifecycle-bearing.** Bootstrap and identity
   providers have graph presence without graph behavior; the four injected providers have
   neither. The predictor in every case is whether a runtime object is stored.
2. **The graph is an incomplete dependency model.** It records source, query, reaction and
   bootstrap relationships, but not the provider dependencies — index backend, state store,
   WAL — that a component equally requires in order to function.
3. **Singleton providers cannot participate in runtime composition.** Because they exist
   only at build time, they cannot be added to a running instance, and anything built on
   per-instance composition inherits that limit.
4. **Identity provider edges are a build-time snapshot.** They are drawn once against the
   components present at build, and never updated for components added later.
5. **Bootstrap providers are first-class in Drasi Server and second-class in DrasiLib.**
6. **Creation does not establish validity.** A query is compiled only at start, so it
   can be created successfully and still be invalid.
7. **Some configuration errors are only detectable after construction**, because they
   depend on properties of the constructed object rather than of its config. No purely
   static validation pass can catch them.
8. **Wiring is already separable from connecting** for sources, which accept
   subscriptions before being started.
9. **Terminal versus non-terminal is decided by component type, not by cause.** A query
   failing to start is non-fatal; a source failing to start is fatal — regardless of
   whether the cause was a typo or an unreachable system.
7. **`Error` conflates "misconfigured" with "disconnected."** Both reach the same state,
   distinguished only by a free-text message.
8. **Nothing leaves `Error` on its own.** The transition is legal but is never performed
   automatically. Reactions can detect subscription loss; no type can recover from it.
