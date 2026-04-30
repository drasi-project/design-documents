# Dual-Load Rolling Plugin Upgrade

## Overview

This proposal details how Drasi can support **zero-downtime plugin upgrades** by loading both old and new plugin binaries simultaneously, then migrating dependent components one-at-a-time using the existing `update_source/reaction` + `Reconfiguring` infrastructure.

The key innovation is **Arc-based library lifetime management**: each runtime holds a reference to its parent plugin's shared library handle. The library stays loaded as long as any runtime from it exists, and is automatically unloaded when all references are dropped. This eliminates the unsafe `dlclose()` timing problem entirely.

This feature enables operators to upgrade source and reaction plugins without taking down the entire system, providing a smooth path from one version to the next while maintaining data flow continuity.

## Terms and definitions

| Term | Definition |
|------|------------|
| Dual-Load | The state where both old and new versions of a plugin are loaded simultaneously in memory |
| Rolling Upgrade | Migrating dependent components one-at-a-time from old to new plugin version |
| Arc-based lifetime | Using Rust's `Arc` (Atomic Reference Counting) to manage shared library lifetimes safely |
| PluginHandle | A reference-counted wrapper around a loaded plugin's shared library |
| Retiring plugin | The old version of a plugin being phased out during an upgrade |
| ABI compatibility | Binary interface compatibility between plugin versions allowing dual-load |
| vtable | The virtual function table used for FFI between host and plugin |

## Objectives

### User scenarios

**Operator upgrading a source plugin**: A Drasi operator running a production instance with 8 PostgreSQL sources needs to upgrade the `source-postgres` plugin from v1.0 to v2.1. They want zero downtime — queries and reactions should continue operating during the upgrade with at most a brief event gap per source.

**Operator rolling back a failed upgrade**: During a rolling upgrade, the 4th source fails to start on the new plugin version. The operator wants to abort and restore all sources to the previous working version without manual intervention.

**Plugin author releasing a patch**: A plugin author has fixed a bug in their source plugin. They want operators to be able to apply the fix without restarting the entire Drasi server.

### Goals

- Enable zero-downtime plugin upgrades using a rolling migration strategy
- Provide safe, automatic library unloading via Arc-based lifetime management
- Support rollback to the previous version at any point during the upgrade
- Maintain data flow continuity for queries and reactions during source upgrades
- Validate ABI compatibility before attempting a dual-load upgrade
- Persist upgrade state for crash recovery
- Expose upgrade lifecycle through REST API and CLI

### Non-Goals

- Cross-ABI version upgrades via dual-load (these require a restart-based upgrade)
- Automatic triggering of upgrades when new binaries are detected (future enhancement)
- Blue-green instance-level upgrades (future consideration)
- Plugin state migration framework (plugin authors implement their own `migrate_state` hook)

## Design requirements

### Requirements

- Dual-load must be safe: no symbol conflicts, no undefined behavior on unload
- Library lifetime must be structurally guaranteed (no manual `dlclose()` timing)
- Upgrade must be resumable after a server crash
- Rollback must be symmetric with upgrade (same mechanism, reverse direction)
- Components not yet migrated must continue running on old version undisturbed
- New components created during upgrade must use the new version
- Memory overhead from dual-load must be bounded and temporary

### Dependencies

- **Reliable start/stop** — If `stop_source()` hangs, the upgrade stalls. The resilience work must make start/stop reliable with timeouts and error states.
- **Atomic batch deploy (staging model)** — Rollback benefits from atomic revert of multiple components. The staging model proposal covers this.
- **Plugin ABI versioning** — Need explicit ABI version in plugin metadata (beyond just SDK crate version). Partially exists today.
- **`update_source`/`update_reaction` stability** — These operations must be battle-tested before using them for production upgrades.

## Design

### High-level design

The upgrade lifecycle follows a state machine with clear phases:

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Upgrade Lifecycle                                  │
│                                                                        │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────────┐  │
│  │  Plan   │───▶│ Execute  │───▶│ Complete │───▶│ Old lib dropped │  │
│  │         │    │ (rolling)│    │          │    │ (Arc count = 0) │  │
│  └─────────┘    └──────────┘    └──────────┘    └─────────────────┘  │
│       │              │               │                                 │
│       │              ▼               ▼                                 │
│       │         ┌──────────┐    ┌──────────┐                          │
│       │         │ Rollback │◀───│  Abort   │                          │
│       │         │          │    │          │                          │
│       ▼         └──────────┘    └──────────┘                          │
│  ┌──────────────────────────────────────────────┐                     │
│  │            ComponentGraph                     │                     │
│  │  ┌─────────────────────────────────────────┐ │                     │
│  │  │ UpgradePlan { plugin, from, to, ... }   │ │                     │
│  │  │ UpgradeTargets [comp1, comp2, comp3]    │ │                     │
│  │  └─────────────────────────────────────────┘ │                     │
│  └──────────────────────────────────────────────┘                     │
└──────────────────────────────────────────────────────────────────────┘
```

**Core Mechanisms**:
1. **Arc-based library lifetime** — The shared library handle is wrapped in an `Arc`. Each runtime holds a clone. The library unloads automatically when the last reference is dropped.
2. **Versioned plugin registry** — During upgrade, the registry tracks both the new (primary) and old (retiring) plugin descriptors.
3. **Rolling migration** — Components are upgraded one-at-a-time using the existing `update_source/reaction` infrastructure.
4. **Crash-safe state machine** — The `UpgradePlan` is persisted in the ComponentGraph and can be resumed after a crash.

### Architecture Diagram

```
┌────────────────────────────────────────────────────────┐
│                    DrasiServer                           │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              PluginRegistry                      │    │
│  │  ┌─────────────────┐  ┌──────────────────────┐  │    │
│  │  │ plugins (active) │  │ retiring (old vers.) │  │    │
│  │  │ kind → descriptor│  │ kind → descriptor    │  │    │
│  │  └─────────────────┘  └──────────────────────┘  │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              ComponentGraph                       │    │
│  │                                                   │    │
│  │  [Source A] ─── PluginHandle(Arc<Lib v2>) ───┐   │    │
│  │  [Source B] ─── PluginHandle(Arc<Lib v2>) ───┤   │    │
│  │  [Source C] ─── PluginHandle(Arc<Lib v1>) ───┤   │    │
│  │                                               │   │    │
│  │  Arc<Lib v2> strong_count = 2                 │   │    │
│  │  Arc<Lib v1> strong_count = 1                 │   │    │
│  │                                               │   │    │
│  │  UpgradePlan { status: InProgress, ... }      │   │    │
│  └─────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

### Detail design

#### Arc-based Library Lifetime Management

The shared library handle is wrapped in an `Arc`. Each runtime created from the plugin holds a clone of this Arc. The library is automatically unloaded when the last Arc is dropped.

```rust
use std::sync::Arc;
use libloading::Library;

/// A loaded plugin binary with reference-counted lifetime.
/// The Library is unloaded (dlclose) only when all PluginHandles are dropped.
pub struct PluginHandle {
    /// Reference-counted library handle.
    /// When Arc strong count reaches 0, Library drops and calls dlclose.
    library: Arc<Library>,
    
    /// Plugin metadata (cached at load time)
    metadata: PluginMetadata,
    
    /// Factory function pointer (valid as long as library is loaded)
    factory: PluginFactory,
}

impl PluginHandle {
    /// Create a new reference to this plugin's library.
    /// The returned handle keeps the library loaded.
    pub fn clone_handle(&self) -> PluginHandle {
        PluginHandle {
            library: Arc::clone(&self.library),
            metadata: self.metadata.clone(),
            factory: self.factory,
        }
    }
}

/// Each runtime (Source, Query, Reaction trait object) holds a PluginHandle.
/// This keeps the parent library alive as long as the runtime exists.
pub struct PluginRuntime<T: ?Sized> {
    /// The actual runtime (Source, Query, or Reaction trait object)
    runtime: Box<T>,
    
    /// Keeps the plugin library loaded while this runtime exists
    _handle: PluginHandle,
}

impl<T: ?Sized> Drop for PluginRuntime<T> {
    fn drop(&mut self) {
        // runtime drops first, then _handle drops.
        // If this was the last PluginHandle, the library is unloaded.
        // Order is guaranteed by Rust's drop order (fields drop in declaration order).
    }
}
```

**Lifecycle guarantees**:
1. During normal operation: each component's runtime holds one `PluginHandle` → Arc strong count = N (where N = number of components using this plugin).
2. During upgrade: new runtimes hold handles to the new library. Old runtimes still hold handles to the old library.
3. When `update_source/reaction` swaps a runtime: the old runtime is dropped → old library's Arc count decreases by 1.
4. When ALL old runtimes have been swapped: old library's Arc count = 0 → `Library::drop()` → `dlclose()` is called automatically.
5. No manual `dlclose()` call needed. No timing risk. No undefined behavior.

> ⚠️ **Caveat:** Guarantee #5 is overstated. The [Library Unloading Risk Analysis](#library-unloading-risk-analysis) documents multiple scenarios where `dlclose`-on-drop produces undefined behavior (background tasks outliving drop, escaped trait objects, thread-local destructors). The POC uses `mem::forget` and never calls `dlclose`, sidestepping these guarantees entirely.

#### Symbol/Resource Conflict Resolution

Two cdylib copies of the same plugin loaded via `dlopen` could conflict. This is resolved using `RTLD_LOCAL` flag (already the default for Rust's `libloading`).

```rust
// libloading::Library::new() uses RTLD_LOCAL by default on Unix.
// This means symbols from the loaded library are NOT visible to other
// loaded libraries. Each plugin's symbols are isolated.
let lib = unsafe { libloading::Library::new(&path) }?;
```

**Why this is sufficient**:
- Drasi plugins are Rust-native and use `rustls` (not system OpenSSL)
- Each cdylib plugin already creates its own isolated tokio runtime
- `RTLD_LOCAL` prevents symbol leakage between libraries
- Port/resource conflicts don't apply because plugins don't bind ports — they communicate through the host-sdk vtable callbacks

**Plugin authoring constraint**: *"Plugins that use C libraries with process-global state must not assume exclusive access to that state. During upgrades, two versions of your plugin may be loaded simultaneously."*

#### Versioned Plugin Registry

The `PluginRegistry` is extended to support versioned loading during upgrades:

```rust
pub struct PluginRegistry {
    /// Normal operation: kind → latest loaded descriptor
    /// During upgrade: kind → descriptor of the NEW version (forward progress)
    plugins: HashMap<PluginKind, PluginDescriptor>,
    
    /// During upgrade only: tracks the old version being phased out.
    /// Removed when Arc<Library> strong count reaches 0.
    retiring: HashMap<PluginKind, PluginDescriptor>,
}

impl PluginRegistry {
    /// Load a new version of an already-loaded plugin.
    /// Moves current descriptor to `retiring`, stores new as primary.
    pub fn upgrade_plugin(
        &mut self,
        kind: PluginKind,
        new_descriptor: PluginDescriptor,
    ) -> Result<(), PluginError> {
        if let Some(current) = self.plugins.remove(&kind) {
            self.retiring.insert(kind.clone(), current);
        }
        self.plugins.insert(kind, new_descriptor);
        Ok(())
    }
    
    /// Get factory for creating NEW components (always uses latest).
    pub fn get_factory(&self, kind: &PluginKind) -> Option<&PluginFactory> {
        self.plugins.get(kind).map(|d| &d.factory)
    }
    
    /// Get factory for a SPECIFIC version (used during rollback).
    pub fn get_retiring_factory(&self, kind: &PluginKind) -> Option<&PluginFactory> {
        self.retiring.get(kind).map(|d| &d.factory)
    }
    
    /// Called when old library is fully unloaded.
    pub fn complete_retirement(&mut self, kind: &PluginKind) {
        self.retiring.remove(kind);
    }
}
```

**Routing rules during upgrade**:

| Operation | Which factory? | Rationale |
|-----------|---------------|-----------|
| Create new component | NEW version | Forward progress — don't create more work |
| Upgrade existing component | NEW version | That's the whole point |
| Rollback component | OLD version (from `retiring`) | Revert to known-good |
| Component not yet migrated continues running | OLD version (existing runtime) | No change until explicitly migrated |

#### ABI Compatibility Validation

```rust
/// Validates that two plugin versions can coexist during a rolling upgrade.
pub fn validate_abi_compatibility(
    current: &PluginMetadata,
    candidate: &PluginMetadata,
) -> Result<(), UpgradeValidationError> {
    // 1. Same host-sdk ABI version (major.minor must match)
    if current.sdk_abi_version.major != candidate.sdk_abi_version.major
        || current.sdk_abi_version.minor != candidate.sdk_abi_version.minor
    {
        return Err(UpgradeValidationError::AbiMismatch {
            current_abi: current.sdk_abi_version,
            candidate_abi: candidate.sdk_abi_version,
            reason: "Major.minor ABI version must match for dual-load upgrade. \
                     Use restart-upgrade for cross-ABI upgrades.".into(),
        });
    }
    
    // 2. Same target triple
    if current.target_triple != candidate.target_triple {
        return Err(UpgradeValidationError::TargetMismatch {
            current: current.target_triple.clone(),
            candidate: candidate.target_triple.clone(),
        });
    }
    
    // 3. Plugin kind must match
    if current.kind != candidate.kind {
        return Err(UpgradeValidationError::KindMismatch {
            current: current.kind.clone(),
            candidate: candidate.kind.clone(),
        });
    }
    
    Ok(())
}
```

When ABI is incompatible, the upgrade is rejected with a clear message directing the operator to use **restart-upgrade** instead.

#### Memory Overhead Management

Dual-load means double memory for that plugin's code + static data. This is acceptable because:
- The dual-load window is finite (typically seconds to minutes per component)
- Most Drasi plugins are Rust-native with small static footprints (< 5MB code + data per plugin)
- Memory is automatically reclaimed when old runtimes drop

> ⚠️ **Caveat:** The POC and recommended Phase 1 approach use `mem::forget` — memory is **never** reclaimed. Old libraries remain mapped for the lifetime of the process. The "automatic reclamation" described above requires Arc-based unloading, which introduces segfault risk (see [Library Unloading Risk Analysis](#library-unloading-risk-analysis)).

```yaml
# Plugin upgrade config
upgrade:
  strategy: rolling     # Dual-load rolling (default)
  # strategy: restart   # Full restart (for constrained environments)
  maxConcurrentMigrations: 2  # Limit concurrency to bound memory
```

For resource-constrained environments, operators can choose `strategy: restart` which stops all dependents → unloads old → loads new → restarts all.

#### Dependency Management During Upgrade

**Strategy: Laissez-Faire** — When a source is being upgraded, its dependent queries and reactions are left running and untouched.

```
Timeline:
─────────────────────────────────────────────────────────────────────
Source:   [running]──[stop]──[swap]──[start]──[running]
                        │                        │
                     gap starts              gap ends
Query:    [running ─────── idle (no events) ─────── running]
Reaction: [running ─────── idle (no results) ────── running]
─────────────────────────────────────────────────────────────────────
```

> ⚠️ **Caveat:** This gap behavior is **identical** to what happens during a restart-based upgrade. The same resilience properties (cursor-based resumption, query idling, reaction leaf-node isolation) apply regardless of whether the source is swapped in-process or restarted with the entire server. This undermines the zero-downtime value proposition — see [Argument Against Hot Upgrades](#argument-against-hot-upgrades).

**Rationale**:
1. Queries are resilient to source interruption — designed to handle sources going offline
2. State store preserves position — source resumes from its last cursor, no events lost
3. No cascade simplifies the operation — fewer failure modes
4. Reactions are leaf nodes — no downstream impact

| Source has state store? | Gap behavior | Data consistency |
|------------------------|--------------|------------------|
| Yes, cursor preserved | Source resumes from last position. Events delivered after brief delay. | **No data loss.** Eventually consistent. |
| Yes, but format changed | Plugin's `migrate_state` hook runs. If fails, source re-bootstraps. | May see duplicate events during catch-up. |
| No state store | Source starts fresh. Events during gap are permanently lost. | **Gap in data.** |

#### UpgradePlan State Machine

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpgradePlan {
    pub id: String,
    pub plugin_kind: PluginKind,
    pub from_version: String,
    pub to_version: String,
    pub new_binary_path: PathBuf,
    pub targets: Vec<UpgradeTarget>,
    pub status: UpgradeStatus,
    pub planned_at: DateTime<Utc>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpgradeTarget {
    pub component_id: String,
    pub component_type: ComponentKind,
    pub status: ComponentUpgradeStatus,
    pub error: Option<String>,
    pub started_at: Option<DateTime<Utc>>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum UpgradeStatus {
    Planned,
    InProgress,
    Completing,
    Complete,
    RollingBack,
    RolledBack,
    Failed { message: String },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum ComponentUpgradeStatus {
    Pending,
    Upgrading,
    Upgraded,
    Failed,
    RolledBack,
    Skipped,
}
```

#### Upgrade Execution Flow

**Phase 1: Plan**

```rust
impl DrasiServer {
    pub async fn plan_upgrade(
        &self,
        plugin_kind: &PluginKind,
        new_binary_path: &Path,
    ) -> Result<UpgradePlan, UpgradeError> {
        // 1. Load metadata from new binary (without activating)
        let new_metadata = load_plugin_metadata(new_binary_path)?;
        
        // 2. Get current plugin metadata
        let current = self.plugin_registry.get(plugin_kind)
            .ok_or(UpgradeError::PluginNotLoaded)?;
        
        // 3. Validate ABI compatibility
        validate_abi_compatibility(&current.metadata, &new_metadata)?;
        
        // 4. Find all dependent components across all instances
        let targets = self.find_dependents(plugin_kind).await?;
        
        // 5. Check no existing upgrade in progress for this plugin
        if self.has_active_upgrade(plugin_kind).await {
            return Err(UpgradeError::UpgradeAlreadyInProgress);
        }
        
        // 6. Create and store UpgradePlan
        let plan = UpgradePlan { /* ... */ };
        self.store_upgrade_plan(&plan).await?;
        
        Ok(plan)
    }
}
```

**Phase 2: Execute**

```rust
impl DrasiServer {
    pub async fn execute_upgrade(
        &self,
        plan_id: &str,
    ) -> Result<UpgradeResult, UpgradeError> {
        let mut plan = self.get_upgrade_plan(plan_id).await?;
        
        // 1. Load the new plugin binary (dual-load)
        let new_handle = self.load_plugin_for_upgrade(&plan).await?;
        
        // 2. Register in registry as primary (old moves to retiring)
        self.plugin_registry.upgrade_plugin(
            plan.plugin_kind.clone(),
            new_handle.descriptor(),
        )?;
        
        // 3. Rolling migration
        plan.status = UpgradeStatus::InProgress;
        plan.started_at = Some(Utc::now());
        
        for target in &mut plan.targets {
            if target.status != ComponentUpgradeStatus::Pending {
                continue; // Skip already-processed (resume after crash)
            }
            
            target.status = ComponentUpgradeStatus::Upgrading;
            target.started_at = Some(Utc::now());
            self.update_upgrade_plan(&plan).await?;
            
            match self.upgrade_component(target, &new_handle).await {
                Ok(()) => {
                    target.status = ComponentUpgradeStatus::Upgraded;
                    target.completed_at = Some(Utc::now());
                }
                Err(e) => {
                    target.status = ComponentUpgradeStatus::Failed;
                    target.error = Some(e.to_string());
                    plan.status = UpgradeStatus::RollingBack;
                    return self.rollback_upgrade(&mut plan).await;
                }
            }
        }
        
        // 4. Complete — old library unloads via Arc automatically
        self.plugin_registry.complete_retirement(&plan.plugin_kind);
        plan.status = UpgradeStatus::Complete;
        plan.completed_at = Some(Utc::now());
        self.update_upgrade_plan(&plan).await?;
        self.update_lockfile(&plan).await?;
        
        Ok(UpgradeResult::from(plan))
    }
    
    async fn upgrade_component(
        &self,
        target: &UpgradeTarget,
        new_handle: &PluginHandle,
    ) -> Result<(), UpgradeError> {
        match target.component_type {
            ComponentKind::Source => {
                let new_source = new_handle.factory
                    .create_source(&target.component_id).await?;
                
                // Uses existing update_source which handles:
                // - Transition to Reconfiguring state
                // - Stop if running → Swap runtime → Restart
                // - Preserve graph node, edges, history
                self.drasi_lib
                    .update_source(&target.component_id, new_source)
                    .await
                    .map_err(|e| UpgradeError::ComponentFailed {
                        component: target.component_id.clone(),
                        error: e.to_string(),
                    })?;
                
                Ok(())
            }
            ComponentKind::Reaction => {
                let new_reaction = new_handle.factory
                    .create_reaction(&target.component_id).await?;
                
                self.drasi_lib
                    .update_reaction(&target.component_id, new_reaction)
                    .await
                    .map_err(|e| UpgradeError::ComponentFailed {
                        component: target.component_id.clone(),
                        error: e.to_string(),
                    })?;
                
                Ok(())
            }
            _ => Err(UpgradeError::UnsupportedComponentType),
        }
    }
}
```

**Phase 3: Rollback**

```rust
impl DrasiServer {
    async fn rollback_upgrade(
        &self,
        plan: &mut UpgradePlan,
    ) -> Result<UpgradeResult, UpgradeError> {
        plan.status = UpgradeStatus::RollingBack;
        self.update_upgrade_plan(plan).await?;
        
        let old_factory = self.plugin_registry
            .get_retiring_factory(&plan.plugin_kind)
            .ok_or(UpgradeError::OldPluginNotAvailable)?;
        
        // Rollback only components that were upgraded
        for target in &mut plan.targets {
            if target.status != ComponentUpgradeStatus::Upgraded {
                continue;
            }
            
            match self.rollback_component(target, old_factory).await {
                Ok(()) => {
                    target.status = ComponentUpgradeStatus::RolledBack;
                }
                Err(e) => {
                    target.error = Some(format!(
                        "ROLLBACK FAILED: {}. Component may be in inconsistent state.", e
                    ));
                }
            }
        }
        
        // Restore registry: move retiring back to primary
        self.plugin_registry.cancel_upgrade(&plan.plugin_kind);
        
        plan.status = UpgradeStatus::RolledBack;
        plan.completed_at = Some(Utc::now());
        self.update_upgrade_plan(plan).await?;
        
        Ok(UpgradeResult::from(plan.clone()))
    }
}
```

**Rollback safety**: The Arc-based lifetime guarantees that the old library is still loaded during rollback — old runtimes still hold handles to it. After rollback completes, the new library's Arc count reaches 0 and is automatically unloaded.

#### Crash Recovery

If the server crashes mid-upgrade:

1. On startup, check for UpgradePlan in ComponentGraph with status `InProgress` or `RollingBack`
2. For `InProgress`:
   - Components marked `Upgraded` are fine (already on new version)
   - Components marked `Pending` haven't been touched (still on old version)
   - Components marked `Upgrading` are indeterminate — check runtime status
3. For `RollingBack`: Resume rollback from where it left off
4. Both old and new binaries must still be on disk (lockfile records both)

**Default crash recovery strategy**: Log the incomplete upgrade, wait for operator to `execute` (resume) or `rollback`.

### API Design

#### REST Endpoints

```
POST /api/v1/plugins/{pluginKind}/upgrade/plan
  Body: { "binaryPath": "/path/to/new-plugin.so" }
  -or-  { "reference": "registry.io/source-postgres:2.1.0" }
  
  Response 200:
  {
    "plan": {
      "id": "upgrade-xxxx",
      "pluginKind": "source-postgres",
      "fromVersion": "1.0.0",
      "toVersion": "2.1.0",
      "targets": [...],
      "status": "Planned"
    }
  }
  
  Response 400: ABI incompatible
  Response 409: Upgrade already in progress


POST /api/v1/plugins/upgrades/{planId}/execute
  Body: { 
    "strategy": "rolling",
    "maxConcurrent": 1,
    "abortOnFailure": true
  }
  
  Response 202 (Accepted):
  {
    "plan": { ... status: "InProgress" ... }
  }


GET /api/v1/plugins/upgrades/{planId}
  Response 200:
  {
    "plan": { ... full plan with per-target status ... }
  }


GET /api/v1/plugins/upgrades
  Response 200:
  {
    "upgrades": [
      { "id": "...", "pluginKind": "...", "status": "InProgress", ... }
    ]
  }


POST /api/v1/plugins/upgrades/{planId}/rollback
  Response 202 (Accepted):
  {
    "plan": { ... status: "RollingBack" ... }
  }


DELETE /api/v1/plugins/upgrades/{planId}
  (Cancels a Planned upgrade that hasn't started executing)
  Response 204
  Response 409: Cannot cancel — already executing


GET /api/v1/plugins/{id}/metrics
  Response 200:
  {
    "loaded_libraries": 2,
    "runtimes_on_current": 8,
    "runtimes_on_retiring": 3,
    "estimated_memory_mb": 12.4
  }
```

#### CLI Commands

```bash
# Plan an upgrade (validates, shows what will change)
drasi plugin upgrade plan source-postgres --binary ./new-source-postgres.so
drasi plugin upgrade plan source-postgres --reference registry.io/source-postgres:2.1.0

# Execute a planned upgrade
drasi plugin upgrade execute <plan-id>
drasi plugin upgrade execute <plan-id> --strategy rolling --max-concurrent 2

# Check status
drasi plugin upgrade status
drasi plugin upgrade status <plan-id>

# Rollback
drasi plugin upgrade rollback <plan-id>

# Cancel (before execution)
drasi plugin upgrade cancel <plan-id>

# Simple one-step (plan + execute, for quick upgrades)
drasi plugin upgrade source-postgres --to 2.1.0 --auto-execute
```

### Alternatives Considered

#### Alternative 1: Restart-Only Upgrade (Status Quo)

**Approach**: Stop server → replace binary → restart server.

**Pros**:
- Zero implementation effort
- No complexity
- Matches industry norms (Terraform, Kubernetes, etc.)

**Cons**:
- Full downtime during restart
- All components restart simultaneously (cold start)
- No partial/gradual migration
- No rollback — once restarted with new binary, reverting requires another restart

**Verdict**: Keep as the default/fallback strategy. A simple `drasi plugin upgrade source-postgres --restart` command should remain available.

#### Alternative 2: Sequential Replace (All-at-Once)

**Approach**: Stop ALL dependents → unload old → load new → restart ALL dependents.

**Pros**:
- No dual-load (avoids symbol conflicts and memory overhead)
- Simple implementation
- Deterministic timing

**Cons**:
- Simultaneous downtime for ALL components using that plugin
- If plugin has many dependents (e.g., 50 sources), downtime could be significant
- No partial progress — all-or-nothing

**Verdict**: Offer as a strategy option (`strategy: "all-at-once"`) for operators who prefer simplicity over availability.

#### Alternative 3: Side-by-Side Indefinite

**Approach**: Keep both versions loaded permanently. New components use new version. Old components keep old version forever.

**Pros**:
- True zero-downtime
- No forced migration
- Gradual natural transition

**Cons**:
- Permanent memory overhead
- Configuration complexity
- No clean "upgrade complete" state
- Plugin bugs in old version never get patched

**Verdict**: Rejected. The temporary dual-load captures the benefit without the cost of permanent dual-load.

#### Alternative 4: Blue-Green Instance

**Approach**: Create new DrasiLib instance with new plugin → clone components → switch traffic → destroy old instance.

**Pros**:
- Clean separation
- Trivial rollback (switch back to old instance)

**Cons**:
- Doubles resource usage during transition
- External clients must reconnect (SSE streams, webhooks)
- Instance cloning is complex
- Queries rebuild their index from scratch (expensive bootstrap)

**Verdict**: Interesting for future consideration once instance management is mature, but heavier than rolling upgrade for most use cases.

## Security

- **Binary validation**: New plugin binaries should be verified (checksum or signature) before loading. Unsigned binaries could be rejected in strict mode.
- **File system permissions**: Plugin binary paths must be restricted to trusted directories to prevent loading arbitrary code.
- **No credential exposure during swap**: The `update_source/reaction` mechanism preserves credentials in the ComponentGraph — they are not re-transmitted during upgrade.

## Compatibility impact

- **Existing plugins**: No changes required for existing plugins. They work as before.
- **Plugin authoring constraint**: Plugins using C libraries with process-global state must be aware that two versions may coexist. This is documented as a plugin authoring guideline.
- **ABI version requirement**: Dual-load upgrade requires ABI-compatible versions. Cross-ABI upgrades fall back to restart strategy.
- **API additions**: All new REST endpoints. No breaking changes to existing API.
- **CLI additions**: New `drasi plugin upgrade` subcommand tree. No breaking changes to existing CLI.

## POC Findings

A proof-of-concept implementation was built on the [`atomic-comp`](https://github.com/drasi-project/drasi-server/tree/atomic-comp) branch. The POC validates the core upgrade workflow and reveals important implementation realities.

### What Was Validated

- **Rolling migration via `update_source`/`update_reaction` works** — The atomic runtime swap mechanism successfully upgrades individual components without disrupting others.
- **ABI compatibility validation** — SDK major.minor matching and target triple checks prevent unsafe upgrades.
- **State machine lifecycle** — The `Planned → InProgress → Complete/RolledBack` state machine operates correctly with proper guard conditions.
- **Component discovery via snapshot** — Using `DrasiLib::snapshot_configuration()` to find dependents is practical and avoids coupling to internal graph metadata.
- **REST API lifecycle** — Full CRUD endpoints for plan/execute/rollback/cancel work end-to-end.
- **Multi-component rolling migration** — Successfully upgraded 3 sources sequentially, each swapping from v1 to v2 runtime.

### POC Architecture

```
src/upgrade/
├── mod.rs          — Module root with re-exports
├── error.rs        — UpgradeError enum (11 variants, thiserror)
├── plan.rs         — State machine: UpgradePlan, UpgradeTarget, status types
├── validation.rs   — ABI compatibility checks (SDK version, target triple)
└── engine.rs       — Core orchestration engine (plan/execute/rollback/cancel)

src/api/v1/
└── upgrade_handlers.rs  — REST API handlers + route builder
```

### Key Divergences from This Design

| This Design Doc Proposes | POC Reality |
|---|---|
| Arc-based library lifetime (automatic `dlclose`) | Old library stays loaded via `mem::forget` — no unloading occurs |
| True rollback with old factory from `retiring` map | Rollback is degraded to stop/restart (uses current registry, which is the *new* version) |
| Crash-safe persistent state in ComponentGraph | Plans stored in-memory `RwLock<HashMap>` only — lost on crash |
| Configurable `maxConcurrentMigrations` | Sequential-only migration; no concurrency implemented |
| `complete_retirement()` cleanup of old library | Not needed since libraries are never unloaded |

### Implications for the Design

1. **The `mem::forget` approach works today.** The host-sdk intentionally leaks loaded libraries, meaning old plugin code remains valid indefinitely in memory. The upgrade workflow functions correctly without Arc-based lifetime management.

2. **Rollback fidelity is limited without a `retiring` map.** The POC's rollback performs stop/restart using the *current* registry (which already points to the new version after `upgrade_plugin()`). True rollback — recreating a component with the old factory — requires maintaining old descriptors in a `retiring` map accessible during the rollback phase.

3. **The upgrade engine code is largely host-sdk-agnostic.** Whether libraries are leaked or Arc-managed, the upgrade engine's plan/execute/rollback logic remains the same. The difference is confined to how `PluginRegistry` manages descriptors and whether memory is eventually reclaimed.

### POC Limitations (Production Gaps)

| Area | POC Limitation | Production Fix |
|------|---------------|----------------|
| Rollback | Best-effort stop/restart (doesn't recreate with old factory) | Maintain `retiring` map of old descriptors in `PluginRegistry` |
| Crash recovery | In-progress plans lost on server restart | Persist `UpgradePlan` to disk; check on startup and prompt operator |
| Concurrency | No lock preventing concurrent upgrades of same plugin | Add upgrade mutex per plugin kind |
| Registry conflict | New plugin replaces old immediately via normal load path | Add `upgrade_plugin()` path to orchestrator that stages the replacement |
| Persistence | Upgraded components not reflected in config persistence | Trigger `save()` after successful upgrade |
| Health checks | No post-upgrade health verification per component | Add configurable health probe / soak period between migrations |
| Batch control | All targets upgraded sequentially | Add `maxConcurrent` parameter to execution |

### Test Results

```
$ cargo test --lib
test result: ok. 306 passed; 0 failed; 0 ignored  (15 new + 291 existing)

$ cargo test --test upgrade_test -- --ignored
test result: ok. 4 passed; 0 failed; 0 ignored
```

Integration tests use a dedicated two-version mock plugin (`tests/fixtures/upgrade_test_plugin/`) built via `make build-upgrade-test-plugins`.

---

## Library Unloading Risk Analysis

The Arc-based library lifetime management proposed in this design enables automatic memory reclamation via `dlclose` when the last reference to an old plugin is dropped. However, moving from the current `mem::forget` approach (never unload) to `dlclose`-on-drop introduces **segfault risk** that must be carefully evaluated.

### Why `mem::forget` Is Safe Today

With `mem::forget`, the shared library is **never unmapped** from the process address space. Every function pointer, vtable entry, and code page remains valid for the lifetime of the process. This eliminates an entire class of undefined behavior at the cost of monotonically growing memory usage.

### Risks Introduced by Library Unloading

#### Risk 1: Background Tasks Outliving Runtime Drop — ⚠️ HIGH

Plugins spawn async tasks (tokio tasks, background threads) that hold function pointers into the library's code segment. If the runtime's `Drop` does not **join** all spawned work before returning, the library can unload while plugin code is still executing:

```rust
impl MySource {
    async fn start(&self) {
        // Plugin spawns a tokio task — code lives in the .so
        tokio::spawn(async move {
            loop { self.poll_changes().await; } // ← fn ptr into library
        });
    }
}

// Drop sequence:
// 1. runtime drops — but spawned task still running on tokio executor
// 2. _handle drops → Arc count = 0 → dlclose()
// 3. Spawned task calls poll_changes() → SEGFAULT (code page unmapped)
```

This is the highest-risk scenario because Drasi plugins routinely spawn background tasks for change polling, event streaming, and connection management.

#### Risk 2: Escaped Trait Objects — ⚠️ MEDIUM

If the host holds a `Box<dyn Stream>`, `Box<dyn Future>`, or any trait object returned by the plugin, the vtable pointer lives in the library's code segment:

```
Host holds: Box<dyn Stream> → vtable ptr → 0x7ff... (in libplugin_v1.so)
                                            ↑
Library unmaps this address → next poll → SEGFAULT
```

Any trait object that crosses the plugin→host boundary is a potential dangling vtable if the library unloads before the host drops the object.

#### Risk 3: Thread-Local / atexit Handlers — ⚠️ LOW

If a plugin or its dependencies (e.g., `tracing`, `log`, `openssl`) register thread-local destructors or `atexit` handlers, `dlclose` causes those destructors to fire pointing at unmapped code when threads exit.

#### Risk 4: Struct Field Reorder Fragility — ⚠️ LOW but insidious

The safety of `PluginRuntime<T>` depends on `runtime` being declared **before** `_handle` (Rust drops fields in declaration order). A well-intentioned refactor that reorders fields silently introduces undefined behavior with no compiler warning:

```rust
pub struct PluginRuntime<T: ?Sized> {
    runtime: Box<T>,       // MUST drop first (releases vtable references)
    _handle: PluginHandle, // MUST drop second (may trigger dlclose)
}
```

### Mitigation Strategies

| Risk | Mitigation | Trade-off |
|------|-----------|-----------|
| Background tasks | Plugin `Drop` must `block_on(shutdown)` — await/join all spawned work before returning | Adds latency to component stop; requires plugin author discipline |
| Escaped trait objects | Plugin API contract: host must drop all plugin-returned objects before calling `update_source` | Hard to enforce at compile time; requires careful API design |
| Thread-locals | Use `RTLD_NODELETE` flag (keeps code pages mapped, releases only data segments) | Partially defeats the purpose of unloading |
| Field reorder | Use `ManuallyDrop` + explicit drop order in `Drop` impl; add `// SAFETY:` comments | Adds implementation complexity |
| Grace period | After all Arc references drop, wait N seconds before allowing `dlclose` | Reduces (but doesn't eliminate) risk from straggler tasks |

---

## Argument Against Hot Upgrades

Given the POC findings, library unloading risks, and the overall complexity surface, there is a strong case that **Drasi should not support hot plugin upgrades at all** and should instead invest in making restart-based upgrades fast and seamless.

### The Complexity-to-Value Ratio Is Unfavorable

Hot upgrades introduce substantial machinery — a state machine, crash recovery, a `retiring` registry, ABI validation, rollback coordination, and new API/CLI surface — to solve a problem that affects a narrow operational window. The actual downtime during a restart-based upgrade of a single Drasi server is measured in **seconds** (process restart + plugin load + source reconnect). The engineering investment to shave those seconds to zero is disproportionate:

| Aspect | Hot Upgrade | Fast Restart |
|--------|------------|--------------|
| New code surface | ~2,000+ lines (engine, API, state machine, validation) | ~50 lines (graceful shutdown + ordered startup) |
| Failure modes | Partial migration, rollback failures, stuck components, indeterminate crash states | Clean: either the server is up or it isn't |
| Testing burden | Multi-version mock plugins, crash recovery tests, concurrent operation tests, ASAN/MSAN for unloading | Standard integration tests |
| Plugin author burden | Must ensure clean shutdown, no escaped references, no global state conflicts | None — plugins are restarted cleanly |
| Operational complexity | Operators must understand plan/execute/rollback lifecycle, monitor per-component status | `drasi server restart` or container orchestrator handles it |

### Sources Are Already Designed for Interruption

Drasi's architecture already handles source disconnection gracefully:

- **Queries are resilient to source gaps** — they idle when no events arrive and resume when the source reconnects.
- **Cursor-based state stores preserve position** — sources resume from their last checkpoint after restart, delivering missed events with no data loss.
- **Reactions are leaf nodes** — they simply wait for results and have no downstream cascading impact.

This means the "zero-downtime" benefit of hot upgrades is largely **already provided by the existing architecture**. A restarted source reconnects, catches up from its cursor, and downstream queries/reactions see a brief delay — the same behavior described in this document's "Laissez-Faire" dependency management strategy during a rolling upgrade.

### The Segfault Risk Is Not Eliminable

As documented in the Library Unloading Risk Analysis, `dlclose`-based unloading introduces an entire class of undefined behavior that is:

- **Not detectable at compile time** — escaped function pointers, background tasks, and vtable lifetimes cannot be statically verified.
- **Not fully under our control** — plugin authors link arbitrary third-party crates that may register thread-locals, global callbacks, or spawn OS threads.
- **Catastrophic when triggered** — a segfault kills the entire server process, affecting all instances, all sources, and all queries. A single bad plugin upgrade can cause total system failure.

Even with the `mem::forget` mitigation (never unload), the hot upgrade path still carries risk from the `retiring` map, partial migration states, and rollback failures that don't exist with a clean restart.

### Conclusion

Hot plugin upgrades are a **nice-to-have** feature that introduces **must-not-have** risk characteristics. The operational benefit (avoiding a few seconds of source interruption) does not justify the complexity, testing burden, and failure-mode surface area. Drasi's existing architecture — cursor-based resumption and query resilience to source gaps — already provides the availability guarantees that hot upgrades aim to deliver.

---

## Alternative: Registry-Driven Upgrade UX

Rather than hot-swapping plugins at runtime, a more pragmatic approach is to invest in **upgrade UX** — making it trivial for operators to discover, download, and stage new plugin versions, with the actual upgrade applied on the next server restart.

### User Experience

The operator's workflow becomes:

1. **Browse** available plugin updates from a registry (via VS Code extension, CLI, or web UI)
2. **Download** the new binary to the server's plugin directory (staged, not yet active)
3. **Review** what will change — which components use this plugin, what version they're on
4. **Restart** the server at a convenient time — new plugin activates automatically on startup

This separates the **decision** (which version to run) from the **mechanism** (restarting the server), giving operators full control over timing without requiring complex in-process machinery.

### VS Code Extension Integration

The Drasi VS Code extension could provide a plugin management panel:

```
┌─────────────────────────────────────────────────────┐
│  Drasi Plugins                                       │
├─────────────────────────────────────────────────────┤
│  ● source-postgres    v1.0.0  →  v2.1.0 available  │
│    [View Changelog]  [Download]                      │
│                                                      │
│  ● source-mongodb     v1.2.0  (up to date)          │
│                                                      │
│  ● reaction-webhook   v0.9.0  →  v1.0.0 available  │
│    [View Changelog]  [Download]                      │
│                                                      │
│  Staged Updates: 1 pending                           │
│    source-postgres v2.1.0 — applies on next restart  │
│                                                      │
│  [Restart Server Now]  [Schedule Restart]            │
└─────────────────────────────────────────────────────┘
```

### CLI Equivalent

```bash
# Browse available updates
drasi plugin list --check-updates

# Download (stage) a new version
drasi plugin download source-postgres --version 2.1.0

# See what's staged
drasi plugin status

# Apply (restart)
drasi server restart
```

### Why This Is Better Than Hot Upgrades

| Concern | Hot Upgrade | Registry + Restart |
|---------|------------|-------------------|
| Complexity | State machine, rollback, crash recovery, dual-load | File download + restart |
| Risk | Segfaults, partial migration, stuck states | Clean process restart — no new failure modes |
| Rollback | Complex (needs `retiring` map, old factory) | Keep old binary on disk, restart with it |
| Discoverability | Operator must know binary path or registry reference | UI shows available updates with changelogs |
| Timing control | Upgrade happens immediately (pressure to get it right) | Operator chooses when to restart |
| Validation | ABI check at upgrade time | ABI check at download time — fail early, before restart |

### Registry Design (Sketch)

A plugin registry serves metadata and binaries:

```
GET /v1/plugins/source-postgres/versions
→ [{ "version": "2.1.0", "sdk_abi": "0.6", "changelog": "...", "checksum": "sha256:..." }]

GET /v1/plugins/source-postgres/versions/2.1.0/binary?target=aarch64-apple-darwin
→ (binary download)
```

The server's plugin directory would have a `staged/` subdirectory:

```
plugins/
├── active/
│   ├── source-postgres-1.0.0.so
│   └── reaction-webhook-0.9.0.so
└── staged/
    └── source-postgres-2.1.0.so    ← downloaded, not yet active
```

On startup, the server checks `staged/` — if a newer version of a plugin exists there, it promotes it to `active/` and loads it. The old binary is moved to `archive/` for rollback.

### Startup Promotion Logic

```rust
// On server startup, before loading plugins:
for staged_binary in scan_staged_dir()? {
    let metadata = load_plugin_metadata(&staged_binary)?;
    let active_path = active_dir.join(&metadata.filename());
    
    // Validate ABI compatibility with server
    validate_server_abi(&metadata)?;
    
    // Archive current version for rollback
    if let Some(current) = find_active_plugin(&metadata.kind) {
        archive_plugin(&current)?;
    }
    
    // Promote staged → active
    fs::rename(&staged_binary, &active_path)?;
    log::info!("Promoted {} v{}", metadata.kind, metadata.version);
}
```

### Rollback

```bash
# If something goes wrong after restart:
drasi plugin rollback source-postgres
# → restores archived v1.0.0 to active/, requires another restart

drasi server restart
```

This is simpler than in-process rollback because the server is in a clean state — no partial migrations, no mixed-version components, no `retiring` maps.


