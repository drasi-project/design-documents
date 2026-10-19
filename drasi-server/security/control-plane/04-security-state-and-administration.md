# Drasi Server Security State and Administration

**Status:** Draft

## Overview

V1 starts with a simple, local control-state and administration model. We will expand it when justified by user demand, rather than introduce additional management machinery in anticipation of it. Backward compatibility, user migration, and data migration are not requirements.

Drasi Server will commit what resources exist, who controls them, and what they are intended to do before applying runtime changes. DrasiLib supplies the runtime projection of that accepted state; live components and their inspection output do not reconstruct configuration or authority.

This design defines state ownership, initialization, mutation and startup contracts, and local host administration. It describes target behavior, not the current implementation.

## Scope and dependencies

| Concern | Owning contract |
|---|---|
| API caller identity, login trust, and browser credentials | [Authentication](01-authentication.md) |
| Roles, permanent ownership, bindings, input `use`, visibility, and revocation | [Roles and authorization](02-roles-and-authorization.md) |
| Host authentication options, instance providers, credential references, and provider lifecycle | [Instance credentials](03-instance-credentials.md) |
| Durable authority, configuration sources, initialization, and host access recovery | This design |
| Audit events and outage behavior | Apply the shared policy recorded in [02](02-roles-and-authorization.md#audit-and-security-dependencies); audit sink implementation is separate |
| Native artifacts, listeners, delivery, and processing recovery | Separate Code & Plugin, ingress, egress, and processing designs |

Server-managed execution must follow this contract across CLI startup, programmatic Server construction, default-instance aliases, and composite API operations. Ordinary embedded DrasiLib applications are not required to adopt Server administration or its control store.

## Terms and state boundaries

| Term or state | Meaning and authority |
|---|---|
| Deployment | One initialized Server installation with a stable identity and protected local state. One active Server owns its control state at a time |
| Host administrator | A person or automation trusted through deployment/OS access, not another Drasi API role |
| Host startup configuration | Explicit deployment inputs read at startup, such as login trust, public-client settings, and operational paths. It does not reapply managed resources or bindings |
| Seed input | Trusted initialization input, potentially YAML. It establishes initial managed state once; it is not a live configuration source afterward |
| Control state | Committed resource identities, accepted configuration, ownership, bindings, references, provider availability, and intended execution. This is the authority for Server-managed behavior |
| Runtime state | Constructed objects, current status, and runtime observations derived from control state. A missing runtime object does not mean the saved resource is absent |
| Processing state | Events, indexes, checkpoints, outboxes, results, and plugin state. It cannot establish permissions or replace accepted configuration |
| Audit | Records of actors, actions, decisions, and outcomes. It is not a replacement control-state database |

Host authentication options are named definitions of permitted credential-acquisition mechanisms, as defined in 03. They are distinct from login OIDC settings, instance Secret Store/IDP configurations, and the credential values acquired at runtime.

### One mandatory Server-owned control store

Use one built-in, local transactional control store per deployment. It must persist accepted state independently of runtime construction and independently of the optional processing `StateStoreProvider`. There is no `persistConfig: false` equivalent for managed Server authority, including private development mode.

The store must retain:

- Deployment identity, initialization state, authentication mode, and revisions identifying accepted state.
- Instance and component UIDs, readable IDs, types, and parent-instance relationships.
- Original accepted declarative configuration, each user component's permanent Owner, and role bindings.
- Identity-bound data dependencies and provider references; typed external Secret names remain references, not new Secret resources.
- Instance Secret Store and shared IDP definitions and their administratively enabled/disabled state.
- Host-provided authentication option definitions established at initialization.
- Each component's intended execution state: Running or Stopped, independently of observed runtime status.

Related records must commit consistently. A component must not appear without its required identity/Owner/reference state, nor a binding without a valid target. Runtime events, metrics, and query results must not rewrite this authority.

Persist accepted configuration, not a snapshot of runtime `properties()`, a redacted API response, or resolved values. Preserve reference semantics and accepted non-secret settings; preserving YAML comments or textual formatting is not required. Unserializable required state must reject the mutation, not be silently omitted.

Resolved credentials stay outside control state under 03. An optional Secret Store may legitimately use an operator-protected local file or keyring; that does not make it the control store or permit credentials to be copied into that store.

### Deployment ownership and host protection

The host establishes the deployment's state location and protects its files and directory through OS/deployment controls. API roles cannot select arbitrary control-state paths or edit the store directly. Missing files must not silently redirect Server startup to another location or a temporary in-memory store.

The running Server holds exclusive deployment ownership. Initialization and offline maintenance must also acquire exclusive access and refuse to race an active Server or another maintenance command. A supervisor must not restart the Server into an offline operation.

This is local coordination, not a distributed control plane or protection against the host administrator. The host and admitted native code remain trusted; no tamper-proof state or process-isolation guarantee is introduced.

## Initialization and configuration sources

### Separate startup settings from seed-once authority

| Input category | Treatment |
|---|---|
| Explicit host startup settings, including login issuer/audience and UI client settings | Read through the host-controlled startup contract; resource APIs and hidden overrides cannot redefine them |
| Instances, components, exact Owners, initial bindings, and instance provider configurations | Import during initialization; subsequent managed changes use authorized operations against committed state |
| Host authentication options | Import and commit during initialization; no post-initialization editing workflow in V1 |
| Non-secret environment substitutions in managed seed configuration | Resolve during initialization and commit the accepted values; do not re-resolve them silently at restart |
| Typed Secret and shared-IDP references | Preserve as references; acquire values only through the runtime credential contract |
| Runtime host credential provisioning or an explicitly selected SDK discovery chain | Follow 03; do not persist acquired values or mistake initialization-time substitution for a fixed-identity guarantee |

A host-enabled SDK chain may select a different identity when its environment or login context changes. That is the explicit credential-policy tradeoff in 03, not permission to reinterpret saved component configuration or enable a previously unavailable mechanism.

Normal startup must not replay seed resources, Owners, bindings, or provider definitions. Editing the seed file is not an update operation. Ordinary API updates must not overwrite the host's seed or startup configuration.

### Explicit initialization

Initialization must target an uninitialized deployment and hold exclusive access. It must validate the complete proposed initial state, establish deployment identity, resolve references, and commit a consistent initialized state before any managed runtime construction.

Require at least one explicitly configured Server Admin Principal or permitted group binding for production initialization. Seeded user components require exact Owners under 02; there is no ownerless component or administrator execution-approval substitute. Validate applicable Owner input authority and provider-reference rules without acquiring credentials.

Initialization explicitly selects production or private development mode. Development uses the stable fixed Principal from 01. The selection is committed, not inferred from localhost, failed login, missing configuration, or an ordinary restart flag. Changing modes requires fresh initialization; this does not relax development's credential or authorization rules.

Initial component intent may be declared by trusted seed input; absent intent defaults to Stopped. Initialization does not execute that intent. API-created components always begin Stopped under 02, and normal startup may activate only eligible Running-intent components.

Initialization must refuse to overwrite an initialized deployment. An interrupted or failed attempt must not leave partially established authority usable as a completed deployment. A committed initialization with a later audit failure is reported as committed but incomplete, not as permission to overwrite it on retry.

Validation is not activation: validate syntax, references, role/Owner rules, and available safe schema metadata without constructing providers/components, resolving Secrets, obtaining tokens, contacting external systems, or loading native code merely to validate. Report validation limits explicitly; structural acceptance does not certify external credential usability.

## Mutations and runtime effects

```text
Authenticated API caller
        |
Resolve target + validate proposed state + check current authority
        |
Atomically commit accepted control state
        |
Apply runtime effects using current committed authority
        |
Report saved state, runtime outcome, and required audit outcome

Offline host recovery uses host authority, not a fabricated API caller.
```

### Consistent admission and commit

Authorize the actual caller and the complete resulting operation under 02/03. Administrative edits preserve Owner-based input checks. Resolve names to the correct identities before committing relationships; a recreated name cannot inherit its predecessor's authority.

Authorization must still hold at commit. Concurrent operations must not publish partial authority or let an earlier check overwrite an intervening revocation. Stale configuration replacements must conflict rather than silently erase newer accepted changes. The implementation may choose its revision and synchronization scheme; it must identify the accepted state and preserve these guarantees.

A failed control-state commit must not apply the requested runtime mutation or publish new permissions. There is no successful in-memory-only change followed by optional persistence. If trustworthy current authority cannot be established, deny protected work rather than fall back to runtime metadata or initial seed bindings.

Do not hold a control-store transaction across plugin construction, external I/O, lifecycle calls, or audit-sink writes. Runtime effects are separate from the durable commit; before applying them, ensure the target identity and accepted revision remain applicable. Stale work or late events must not restart a retired/reconfigured resource or restore withdrawn authority.

### Operation effects

This table defines state effects, not another permission catalogue. Actors and preconditions remain those in 02 and 03.

| Operation | Control-state and runtime boundary |
|---|---|
| Create | Commit identity, accepted configuration, Owner, references, and Stopped intent together; no runtime construction or external acquisition |
| Update | Preserve identity, Owner, and ordinary bindings; validate the new revision and saved references; enforce explicit stopped preconditions and leave affected resources Stopped |
| Binding mutation | Commit the exact assignment/revocation consistently and apply its effect to current access; do not treat a copied permission cache as independent authority |
| Provider mutation | Use saved dependencies, including stopped consumers, Secret-backed IDPs, and Source bootstrap references; preserve 03's distinct replacement, referenced-deletion, disable, and re-enable rules |
| Start/stop | Commit the requested intent and then apply permitted runtime effects; Start requires current authority and credential availability. Stop is not a grant revocation |
| Delete | Enforce target authority and saved-reference preconditions; retire identity and linked authority consistently. Do not implicitly delete another user's dependent resource or transfer authority to a replacement |
| Processing-state reset, if exposed | Preserve configuration, identity, ownership, bindings, and provider references. It is not a control-state reset or a new feature required by this design |

Being Blocked is not equivalent to being explicitly Stopped for a configuration/provider change. Disabling a provider or losing Owner input authority can block execution while retaining Running intent. Re-enabling an unchanged provider or restoring qualifying authority may resume that intent; explicit Stopped intent remains Stopped.

Read configuration, resource inventory, and saved dependencies from accepted state, with runtime observations added separately and filtered by 02. This includes resources whose factories cannot currently construct a runtime. Runtime graph containment is not human ownership; internal read helpers do not acquire persistent user-resource authority.

### Completion and audit

| Outcome | Caller-visible meaning |
|---|---|
| Not committed | No requested control-state change was accepted and no corresponding runtime mutation was applied |
| Committed and complete | The change was accepted and the operation's required runtime/audit phases completed |
| Committed, completion incomplete | Accepted state remains saved, but a required later phase failed; identify the failed phase and actual runtime outcome without disclosing protected details |

These outcomes apply to each mutation. A composite operation must disclose any committed changes rather than label them uncommitted because a later entry failed.

A failed activation or audit write does not roll back a committed configuration or resurrect a revoked binding. Do not report a failed stop as a stopped runtime. After an interrupted or ambiguous response, callers must be able to inspect authorized accepted state before retrying; the response contract must distinguish commit outcome from activation outcome. Exact response fields remain implementation choices.

Apply the audit-outage rules in 02, including its narrow Stop/revocation exceptions. Those exceptions do not permit uncommitted mutations when the control store cannot accept a change. Host OS process control remains available outside the API model; it does not persist a new component intent.

Audit uses the actual API actor, host identity for local maintenance, or system/component attribution for automatic work. An existing Owner is not impersonated by a reconciler. Audit storage and detailed outage/readiness implementation remain separately owned.

## Normal startup and runtime availability

Normal startup must open an existing initialized deployment under exclusive ownership, validate its identity and supported format, and establish trustworthy security state before protected execution. Missing, unreadable, incompatible, or inconsistent control state must not create an empty replacement, select older authority automatically, invent an administrator, or enable development mode.

Load saved resources independently of runtime activation. Establish the applicable authentication and authorization contracts, then reconcile eligible Running-intent components against current permissions, provider availability, and other operation preconditions. Stopped resources must not activate merely because a runtime plugin defaults to `autoStart`.

An individual component or provider activation failure must preserve its accepted configuration and authorized management visibility. It must not, by itself, prevent the management API from operating or unrelated eligible work from running. A valid reference to an unavailable provider is different from a corrupt authority record.

Authentication-service availability follows 01, including usable cached signing keys and fail-closed requests when verification is unavailable. It is not permission to erase state or stop pipelines merely because their creator cannot log in. Audit-only outages follow 02 rather than being treated as control-state corruption.

Server shutdown and runtime errors must not rewrite Running intent, ownership, or grants. Restart rechecks current authority rather than trusting a prior Running status. This contract does not specify processing recovery, delivery guarantees, or retry algorithms.

## Local host administration

Seed YAML is input to initialization, not a second live administration channel. A user-facing CLI that calls Server APIs follows ordinary OIDC/role checks. A host-maintenance CLI runs locally under protected OS/deployment access and records the observed host actor; it does not create a second account or role system.

| Workflow | V1 contract |
|---|---|
| Validate input | Read-only structural/reference checks with the side-effect limits above; no credential-resolution preview or deployment mutation |
| Initialize | Create one consistent deployment from trusted input; explicit authority, exclusive access, no overwrite of initialized state |
| Run | Open committed state and reconcile only eligible intent; no seed replay or automatic initialization |
| Inspect deployment | Local read-only diagnostics without resolving or dumping credentials; does not repair or rewrite authority |
| Recover API administration | Offline, narrowly add Server Admin authority for an exact Principal, as described below |

The host protects CLI access and deployment files. Access to a CLI executable alone is not a security boundary, and an API Server Admin role does not confer host-maintenance authority. Routine resource, role-binding, and instance-profile changes continue through the authenticated API; no general offline editor is introduced.

### Additive access recovery

Recovery requires the Server stopped and exclusive deployment access. The host supplies an exact issuer/subject compatible with the configured login trust, not a display name, password, or claimed OIDC session. Commit an ordinary Server Admin binding for that Principal and record the observed host identity and recovery outcome.

Preserve existing administrator bindings, component Owners, resources, provider definitions, and intent. The operation does not transfer ownership, reset authentication, replace all administrators, or grant an arbitrary role. If the exact binding already exists, report that fact without duplicating it.

Access recovery requires valid readable control state; it is not a corrupt-store repair tool. A broken host login configuration must be corrected through its host-controlled startup contract, not bypassed with an anonymous endpoint or default password. Recovery does not acquire or validate a live token on the target's behalf.

The shared audit contract applies: recovery is not an additional audit-outage exemption for granting authority. If a grant commits and subsequent audit fails, report committed-but-incomplete accurately; do not imply that existing authority was reset.

## Deferred host maintenance

**Decided direction, deferred beyond V1:** support local host administration through a CLI while the Server runs. The CLI will use an OS-protected local channel to the running Server, which validates, commits, applies, and audits the change. It must not become a concurrent direct writer to the Server's control store or a remotely exposed host-administration API.

This does not promise unrestricted hot reconfiguration. Operations retain reference and lifecycle preconditions; changing an in-use credential-acquisition mechanism may require explicitly stopping affected components, without requiring every unrelated workload to stop. Exact commands and local communication mechanisms are deferred.

V1 therefore has no workflow for adding, redefining, or removing host authentication options after initialization. Changing those definitions requires fresh initialization, not editing seed YAML and restarting. This does not defer ordinary instance-profile administration under 03, external Secret rotation, or runtime acquisition through an already permitted mechanism. Local credential-backend access remains subject to 03's host-provisioning boundary. Narrow offline access recovery remains in scope.

## Threats, risks, and decisions

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Runtime becomes the authority store | A save reconstructs configuration from a running plugin's properties | Lost references, exposed credentials, or authority inconsistent with accepted intent | Persist original accepted configuration and security records independently of runtime; never reconstruct them from inspection |
| Partial security mutation | A component appears without its Owner, or a binding points to an uncommitted resource | Ownerless resources or unintended access | Commit related identity, configuration, ownership, binding, and reference changes consistently |
| Stale edit restores authority | A request checked before revocation commits an old permission snapshot afterward | Revoked access becomes usable again | Current authorization at commit, concurrency protection, and conflicts for stale replacements |
| Persistence is bypassed | An API-enabled builder runs with optional in-memory configuration | Privileges and intent disappear or change on restart | Mandatory control store for all Server-managed entry points, independent of embedded Lib and processing storage |
| Seed replay restores access | A revoked initial administrator returns on restart | Restoration of withdrawn privilege | Apply seed once; normal startup and API saves do not replay or rewrite it |
| Restart silently changes configuration | A seed environment value now names another resource | Accepted identity/configuration changes without a mutation | Resolve non-secret managed substitutions once; keep deliberate runtime credential discovery separate |
| Missing state enables a default deployment | A wrong path causes startup to create a new unsecured configuration | Unauthenticated access or invented administration | Explicit initialization; no empty-store, alternate-path, in-memory, or development fallback |
| Two writers disagree | Recovery edits grants while the Server serves cached authority | Lost updates and inconsistent enforcement | Exclusive deployment ownership; offline V1 recovery, no concurrent store editor |
| Unavailable runtime hides a saved resource | A Source cannot construct because its credentials are unavailable | Administrators lose visibility or ability to correct accepted configuration | Inventory/configuration comes from control state; block affected activation rather than discard the resource |
| Old runtime work survives replacement | A delayed Start applies to a recreated readable name | Execution under retired identity or configuration | Bind effects to applicable accepted identity/revision and current authority; late events cannot rewrite intent |
| Provider maintenance misses saved consumers | A disabled Source or Secret-backed IDP is absent from runtime inspection | Unsafe deletion or unexpected credential retargeting | Use saved direct/indirect references and 03's explicit stopped/replacement rules |
| Processing cleanup erases permissions | A state reset clears the store containing role bindings | Loss or recreation of control authority | Separate control and processing stores; reset/cleanup cannot authorize changes to security state |
| Recovery becomes a general bypass | A local command impersonates an API admin or resets every Owner | Unattributed privilege changes or ownership takeover | Local host attribution, exact additive Server Admin grant, preserved unrelated state; no anonymous recovery endpoint |
| Error response hides a committed change | Audit fails after a new binding commits | Caller assumes no change and retries incorrectly | Explicit commit/completion outcomes; authoritative inspection before retry, no false rollback |
| Audit containment bypasses persistence | A Stop exception is used to make unsaved administrative changes | Restart disagrees with reported authority or intent | Audit exceptions do not relax durable control-state requirements |
| Host or admitted code tampers with state | A privileged operator replaces the database or executable | Defeat of the control-plane boundary | Accepted trusted-host/native-code boundary; OS protection and honest limits, not a tamper-proof claim |
| Static host options limit maintenance | A deployment needs another authentication mechanism after initialization | Operational flexibility | Accepted V1 limit: fresh initialization; bounded local online administration is a recorded deferred direction |

## Verification and implementation boundaries

| Case | Required result |
|---|---|
| Fresh initialization, missing initial admin/Owner, repeated initialization | Only complete valid authority initializes; existing deployments are not overwritten |
| Missing/corrupt/incompatible state or wrong deployment location | No protected execution through replacement state, seed replay, or invented defaults |
| Concurrent mutations and revocation during admission | No partial authority, lost revocation, or stale accepted replacement |
| Commit failure versus post-commit activation/audit failure | No uncommitted effects; committed outcomes remain accurately inspectable |
| Stopped or unconstructable component/provider | Saved identity/configuration remains available through authorized management views |
| Restart after grants, intent, or provider availability change | Use current committed state; no old seed permissions or automatic activation of Stopped intent |
| Seed substitutions versus host-enabled runtime discovery | Managed literals remain stable; runtime credentials follow 03 without being persisted |
| Referenced provider replacement/deletion and stopped versus Blocked consumers | Apply saved-dependency and lifecycle rules even without runtime objects |
| Offline access recovery and concurrent Server startup | Exclusive access, exact additive binding, host audit, preserved unrelated state |
| Programmatic Server, aliases, composite operations, and internal read helpers | Same authority boundary; no persistence opt-out or accidental helper ownership/persistence |
| Reset/cleanup and local Secret Store backends | No control-authority deletion or credential copying into control/processing state |

Current integration points include Server configuration loading, `ConfigPersistence`, shared mutation handlers, initialization commands, and instance construction, plus DrasiLib's component graph and lifecycle interfaces. Its graph-first registration is reusable runtime infrastructure, not a durable security transaction. Existing runtime-property snapshots and plugin processing stores cannot substitute for the accepted-state contract.

Database choice, record versus snapshot layout, revision representation, locking, Rust modules, SDK/FFI interfaces, CLI spelling, and response serialization remain implementation choices. They must satisfy these contracts and the owning documents rather than introduce new permissions or management surfaces.

No automatic migration, backup/restore workflow, distributed state service, general offline editor, or processing-recovery engine is required. Native plugin preparation/activation remains outside this design.

## Alternatives considered

Saving runtime snapshots preserves the wrong authority direction and cannot reliably retain declarative references without leaking resolved values. Reusing optional plugin state storage couples security to processing cleanup and optional persistence. Replaying YAML on every startup can restore revoked access. General offline editing and live host IPC add unnecessary V1 maintenance surfaces; narrow initialization/recovery and a deferred bounded CLI are sufficient.

## References

- [Drasi Server authentication](01-authentication.md)
- [Drasi Server roles and authorization](02-roles-and-authorization.md)
- [Drasi Server instance credentials](03-instance-credentials.md)
- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
