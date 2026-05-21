# Atomic Multi-Component Deployment for DrasiLib

## Overview

Today, deploying multiple components (sources, queries, reactions) as a logical unit in DrasiLib is not atomic. Two code paths create multiple components sequentially:

1. **Config file loading at startup** - loops over sources, queries, reactions; aborts on first failure with no rollback of already-created components.
2. **Instance clone** - best-effort rollback on failure, but vulnerable to concurrent mutation races.

When component #3 of 5 fails, the system is left in a partially-deployed state: some components exist in the graph and runtime, others don't. This is confusing for operators and hard to recover from. This proposal introduces a staging model that separates slow preparation from fast atomic commitment, ensuring the live system is never in an intermediate state.

## Terms and Definitions

| Term | Definition |
|------|------------|
| Staging Area | A private, per-operation buffer that accumulates initialized components before committing them atomically to the live graph. |
| DeployStaging | The Rust struct representing a staging area that holds initialized but not-yet-committed components. |
| GraphTransaction | An existing abstraction in DrasiLib that provides atomic graph mutations with automatic rollback on failure. |
| ComponentGraph | The live, shared graph structure that tracks all deployed sources, queries, and reactions and their relationships. |
| Compensating Rollback | The current approach of manually undoing completed steps when a later step fails — error-prone and complex. |

## Objectives

### User Scenarios

**Operator deploying a multi-component pipeline**: An operator deploys a source, two queries, and a reaction as a logical unit. If the second query fails to initialize, none of the components should appear in the live system. The operator should see a clean error and be able to retry the entire deployment without manual cleanup.

**Instance clone**: A user clones an existing Drasi instance. The clone operation creates all components of the source instance in the target. If any component fails to create, the target instance should remain in its prior state with no orphaned components.

### Goals

1. **All-or-nothing semantics**: Multi-component deployments either fully succeed or leave the live system completely unchanged.
2. **No blocking**: Slow component initialization (I/O, plugin loading) must not hold any shared locks that block other operations.
3. **Eliminate compensating rollback**: Remove the need for manual undo logic that can itself fail and leave orphans.
4. **Concurrency safety**: Concurrent operations (deploys, deletes, reads) must not observe or interact with partially-deployed components.
5. **Clean error reporting**: Failures at any stage produce a clear error and require no manual recovery.

### Non-Goals

1. **Persistence atomicity**: This proposal does not address the case where components are committed to memory but persistence to disk fails. That is an existing orthogonal concern.
2. **Single-component operations**: The existing `add_source_with_metadata` / `remove_source` APIs for individual components are not being replaced — they already handle their own single-component error cases.
3. **Distributed transactions**: This proposal addresses atomicity within a single DrasiLib instance, not across multiple instances or external systems.

## Design Requirements

### Requirements

1. Components must be fully initialized (including potentially slow I/O) before any shared state is modified.
2. The graph write lock must only be held for in-memory operations (sub-millisecond duration).
3. If any component registration fails during commit, the entire batch must be rolled back automatically via `GraphTransaction`.
4. Dropping a staging area without committing must be safe and require no cleanup of shared state.
5. Auto-start failures after commit must be non-fatal and reported to the caller.

### Dependencies

- **GraphTransaction** (drasi-lib): The existing transaction abstraction that provides atomic graph mutations with automatic rollback.
- **ComponentGraph** (drasi-lib): The live graph structure with `RwLock` semantics.
- **Source/Query/Reaction `initialize()` methods** (drasi-lib): Must be callable without holding any graph lock (already the case).

### Out of Scope

- **Persistence layer changes**: The current behavior where persistence happens after commit at the Drasi Server layer is unchanged.
- **Plugin factory failures**: Plugin factories are called at the Drasi Server layer before staging. If creating a plugin instance fails, no staging area has been created yet.

## Design

### High-Level Design

The core insight is: **separate slow preparation from fast atomic commitment.** Like git — staging is private and can take as long as needed, committing is shared and instant.

The key property: **the live system is never in an intermediate state.** Components are either fully deployed (after commit) or don't exist at all (before commit, or if anything fails). There is no window where rollback from the live graph is needed.

### Architecture Diagram

```
 begin_deploy()         add_source()           commit()
      │                      │                     │
      ▼                      ▼                     ▼
┌──────────┐    ┌─────────────────────┐    ┌──────────────────────┐
│  Create  │    │  Initialize source  │    │  Acquire write lock  │
│  staging │───▶│  (stores in staging │───▶│  Single GraphTxn:    │
│  area    │    │   NOT in live graph) │    │    register all      │
│          │    │                     │    │    store all runtimes │
│          │    │  No lock held.      │    │  Commit txn          │
│          │    │  Can be slow.       │    │  Release write lock  │
│          │    │  Only affects this  │    │  (sub-millisecond)   │
│          │    │  operation.         │    │                      │
└──────────┘    └─────────────────────┘    └──────────────────────┘
                         │                         │
                    On failure:                On failure:
                    Drop staging area.         GraphTxn auto-rollback.
                    Live graph untouched.      Drop staged runtimes.
                    No rollback needed.        Live graph untouched.
```

### Detail Design

#### Why This Eliminates Blocking and Rollback Problems

| Concern | Answer |
|---------|--------|
| Provisioning hangs? | Only this deploy is stuck. No lock is held. All other operations proceed normally. |
| Need to cancel? | Drop the staging area. Live graph never knew about it. No cleanup needed. |
| Concurrent mutation during deploy? | Proceeds normally — staging is invisible to other callers. |
| Commit conflicts (duplicate ID)? | `GraphTransaction` fails at commit → auto-rollback → staged runtimes dropped → clean error. |
| Dependency removed between staging and commit? | Commit validates deps → fails → transaction rolls back → clean error. |
| Provisioning failure? | Drop staging area. Live graph is untouched. No deregistration, no compensation. |

#### Comparison with Mutex-Based Approach

| Property | Mutex Approach | Staging Approach |
|----------|---------------|------------------|
| Blocks other operations during slow work? | Yes — mutex held for entire operation | No — no shared lock during preparation |
| Hangs forever if plugin hangs? | Yes — mutex never released | No — only this deploy is stuck |
| Needs compensating rollback from live graph? | Yes — Phase 2 failures require deregister | No — live graph never touched until commit |
| Complexity of rollback? | High — must track and undo each step | None — just drop the staging area |
| Lock hold duration | Unbounded (provisioning time) | Sub-millisecond (in-memory graph ops only) |

#### Commit Implementation

The commit acquires the graph write lock and performs all registrations in a single `GraphTransaction`. If ANY registration fails (duplicate ID, missing dependency, etc.), the entire transaction is rolled back automatically and the live graph is unchanged. The write lock is held only for in-memory graph operations (sub-millisecond).

After commit, components are optionally auto-started. Start failures are non-fatal and reported in the result.

#### Drop Safety

If `commit()` is not called, all staged runtimes are simply dropped. No graph cleanup is needed because nothing was ever written to the live graph. The `Arc<dyn Source/Query/Reaction>` destructors handle any resource cleanup (closing connections, releasing memory, etc.). Abort is free.

#### Auto-Start Semantics

| Concern | Resolution |
|---------|-----------|
| Who owns auto-start? | **DrasiLib** owns it within `commit()`. The server does NOT call `start_*` separately. |
| What if start fails? | Non-fatal. Component remains in graph (Added/Error state). Reported in `DeployResult.start_failures`. |
| Is it returned to the API caller? | Yes — the server handler surfaces `start_failures` in the response body (HTTP 200 with warnings, not 500). |
| Is it persisted? | Yes — the component is persisted (it was successfully created). On restart, auto-start will be reattempted. |
| Double-start bug? | Eliminated. Staging + commit doesn't go through `add_source_with_metadata` (which has its own auto-start logic). |

#### Failure Semantics Summary

| Point of Failure | Behavior | Observable State |
|-----------------|----------|-----------------|
| `add_source/query/reaction` (staging) | Staging area dropped. Runtimes cleaned up via `Drop`. | Unchanged — live graph never touched. |
| `commit()` — graph registration fails | `GraphTransaction` auto-rollback. Staged runtimes dropped. | Unchanged — live graph never touched. |
| `commit()` — `set_runtime` fails after txn commit | Components registered in graph but missing runtime. Needs cleanup (see Edge Case below). | Partially committed — requires recovery. |
| Post-commit auto-start | Non-fatal. Components exist but aren't running. | Components deployed, some not started. Caller informed via `start_failures`. |
| Persistence (Drasi Server layer) | Components exist in runtime but not on disk. | Runtime ahead of disk — existing behavior, orthogonal to this proposal. |

#### Edge Case: `set_runtime` Failure After Txn Commit

If `graph.set_runtime()` fails after `txn.commit()` but before all runtimes are stored, we have a partially-committed state. This is unlikely (it's just a HashMap insert that can only fail if the ID doesn't exist — but we just registered it). To handle this defensively, after `txn.commit()`, if `set_runtime` fails, deregister everything that was just committed. This is the ONE compensating rollback path, and it's guaranteed fast (in-memory only, under the write lock we already hold).

#### Concurrency Properties

No mutex is needed. The concurrency model relies on two properties:

1. **Staging is private**: Other operations cannot see or interact with staged components. No shared state is mutated during preparation.
2. **Commit is instant**: The write lock is held only for in-memory graph mutations (adding nodes, edges, storing `Arc` pointers into a `HashMap`). This is O(n) in number of components with constant-time per-component — a deploy of 100 components holds the lock for microseconds.

| Scenario | Behavior |
|----------|----------|
| Two concurrent deploys | Both stage independently. First to `commit()` wins the write lock. Second either succeeds (no conflicts) or fails at registration (duplicate ID → clean error, drop staging). |
| Deploy + concurrent `DELETE /sources/X` | If source X is a dependency, the deploy's commit will fail validation ("source X not found") → clean error. If not a dependency, both succeed independently. |
| Deploy + concurrent `POST /sources/Y` with same ID | Whichever commits first wins. The second gets a "duplicate ID" error at registration → clean rollback. |
| Deploy + concurrent snapshot/read | Reads use the RwLock read path. They see the graph state before commit or after commit — never in between (write lock provides this atomicity). |

### API Design

#### Public Entry Point

```rust
impl DrasiLib {
    /// Begin a staged batch deployment.
    ///
    /// Returns a `DeployStaging` that accumulates components privately.
    /// Nothing touches the live ComponentGraph until `commit()` is called.
    pub fn begin_deploy(&self) -> DeployStaging {
        DeployStaging::new(self)
    }
}
```

#### DeployStaging Struct

```rust
/// A staging area for batch component deployment.
///
/// Components are initialized and held privately. The live ComponentGraph
/// is not modified until `commit()` is called. If dropped without commit,
/// all staged runtimes are cleaned up and the live system is unchanged.
pub struct DeployStaging<'a> {
    core: &'a DrasiLib,
    staged_sources: Vec<StagedSource>,
    staged_queries: Vec<StagedQuery>,
    staged_reactions: Vec<StagedReaction>,
    auto_start: bool,
}

struct StagedSource {
    id: String,
    runtime: Arc<dyn Source>,
    metadata: HashMap<String, String>,
}

struct StagedQuery {
    id: String,
    runtime: Arc<dyn Query>,
    config: QueryConfig,
}

struct StagedReaction {
    id: String,
    runtime: Arc<dyn Reaction>,
    metadata: HashMap<String, String>,
}
```

#### Staging Methods (Slow, No Lock)

```rust
impl<'a> DeployStaging<'a> {
    /// Stage a source for deployment.
    ///
    /// Initializes the source runtime privately. This may involve I/O
    /// (plugin initialization) and can take arbitrary time. No graph
    /// lock is held. If this fails, nothing has been modified.
    pub async fn add_source(
        &mut self,
        source: impl Source + 'static,
        metadata: HashMap<String, String>,
    ) -> Result<()> {
        let source: Arc<dyn Source> = Arc::new(source);
        let id = source.id().to_string();

        let context = SourceRuntimeContext::new(
            &self.core.config.instance_id,
            &id,
            self.core.source_manager.state_store().await,
            self.core.source_manager.update_tx(),
            None,
        );

        // Initialize — potentially slow, no graph lock held
        source.initialize(context).await;

        self.staged_sources.push(StagedSource {
            id,
            runtime: source,
            metadata,
        });
        Ok(())
    }

    /// Stage a query for deployment.
    pub async fn add_query(&mut self, config: QueryConfig) -> Result<()> {
        let id = config.id.clone();

        let query = DrasiQuery::new(
            &self.core.config.instance_id,
            config.clone(),
            self.core.source_manager.clone(),
            self.core.query_manager.index_factory(),
            self.core.middleware_registry.clone(),
        )?;

        let context = QueryRuntimeContext::new(
            &self.core.config.instance_id,
            &id,
            self.core.query_manager.update_tx(),
        );
        query.initialize(context).await;

        let query: Arc<dyn Query> = Arc::new(query);

        self.staged_queries.push(StagedQuery { id, runtime: query, config });
        Ok(())
    }

    /// Stage a reaction for deployment.
    pub async fn add_reaction(
        &mut self,
        reaction: impl Reaction + 'static,
        metadata: HashMap<String, String>,
    ) -> Result<()> {
        let reaction: Arc<dyn Reaction> = Arc::new(reaction);
        let id = reaction.id().to_string();

        let context = ReactionRuntimeContext::new(
            &self.core.config.instance_id,
            &id,
            self.core.reaction_manager.state_store().await,
            self.core.reaction_manager.update_tx(),
            None,
        );

        // Initialize — potentially slow, no graph lock held
        reaction.initialize(context).await;

        self.staged_reactions.push(StagedReaction {
            id,
            runtime: reaction,
            metadata,
        });
        Ok(())
    }

    /// Set whether to auto-start components after commit.
    /// Default: true.
    pub fn auto_start(mut self, auto_start: bool) -> Self {
        self.auto_start = auto_start;
        self
    }
}
```

#### Commit (Fast, Atomic)

```rust
impl<'a> DeployStaging<'a> {
    /// Atomically commit all staged components to the live graph.
    ///
    /// This acquires the graph write lock and performs all registrations
    /// in a single `GraphTransaction`. If ANY registration fails (duplicate
    /// ID, missing dependency, etc.), the entire transaction is rolled back
    /// automatically and the live graph is unchanged.
    ///
    /// The write lock is held only for in-memory graph operations
    /// (sub-millisecond). It cannot hang or block for I/O.
    ///
    /// After commit, optionally starts all components with auto_start=true.
    /// Start failures are non-fatal and reported in the result.
    pub async fn commit(self) -> Result<DeployResult> {
        self.core.state_guard.require_initialized()?;

        let source_ids: Vec<String> = self.staged_sources.iter().map(|s| s.id.clone()).collect();
        let query_ids: Vec<String> = self.staged_queries.iter().map(|q| q.id.clone()).collect();
        let reaction_ids: Vec<String> = self.staged_reactions.iter().map(|r| r.id.clone()).collect();

        // Atomic commit: single write lock, single GraphTransaction
        {
            let mut graph = self.core.component_graph.write().await;
            let mut txn = graph.begin();

            // Register all sources
            for staged in &self.staged_sources {
                let mut meta = HashMap::new();
                meta.insert("kind".to_string(), staged.runtime.type_name().to_string());
                meta.insert("autoStart".to_string(), staged.runtime.auto_start().to_string());
                meta.extend(staged.metadata.clone());

                let node = ComponentNode {
                    id: staged.id.clone(),
                    kind: ComponentKind::Source,
                    status: ComponentStatus::Added,
                    metadata: meta,
                };
                txn.add_component(node).map_err(|e| {
                    DrasiError::operation_failed("source", &staged.id, "deploy", format!("{e}"))
                })?;
            }

            // Register all queries with dependency edges
            for staged in &self.staged_queries {
                let source_deps = &staged.config.sources;

                for dep in source_deps {
                    let exists_in_graph = graph.contains(&dep.source_id);
                    let exists_in_batch = source_ids.contains(&dep.source_id);
                    if !exists_in_graph && !exists_in_batch {
                        return Err(DrasiError::operation_failed(
                            "query", &staged.id, "deploy",
                            format!("Source '{}' not found", dep.source_id),
                        ));
                    }
                }

                let mut meta = HashMap::new();
                meta.insert("autoStart".to_string(), staged.config.auto_start.to_string());
                let node = ComponentNode {
                    id: staged.id.clone(),
                    kind: ComponentKind::Query,
                    status: ComponentStatus::Added,
                    metadata: meta,
                };
                txn.add_component(node).map_err(|e| {
                    DrasiError::operation_failed("query", &staged.id, "deploy", format!("{e}"))
                })?;

                for dep in source_deps {
                    txn.add_relationship(&dep.source_id, &staged.id, RelationshipKind::Feeds)?;
                }
            }

            // Register all reactions with dependency edges
            for staged in &self.staged_reactions {
                let query_deps = staged.runtime.query_ids();

                for dep in &query_deps {
                    let exists_in_graph = graph.contains(dep);
                    let exists_in_batch = query_ids.contains(dep);
                    if !exists_in_graph && !exists_in_batch {
                        return Err(DrasiError::operation_failed(
                            "reaction", &staged.id, "deploy",
                            format!("Query '{}' not found", dep),
                        ));
                    }
                }

                let mut meta = HashMap::new();
                meta.insert("kind".to_string(), staged.runtime.type_name().to_string());
                meta.insert("autoStart".to_string(), staged.runtime.auto_start().to_string());
                meta.extend(staged.metadata.clone());
                let node = ComponentNode {
                    id: staged.id.clone(),
                    kind: ComponentKind::Reaction,
                    status: ComponentStatus::Added,
                    metadata: meta,
                };
                txn.add_component(node).map_err(|e| {
                    DrasiError::operation_failed("reaction", &staged.id, "deploy", format!("{e}"))
                })?;

                for dep in &query_deps {
                    txn.add_relationship(dep, &staged.id, RelationshipKind::Feeds)?;
                }
            }

            // All registrations passed — commit the transaction
            txn.commit();

            // Store all runtimes in the graph (still under write lock)
            for staged in self.staged_sources {
                graph.set_runtime(&staged.id, Box::new(staged.runtime))?;
            }
            for staged in self.staged_queries {
                graph.set_runtime(&staged.id, Box::new(staged.runtime))?;
            }
            for staged in self.staged_reactions {
                graph.set_runtime(&staged.id, Box::new(staged.runtime))?;
            }
        }
        // Write lock released here — total hold time: sub-millisecond

        // Post-commit: Auto-start (non-fatal, no lock held)
        let mut start_failures = Vec::new();

        if self.auto_start && self.core.is_running().await {
            for id in &source_ids {
                if let Err(e) = self.core.source_manager.start_source(id.clone()).await {
                    start_failures.push(StartFailure {
                        component_id: id.clone(),
                        component_type: "source".to_string(),
                        error: e.to_string(),
                    });
                }
            }
            for id in &query_ids {
                if let Err(e) = self.core.query_manager.start_query(id.clone()).await {
                    start_failures.push(StartFailure {
                        component_id: id.clone(),
                        component_type: "query".to_string(),
                        error: e.to_string(),
                    });
                }
            }
            for id in &reaction_ids {
                if let Err(e) = self.core.reaction_manager.start_reaction(id.clone()).await {
                    start_failures.push(StartFailure {
                        component_id: id.clone(),
                        component_type: "reaction".to_string(),
                        error: e.to_string(),
                    });
                }
            }
        }

        Ok(DeployResult {
            sources_created: source_ids,
            queries_created: query_ids,
            reactions_created: reaction_ids,
            start_failures,
        })
    }
}
```

#### Drop Safety

```rust
impl<'a> Drop for DeployStaging<'a> {
    fn drop(&mut self) {
        // If commit() was not called, all staged runtimes are simply dropped.
        // No graph cleanup is needed because nothing was ever written to the
        // live graph. The Arc<dyn Source/Query/Reaction> destructors handle
        // any resource cleanup (closing connections, releasing memory, etc.).
        //
        // This is the key advantage of the staging model: abort is free.
    }
}
```

#### Result Types

```rust
#[derive(Debug, Clone)]
pub struct DeployResult {
    pub sources_created: Vec<String>,
    pub queries_created: Vec<String>,
    pub reactions_created: Vec<String>,
    /// Components that were committed but failed to auto-start.
    /// These exist in the graph in Added/Error state and can be
    /// started manually later.
    pub start_failures: Vec<StartFailure>,
}

#[derive(Debug, Clone)]
pub struct StartFailure {
    pub component_id: String,
    pub component_type: String,
    pub error: String,
}
```

### How This Changes Existing Code

#### Instance Clone (Drasi Server)

```rust
// Before: manual loop with manual rollback (250+ lines)
for src_snap in &snapshot.sources {
    let (source, meta) = create_source_locked(...).await?;
    core.add_source_with_metadata(source, meta).await?;
    sources_created.push(...);
}
// ... queries, reactions ...
// ... rollback_sources, rollback_queries, rollback_reactions ...

// After: staging model
let mut staging = target_core.begin_deploy().auto_start(false);
for src_snap in &snapshot.sources {
    let (source, meta) = create_source_locked(&plugin_registry, source_config).await?;
    staging.add_source(source, meta).await?;
}
for q_snap in &snapshot.queries {
    staging.add_query(q_snap.config.clone()).await?;
}
for rx_snap in &snapshot.reactions {
    let (reaction, meta) = create_reaction_locked(&plugin_registry, reaction_config).await?;
    staging.add_reaction(reaction, meta).await?;
}
// If any add_* above fails, staging is dropped. Live graph untouched. Done.

staging.commit().await?;  // Atomic — all or nothing
persist_after_operation(&config_persistence, "cloning instance").await?;
```

The entire manual rollback infrastructure (`rollback_sources`, `rollback_queries`, `rollback_reactions`, `clone_error`) is eliminated.

#### Config File Loading (Drasi Server Startup)

The DrasiLib builder used at startup already accumulates components before `build()` — this is effectively the same pattern (prepare privately, then commit). No change needed for the startup path.

For a future "apply config changes at runtime" feature, `begin_deploy()` provides the atomic primitive.

#### Single-Component API Handlers

Individual `POST /sources`, `DELETE /queries`, etc. continue to use the existing `add_source_with_metadata` / `remove_source` methods. These are single-step operations that already handle their own error cases.

### Alternatives Considered

#### Mutex-Based Approach

A simpler alternative would be to wrap the entire multi-component operation in a mutex, preventing concurrent access during deployment.

**Why this was rejected:**
- Holds the lock for the entire duration of slow initialization (I/O, plugin loading), blocking all other operations.
- If a plugin hangs during initialization, the mutex is never released.
- Still requires compensating rollback from the live graph on failure.
- Lock hold duration is unbounded and proportional to the number of components × initialization time.

The staging approach is strictly superior: it achieves the same atomicity guarantees without blocking and without compensating rollback.

## Security

N/A — This proposal does not introduce new external interfaces, network endpoints, or credential handling. It is an internal restructuring of deployment mechanics.

## Compatibility Impact

- **No breaking changes to public APIs**: The existing `add_source_with_metadata`, `add_query`, `add_reaction_with_metadata` methods remain unchanged.
- **New API surface**: `begin_deploy()` is additive and does not affect existing callers.
- **Behavioral change for instance clone**: Clone operations will now be atomic (all-or-nothing) instead of best-effort-with-rollback. This is a strictly better behavior for operators.

## Supportability

### Telemetry

- Log at `INFO` level when a staging area is created, when components are staged, and when commit succeeds.
- Log at `WARN` level for auto-start failures (with component IDs and error details).
- Log at `ERROR` level for commit failures (with the failing component and reason).
- Metric: `deploy_commit_duration_ms` — time spent holding the write lock during commit (expected sub-millisecond).

### Verification

1. **Unit tests**: Verify `DeployStaging` correctly accumulates components and that drop without commit is safe.
2. **Integration tests**:
   - All-or-nothing: if one component fails during staging, none appear in the graph.
   - All-or-nothing: if commit fails (duplicate ID), the graph is unchanged.
   - Concurrent deploys don't corrupt the graph.
   - Drop without commit leaves graph unchanged.
   - Start failures are reported but don't affect committed state.
3. **Stress tests**: Multiple concurrent deploys + deletes + reads to verify no data races or deadlocks.

## Open Issues

1. **Should `set_runtime` move into the GraphTransaction?** Today `GraphTransaction` only handles nodes and edges. Extending it to also stage runtimes (stored on commit, dropped on rollback) would eliminate the edge case where `set_runtime` fails after `txn.commit()`. This is a small addition to `GraphTransaction` and would make the commit truly atomic across registration + runtime storage.

2. **Should single-component operations use staging too?** The existing `add_source_with_metadata` has its own compensating rollback (deregister on provision failure). Should it be rewritten to use a staging deploy of one component? This would unify the code paths but adds indirection for the common case. Recommendation: keep single-component ops as-is for now; they work correctly and are simpler.
