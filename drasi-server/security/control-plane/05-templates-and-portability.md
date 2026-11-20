# Drasi Server Templates and Portability

**Status:** Draft

## Overview

V1 starts with a simple recipe and configuration-portability model. We will expand it when justified by user demand, rather than introduce additional deployment machinery in anticipation of it. Backward compatibility, user migration, and data migration are not requirements.

Host-published templates describe reusable sets of Sources, Queries, and Reactions. Export produces portable component configuration; import and template deployment use ordinary authorized create/update operations. Configuration can travel, but ownership, permissions, credentials, and deployment authority do not.

This design defines target behavior, not the current implementation. It covers the portable content, parameter, and operation contracts without creating another permission system or a separate runtime deployment engine.

## Scope and dependencies

| Concern | Owning contract |
|---|---|
| Caller identity and authenticated API access | [Authentication](01-authentication.md) |
| Catalog visibility, configuration reads, permanent ownership, and create/update authority | [Roles and authorization](02-roles-and-authorization.md) |
| Typed Secret/IDP references and destination-instance credential availability | [Instance credentials](03-instance-credentials.md) |
| Accepted configuration, current authority at commit, stopped-update rules, and mutation outcomes | [Security state and administration](04-security-state-and-administration.md) |
| Portable content, template parameters, selection, and composite Apply behavior | This design |

Native plugin admission, host provisioning, application/result export, backup/restore, and processing recovery remain outside this design. Reaction output-formatting templates are a different feature; their runtime rendering behavior is not defined here.

## Terms and operation permissions

| Operation or artifact | Meaning | Required authority |
|---|---|---|
| Template catalog | Host-published recipes with descriptive metadata, component declarations, and optional typed parameters | Host authority publishes or changes catalog files; API roles do not confer publication authority |
| List/view templates | Browse recipes and their parameter/component descriptions | `read-catalog` under 02; login, data-reading, or Operator access alone is insufficient |
| Deploy template | Expand a selected catalog recipe and Apply its component configuration to one existing instance | Catalog access plus the authority required by each resulting create/update |
| Portable bundle | Component declarations and readable references for one destination instance, without deployment authority or runtime state | Content alone grants nothing |
| Export | Produce a portable bundle for all ordinary components in one instance or an explicit component selection | `read-config` for every included component |
| Apply/import | Create missing components and update matching existing components from a portable bundle | Ordinary create/update permissions and preconditions for each entry |
| Copy | Export from the source instance, then Apply to the destination | Both source-side read authority and destination-side mutation authority |

These are operation contracts, not new roles or individually assignable permissions. Ownership and bindings follow 02. A component Owner may update an existing target without Instance Author, but cannot use a template to create additional components without creation authority.

The target instance must already exist. Creating an instance is a separate Server Admin operation; a UI wizard may compose the operations but cannot lend authority or treat them as one atomic deployment.

## Portable content

Export reads original accepted configuration from Server control state, never reconstructed runtime properties, resolved credentials, or a redacted inspection response. Stopped, Blocked, or currently unconstructable components remain exportable to authorized callers.

| Content | Treatment |
|---|---|
| Sources, Queries, and Reactions | Include selected ordinary component declarations, readable IDs, kinds/languages, and supported non-secret settings |
| Source/Query dependencies | Preserve readable references; resolve them to destination identities during Apply |
| Secret names and shared-IDP references | Preserve typed references, not acquired values or provider definitions |
| Source bootstrap configuration | Inline settings travel with their Source; named bootstrap references remain references |
| Required plugin kinds | May describe prerequisites; never authorize artifact installation, code loading, or trust changes |
| Template metadata and parameters | Catalog recipes may declare descriptive metadata, typed parameters, and defaults; metadata such as author/version carries no authority |
| Resource UIDs, Owners, bindings, or security revisions | Exclude; do not transfer source-deployment authority or concurrency tokens |
| Instance/provider definitions and host settings | Exclude instance creation/settings, Secret Store/shared-IDP definitions, host authentication options, login trust, host paths/permissions, and native artifact/trust configuration |
| Execution and application state | Exclude Running intent, runtime status, processing data, Query results, checkpoints, and application data such as dashboard contents |
| Resolved credentials | Never include, including inside nested plugin settings, headers, connection strings, or bootstrap configuration |

An ordinary portable bundle is not a deployment seed. Reject attempts to supply protected authority or deployment-control fields rather than silently treating import as initialization. Catalog publication by a trusted host does not exempt a recipe from this content boundary.

Normal non-secret component settings, including configured remote endpoints and output-format strings, remain configuration. Their validation follows the owning component/security contract. A reference to a separately approved host input/output is not permission to create or expand that host approval.

Application state is not component configuration: copying a Dashboard Reaction can copy its accepted subscriptions and settings, not the dashboards users created through its application. Copying a Source does not copy its rows or bootstrap data.

### Selection and external prerequisites

A whole-instance export includes all ordinary user components, not internal/system helpers. An explicit selection includes exactly the requested components. Internal helpers are not portable resources.

Require configuration-read authority for every requested member. If any member is missing or unauthorized, fail the export under 02's disclosure rules; do not silently return only the visible subset or skip a failed serialization.

Do not automatically include dependencies. A selected Query may reference a Source outside the selection; the destination must provide an appropriately authorized input or the caller must adjust the reference. Exporting the Query does not reveal the Source's configuration or grant access to it.

Named bootstrap definitions are likewise not expanded or copied automatically. Inline bootstrap configuration is part of its parent Source and retains the same typed credential and host-access restrictions. This distinction does not introduce separately owned bootstrap resources.

**Example:** Alice exports Query `high-orders`, which references Source `orders`. The bundle contains the Query and the name `orders`, not that Source's configuration. On import, `orders` resolves in the destination instance, and the new Query's Owner must have qualifying `use` there.

Secrets, IDPs, named bootstrap definitions, and other omitted prerequisites are established separately in the destination. The same readable name may refer to different external authority or data there. Portability does not guarantee equivalent credentials, external access, or results, and it does not provision missing dependencies.

## Host-published templates

The API may list, view, and deploy catalog recipes. It must not publish, edit, or delete catalog templates, write exported components into the host catalog, or accept uploaded/inline template recipes for rendering.

Ordinary resource import remains allowed: callers may submit concrete portable component configuration through Apply. That is not a template-publication or arbitrary template-execution path. Export returns content to the authorized caller; the host may separately review and publish a recipe derived from it.

Template IDs select entries within the configured catalog, not arbitrary filesystem paths or remote URLs. Catalog content must satisfy the safe-catalog boundary in 02. Do not populate shared recipes by exposing live component configurations or deployment credential settings.

A deployment expands one selected recipe into ordinary accepted configuration. Later catalog edits or removal do not modify deployed components. Deliberate redeployment uses the same create/update rules as import; there is no persistent template installation, upgrade controller, or automatic reconciliation against the catalog.

Required kinds must be available under the host's code-admission contract. Missing prerequisites must be reported, not repaired by downloading or loading native code during deployment. Safe metadata may support validation without constructing plugin runtimes.

## Typed parameters, not a scripting language

V1 supports declared string, number, and boolean parameters, with optional defaults and required-value checks. Values come only from the deployment request or declared defaults. Reject unknown parameters, missing required values, and values inconsistent with their declared type or bounds before any component mutation.

Parameters may replace scalar values or be embedded in string values, including query text and readable reference names. They must not inject document structure or keys. String interpolation is not permission to introduce additional components, authority fields, or a second interpretation pass.

| Illustrative parameter | Type | Intended use |
|---|---|---|
| `threshold` | Number | Insert a numeric threshold into query text |
| `sourceName` | String | Select a readable Source ID/reference, subject to ordinary destination validation |
| `passwordSecretName` | String | Supply the name inside a typed Secret reference, never its value |

The exact declaration and placeholder syntax are implementation choices. The contract is value-only substitution within a parsed structure, followed by validation of the complete resulting component configuration. If substitution occurs inside a query string, the resulting query still undergoes ordinary query/configuration validation; this is not a general query-parameter or information-flow system.

No process-environment lookup, secret resolution, executable expressions, includes, loops, recursive expansion, or arbitrary YAML fragments are supported. Text that belongs to a component's own output-formatting language must not be evaluated as a deployment program.

Parameters are resolved once at deployment. Persist the resulting non-secret configuration and typed references, not a live dependency on the recipe, its parameter expressions, or the host environment. Export produces that accepted configuration rather than regenerating a parameterized recipe. Missing external Secret values are checked through runtime activation, not by fetching them during rendering or validation.

## One Apply path

```text
Host catalog recipe + typed parameters      Portable component bundle
                    |                                  |
             Expand and validate                       |
                    +----------------+-----------------+
                                     |
                       Authorized sequential Apply
                                     |
                     Commit each accepted create/update
                                     |
                        Components remain Stopped

Export: authorized accepted configuration -> portable component bundle
```

### Admission and matching

Validate the document shape, permitted content, parameter expansion, and unambiguous component declarations before mutations. Reject duplicate entries for the same target. Process prerequisites before consumers; each reference must resolve within the destination's current accepted state when its entry commits.

For each entry:

- If the target is absent, require ordinary creation authority and input `use`; assign a new UID and the caller as permanent Owner.
- If the same-type readable target exists, perform an ordinary configuration update. Preserve its UID, readable ID, Owner, and bindings; validate resulting input authority against that Owner, not the importing editor.
- Require existing targets and operational dependents explicitly Stopped under 02/04. Blocked is not Stopped. Apply must not silently stop running components to make an update succeed.
- Enforce current authorization, reference validity, and stale-update protection at commit. Preliminary validation does not reserve permission or authorize rebinding to a resource recreated in the meantime.

Apply is non-pruning. Omitted resources remain unchanged. Do not silently skip conflicting entries, rename them, change resource type, delete/recreate an existing target, take ownership, or issue missing grants. An administrator needing additional input authority for an unchanged Owner must use ordinary binding administration separately.

**Example:** Pat imports a Query whose ID already belongs to Bob. Pat needs scoped administration to update it, Bob remains Owner, and Bob must have every required input permission. Import is not a way for Pat to acquire the Query or lend it Pat's personal input rights.

### Sequential commits and honest outcomes

Apply is not an all-or-nothing bundle transaction. Each successful entry commits consistently under 04; stop on the first entry that fails or commits with incomplete required completion. Earlier commits remain. Do not perform compensating deletes or claim rollback.

The response identifies committed entries, the failed or committed-but-incomplete entry, unattempted entries, and the accepted-state revisions needed to understand the outcome. Follow 04's per-mutation outcomes, including post-commit audit failures. Do not describe an entire request as uncommitted merely because its last entry failed.

Authorization changes and unavailable security services affect later entries through the normal rules. Do not introduce a second permission cache, a batch-long authority reservation, or an exception to audit-outage policy.

Create/update must not construct runtimes, acquire credentials, connect externally, or bootstrap data. Accepted components remain Stopped regardless of plugin defaults. Starting the resulting pipeline is a separate authorized operation.

After partial completion or an interrupted response, callers inspect authorized accepted state before deciding what to retry. No automatic resume service, distributed transaction, rollback engine, or exactly-once Apply guarantee is introduced.

### Copy and destination identity

Copy is Export followed by Apply, not an independent clone endpoint or authority path. A UI may retain a copy workflow, but it must compose the same checks and outcomes.

New destination components receive new UIDs and caller ownership. Matching existing components are updates and retain their current identity and Owner. A caller wanting distinct copies must supply distinct readable IDs and corresponding references; there is no automatic renaming or ownership transfer.

## Safety, audit, and failure boundaries

Bound document size, component count, nesting, parameter values, and rendered/serialized expansion. Reject over-limit content explicitly; never truncate it into a valid configuration or silently omit entries. Numeric limits and parser/accounting choices remain implementation details, not a per-user quota system.

Host-approved recipes are not permission grants, and request parameters/imported content remain untrusted. Validate nested plugin configuration as well as the bundle envelope. Unsupported credentials or authority cannot be smuggled through an opaque map or alternate field.

Use 01/02's authentication, visibility, and error contracts. Errors may identify the caller's failed entry without revealing hidden resources, credential values, raw configuration, or host file paths.

Audit export and constituent mutations under the shared policy, with the actual caller, relevant identities/revisions, and honest partial outcomes. Template identifiers may provide context; publisher metadata does not replace the acting Principal. Do not log full bundles, raw parameter maps, configurations, or credentials. An audit outage does not make deployment or export a containment exception.

## Threats, risks, and decisions

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Recipe lends publisher authority | Bob deploys an administrator's recipe to read a private Source | Unauthorized data use | Each resulting operation uses Bob's ordinary authority; metadata supplies no grants |
| Remote publication changes a shared recipe | An API caller saves a modified template into the host catalog | Other users deploy attacker-controlled configuration | Host-only publication; export returns content, not a catalog write |
| Import becomes trusted initialization | A bundle includes an Owner, admin binding, or host authentication option | Privilege escalation or authority replacement | Component-only content; reject seed/security fields and reauthorize destination references |
| Export expands into hidden configuration | Exporting Q silently includes its private input Source | Disclosure without configuration-read authority | Explicit selection, no dependency expansion, authorization for every included component |
| Runtime snapshot exposes acquired credentials | Export serializes a plugin's resolved password or loses its IDP reference | Credential disclosure or changed authentication | Export committed declarative configuration; preserve typed references, never runtime properties |
| Existing target changes ownership or borrows editor rights | Pat imports over Bob's Query using Pat's input access | Ownership takeover or unauthorized processing | Ordinary update preserves Bob and checks Bob's input authority |
| Earlier validation survives revocation | Bob loses creation/input access halfway through Apply | Later unauthorized commits | Current authority and identity/revision checks for every mutation; stop on failure |
| Parameter becomes YAML or host access | A parameter adds a new mapping or resolves a process variable | Unintended configuration or host credential use | Declared bounded scalars, value-only substitution, no environment lookup or recursive evaluation |
| Deployment starts external work implicitly | A template includes `autoStart` or triggers a factory while validating | Effects before approved activation | No imported Running intent; commit stopped configuration without runtime construction |
| Partial Apply is reported as rolled back | Source creation commits, Query update fails, and the caller retries blindly | Unexpected saved resources or repeated changes | Report committed/failed/unattempted entries; no rollback claim or automatic retry guarantee |
| Template selection escapes the catalog | A caller supplies a path instead of a template ID | Host-file disclosure | Confine selection to host-published catalog entries; no arbitrary file/URL loading |
| Destination names imply false equivalence | `orders-login` uses a different external identity in the destination | Different data or external authority than expected | Accepted portability limit: validate local references; operators establish destination prerequisites |
| Updated shared components expose changed data | Import adds email fields to an existing shared Query | Broader disclosure to existing readers | Accepted under 02: bindings persist across updates; authorized editor reviews the audience |
| Large input or expansion exhausts resources | Many declarations or oversized parameter values expand a bundle | Availability of the control plane | Bound input, expansion, and serialization; explicit rejection, no quota framework |

## Verification and implementation boundaries

| Case | Required result |
|---|---|
| Catalog browsing, publication attempts, and arbitrary template paths | Enforce `read-catalog`, host-only publication, and catalog confinement |
| Whole-instance/selected export with one unauthorized member | No unauthorized or silently filtered export; ordinary components only |
| Export of stopped or unconstructable components | Accepted configuration and references survive without invoking runtime inspection |
| Omitted inputs, named/inline bootstrap, and destination IDPs | Preserve selection boundaries; validate destination references; do not copy shared definitions or acquire Secrets |
| Typed parameters, query-string substitution, and nested output templates | Correct scalar behavior; no YAML/ENV/secret-value injection or second-pass evaluation |
| Create versus existing-target update, including admin editing | New Owner only on create; preserved identity/bindings and unchanged-Owner input checks on update |
| Running/Blocked targets, missing kinds, and invalid credentials in content | No implicit stop, code installation, runtime construction, or credential-resolution preview |
| Commit failure, audit failure, revocation, and stale targets during Apply | Per-entry consistency, stop-on-failure, accurate partial results, no privilege reservation or rollback fiction |
| Copy, redeploy, catalog changes, and Server restart | Ordinary Apply semantics; no transferred authority, template replay, or activation of stopped resources |
| Errors, audit, and size limits | Bounded work and protected diagnostics without raw configuration or credentials |

Reuse Server resource-validation and mutation services for Apply; template deployment must not retain a separate factory-driven path. Existing solution catalog handlers, UI workflows, snapshot exports, and instance cloning are integration points, not behavior that must be preserved. Remove API catalog publication and the separate clone path; replace runtime-snapshot export with accepted-state export.

DrasiLib supplies runtime facilities but does not own this catalog, portable format, or authorization model. The mandatory Server control store and shared security services from 01-04 are prerequisites, not subsystems to reproduce inside a template engine.

Rust types, parsing libraries, UI layout, and internal synchronization remain implementation choices. They must preserve the behaviors above without adding permissions, deployment controllers, or host-management surfaces.

### API Design

One implementation task must define and document a single shared API contract for export, Apply/import, and template deployment. It must cover HTTP methods and paths, bundle and parameter schemas, destination revision preconditions, and response fields/status codes for committed, failed, committed-but-incomplete, and unattempted entries, with request/response examples.

Server, UI, and other clients must use that contract rather than independently inventing interfaces. These wire details will be settled during implementation, not in this informal specification, and must preserve the authorization, reference, lifecycle, and partial-completion behavior defined here.

## Alternatives considered

Automatic dependency closure makes a small export unexpectedly disclose or copy other configuration; explicit selection and destination prerequisites are simpler. Raw YAML/environment interpolation conflates request values with document structure and host authority. All-or-nothing deployment and compensating rollback add guarantees this design does not require. Independent clone and template deployment engines duplicate mutation policy; ordinary Apply keeps those paths consistent.

## References

- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
