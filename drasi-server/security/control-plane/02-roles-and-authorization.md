# Drasi Server Roles and Authorization

**Status:** Draft

## Overview

Authorization checks whether a Principal may perform an action on a resource or scope. Fixed roles bundle permissions. Ownership supplies defined authority through a separate relationship. These are parts of one model, not competing approaches.

Owner is a permanent relationship between a Principal and a component, not an assignable role. The Principal may represent a person or automation service. Sources, Queries, and Reactions are the owned resources.

This design defines the permissions, assignment rules, and enforcement behavior for shared Drasi Server instances. It does not introduce individually assignable permissions, custom roles, uploaded policies, explicit-deny bindings, or per-record access control. Backward compatibility and migration are not requirements.

## Scope and responsibilities

| Concern | Responsibility |
|---|---|
| Establish caller identity | [Authentication](01-authentication.md) supplies a validated Principal and credential lifetime; login alone grants no resource access |
| Decide and enforce authority | This design governs Server requests, component sharing, and continuing runtime input access |
| Persist resources and authority | Control-state design commits desired state, ownership, bindings, and dependency identities consistently; runtime is not the authority store |
| Supply workload credentials | Instance credential design; credential availability is independent of component permissions |
| Protect application listeners and delivery | Ingress/egress designs integrate with this authority; they do not create separate permission stores |
| Admit native code and administer the host | Separate Code & Plugin Plane and host-administration contracts; API roles do not confer OS authority |

Drasi Server owns the fixed policy and authoritative role bindings. DrasiLib and SDK integration carry the context and enforcement hooks required for Server-managed execution. Applications using DrasiLib directly are not required to adopt Server roles or OIDC.

Missing security integration must fail closed for protected Server-managed operations, not silently become an embedded-mode bypass. The host and admitted native plugins remain trusted and unsandboxed; logical instances are not adversarial process boundaries.

## Identity, resources, and bindings

| Term | Meaning |
|---|---|
| Principal | Exact OIDC issuer plus subject. Display names and email addresses carry no authority |
| Group selector | Exact group value from the configured trusted issuer/claim, not a local membership database |
| Resource identity | Immutable internal UID. A component target includes instance UID, component type, and component UID |
| Readable ID | A selector for resolving a resource, not its authority identity |
| Permission | An action allowed on a resource or scope, subject to the conditions below |
| Role binding | A fixed role assigned to an exact Principal or permitted group selector at a valid target |
| Owner relationship | Exactly one permanent Principal associated with a Source, Query, or Reaction |
| Scoped administrator | Server Admin across instances, or Instance Admin within its assigned instance |

Permissions are additive within their valid scope; no applicable authority means denial. Reject unknown roles/actions and incompatible role, target, or grantee combinations. There are no wildcard identities, resource-name globs, or label-based grants. Explicit development mode supplies the fixed development Principal defined by authentication, through the same authorization policy.

Store resource relationships and role targets by UID, not just readable names. Updating a component preserves its UID and readable ID; deleting and recreating the same name creates a new UID and inherits no ownership, bindings, or dependency authority. Runtime graph containment edges are not human ownership.

## Permissions

These names define semantic actions, not a requirement for a particular Rust enum or storage schema. A role binding supplies a bundle; callers cannot assign individual actions directly.

| Action | Meaning and boundary |
|---|---|
| `view` | Read safe summaries and permitted discovery links; not configuration, diagnostics, or data |
| `read-config` | Read accepted declarative configuration, including typed credential references but never resolved values |
| `read-diagnostics` | Read bounded, data-free status details, errors, schema, events, and operational metadata |
| `read-bindings` | Inspect the full role-binding list for an authorized scope/component |
| `manage-bindings` | Assign/revoke permitted fixed roles within the delegator's scope; cannot assign Owner or elevate outside that scope |
| `create` | Create an instance under Server scope, or a component under instance scope, with the required input authority |
| `update` | Change accepted component configuration while preserving identity and Owner, subject to stopped-update and Owner input checks |
| `delete` | Remove an authorized target and retire its associated authority; does not authorize deleting another user's dependent resources |
| `start`, `stop` | Change intended execution of accepted configuration; not permission to edit it |
| `reset-state` | If exposed, reset processing state without changing configuration, identity, ownership, or permissions; processing semantics are separately owned |
| `read-results` | Read a Query's results, snapshots, and streams through direct Query APIs |
| `use` | Make a Source an input to a Query, or a Query an input to a Reaction |
| `ingest` | Submit events through a Source's supported ingestion operations; no component discovery, reads, or dependency `use` |
| `consume` | Access a Reaction's application and exposed results, including supported shared application-data mutations |
| `manage-credentials` | Administer instance credential profiles and their availability, without revealing resolved values or overriding host admission/external IAM |
| `read-idp-identifiers` | List shared workload IDP identifiers available in the relevant instance; not profile definitions, resolved credential values, or a vault-wide Secret listing |
| `read-catalog` | Browse available component kinds, configuration schemas, and host-published templates for authorized creation/editing; not live deployment inventory or component configuration |

**`use` and `consume` are not synonymous.** Query Consumer supplies Bob with `use` on Q for a Reaction he is otherwise authorized to create. Reaction Consumer supplies Carol with `consume` on Bob's Dashboard. Carol does not need Bob's input permissions and gains no direct access to Q's API.

Apply and template deployment enforce the permissions of each underlying operation. Export requires `read-config` on every included component. There are no separate Apply, Deploy, Export, or credential-use roles.

## Fixed roles

`P` means an exact user/service Principal; `G` means an exact trusted group selector. Component roles target one exact component, never a name pattern.

| Role | Valid target | Grantee | Permissions |
|---|---|---|---|
| Server Admin | Server | P/G | All applicable actions defined above across instances; exclusively permits instance creation/deletion and Server Admin binding administration |
| Instance Admin | One instance | P/G | All applicable actions defined above within its instance, including credential profiles and permitted instance/component bindings; no instance creation/deletion or Server Admin binding administration |
| Instance Author | One instance | P/G | `create` components, `view` all ordinary component summaries in that instance, `read-catalog`, and `read-idp-identifiers` |
| Operator | One component | P/G | `view`, `start`, `stop` |
| Source User | One Source | P only | `view`, `use` as a Query input |
| Source Ingestor | One Source | P/G | `ingest` only |
| Query Reader | One Query | P/G | `view`, `read-results` |
| Query Consumer | One Query | P only | `view`, `read-results`, `use` as a Reaction input |
| Reaction Consumer | One Reaction | P/G | `view`, `consume` |

Administrative permissions remain subject to resource type, scope, host restrictions, and Owner-based input checks. Background `use` from an administrator binding counts only when that binding targets the Owner's exact Principal, not a group.

Author alone cannot edit or operate another user's component. Operator is not a troubleshooting role. There is no Viewer, Instance Viewer, Connection User, or assignable Owner role.

**Ingress dependency:** this document retains the approved submission-only meaning of `ingest`. The pending ingress proposal broadens it to Source application reads and mutations. That requires an explicit amendment; an existing plugin's CRUD routes must not silently expand Source Ingestor authority.

### Permanent ownership

An authenticated create assigns the creator as the component's sole Owner, whether the creator is a person or service. Remote configuration, templates, and imports cannot nominate someone else. Trusted initialization may specify exact Owners; it must not introduce ownerless user components or substitute an administrator execution approval.

| Owned component | Authority supplied by ownership |
|---|---|
| Every Source, Query, or Reaction | `view`, `read-config`, `read-diagnostics`, `read-bindings`, permitted `manage-bindings`, `update`, `start`, `stop`, `delete`, `reset-state` if exposed, and `read-catalog` for editing |
| Source | Additionally `use`, supported `ingest`, and `read-idp-identifiers`; select available instance credential references |
| Query | Additionally `read-results` and `use` |
| Reaction | Additionally `consume` and `read-idp-identifiers`; select available instance credential references |

There is no co-ownership, transfer, Owner binding, or ownership-recovery operation. A scoped administrator may maintain an unavailable Owner's component without taking ownership. Different ownership requires a new component.

Removing Instance Author does not remove ownership. Existing Owners can still edit their components; existing Source/Reaction Owners can still select available instance credentials. Ownership does not grant creation of other components.

### Role-binding administration

Only Server Admin may assign/revoke Server Admin. Instance Admin may administer Instance Admin and Instance Author bindings in its instance, never Server authority. Scoped administrators may manage permitted component bindings.

An Owner may assign/revoke only:

| Owned resource | Delegable roles |
|---|---|
| Source | Operator, Source User, Source Ingestor |
| Query | Operator, Query Reader, Query Consumer |
| Reaction | Operator, Reaction Consumer |

Other non-admin roles cannot delegate or inspect full binding lists. Groups are valid only where the role table permits them; Source User and Query Consumer remain subject-only.

The role-binding interface must support listing, assignment, and revocation with an exact target, fixed role, and typed Principal/group selector. The server resolves selectors, validates delegation, and authorizes against current state at commit. Clients cannot supply arbitrary permission lists, change ownership through a binding, or bypass checks through bulk operations. Wire names and endpoint layout are implementation choices.

Bindings persist until explicitly revoked or their target is retired. The assigning actor is provenance, not a continuing dependency: removing Pat's administrator role does not recursively revoke bindings Pat previously created.

## Input authority and autonomous execution

A Query's permanent Owner must retain `use` on every input Source. A Reaction's permanent Owner must retain `use` on every input Query. Input authority comes from owning the input or applicable role bindings targeting that Owner's exact Principal, including direct scoped-administrator bindings.

Groups authorize current requests using validated claims and current bindings. They do not supply background input authority. Do not retain human tokens, persist group-membership snapshots, poll IdP membership, or create special execution approvals.

**Example:** Bob receives Instance Author through the `analysts` group and Source User directly on Alice's Source S. He creates Q using S. Q continues after Bob logs out, but loses eligibility if Bob loses his last effective `use` on S. Pat's group-derived Instance Admin role permits editing Q, not substituting Pat's input permissions for Bob's.

Source/Query `use` permits derivation and redistribution through authorized downstream components. Drasi does not propagate upstream row-level permissions, prevent external copying, or require additional upstream grants from downstream readers.

Credential availability is separate: Authors and existing Source/Reaction Owners may exercise the shared instance credential authority under the credential design, but this does not grant access to another component. They may reproduce equivalent external data access through a different Source. Components, including those edited by Server Admin, select only their own instance's credential context.

## Operations and continuing access

| Operation | Required authorization behavior |
|---|---|
| Create | Instance Author or scoped admin; validate inputs against the creator's qualifying authority; commit identity, Owner, references, and Stopped intent before runtime effects |
| Update | Owner or scoped admin; check every resulting input against the unchanged Owner, not the editor; target and operational dependents must be stopped and remain stopped |
| Start/stop | Owner, scoped admin, or Operator on the target; Start additionally requires continuing Owner input authority and credential availability |
| Share/revoke | Validate role-target-grantee compatibility and delegation scope; commit the binding change and apply its effects to current access |
| Delete/recreate | Require target authority, retire resource-linked bindings/references consistently, and never transfer old authority to a new UID |
| Apply/import/template | Check each underlying create/update; no publisher authority, implicit role assignments, or imported ownership; new components belong to the caller |
| Export | Require configuration-read authority for every included component; reject unauthorized export rather than silently substituting partial summaries |

Authorization must hold at mutation commit. An administrative edit that needs additional Owner input authority must be preceded by an explicit ordinary role assignment; the edit cannot manufacture that authority.

Updates preserve ordinary bindings. Adding customer emails to a shared Query may expose them to its existing readers after restart. Editors must review that audience; there is no per-revision reader consent.

Create/update must not construct runtime components, resolve Secrets, connect externally, or bootstrap data. Accepted state is the source of truth. Trusted seed input is applied once; restart must not replay initial bindings over revocations.

### Reaction applications

`consume` authorizes the Reaction's accepted application surface, including snapshots, cached results, multiplexed streams, and supported application-data changes while running. Dashboard consumers may create, edit, and delete dashboards, layouts, and widgets, including items created by other consumers. There are no per-item Owners or separate application reader/editor roles.

This does not permit component configuration/diagnostics, full sharing lists, binding changes, lifecycle, component deletion, or direct input-Query API access. Changing subscriptions, credentials, destinations, or listener settings remains component administration regardless of the route or RPC name.

Caller `consume` and the Reaction Owner's continuing input `use` are both necessary. A cache or an already-established session cannot bypass either requirement.

### Revocation and offboarding

Always evaluate effective authority: another applicable binding or ownership may preserve access.

| Change | Effect |
|---|---|
| Caller loses Query result-reading authority | Reject further direct reads and close that caller's direct Query streams; do not stop Q or independently revoke Reaction consumption |
| Caller loses Reaction `consume` | Reject further application requests/results and close affected sessions/streams; do not stop the Reaction for others |
| Caller loses Source `ingest` | Reject further submissions and close affected ingestion streams; do not stop the Source for others |
| Caller loses visibility or Operator | Restrict/close affected summary subscriptions or deny further operations; losing Operator does not undo an earlier Start |
| Owner loses last required input `use` | Block the affected consumer and dependent flow; retained results or subscriptions must not bypass the lost authority; leave unrelated bindings and shared credential profiles unchanged |
| Credential profile becomes unavailable | Apply the credential design's acquisition and dependent-component restrictions; potentially affects multiple components |
| IdP-only group/account changes | Follow credential lifetime; fresh tokens supply current claims, not real-time IdP synchronization |

Local binding changes affect active access even with an otherwise valid token. Protected streams end at credential expiration, including while idle. Loss of runtime authority preserves requested Running intent; valid restored authority may allow automatic resumption. Restart rechecks current authority.

Removing authoring rights or disabling an IdP account does not automatically retire owned pipelines. Offboarding requires explicit binding/component/profile administration. Stop is not permanent withdrawal while an authorized Owner or Operator can restart. No rollback of committed work, recall of delivered data, or universal termination of external sessions is promised.

## Discovery, configuration, and diagnostics

| Surface | Visibility rule |
|---|---|
| Instance/component summaries | Server Admin sees all; Instance Admin/Author sees ordinary summaries in its instance; component roles see only their targets, except Ingestor has no discovery |
| Parent navigation and graphs | Component-only access permits minimal parent navigation, not sibling counts; show Source-to-Query-to-Reaction links only when both endpoints are visible |
| Full configuration, diagnostics, binding lists | Owner or scoped admin only; not implied by summary, input `use`, Operator, or data-reading access |
| Authoring catalogues | Instance Authors, existing component Owners, and scoped admins for permitted creation/editing; login or data/operation roles alone are insufficient |
| Shared IDP identifiers | Require `read-idp-identifiers` for the relevant instance, supplied by Instance Author, Source/Reaction ownership in that instance, or scoped administration; Query ownership or `read-catalog` alone does not grant it |

A safe summary contains readable ID, type/kind or Query language, permanent Owner, and status. It excludes query text, endpoints, results, credential references, full bindings, and raw errors. Apply the same visibility to lists, counts, graphs, and summary events. Do not expose credential profiles or Secrets as shared graph nodes, edges, or dependency counts.

Safe catalogues contain admitted kinds, configuration schemas, and host-published recipes, not live plugin paths, deployment-wide usage, actual component configurations, or credential-profile definitions. Catalogue access does not permit publishing templates or admitting code. OpenAPI access follows the authentication design, but must not bypass catalogue restrictions by embedding otherwise restricted installed-kind/schema information.

Internal/system resources remain administrator-only unless an explicitly defined operation exposes a narrower result. A name prefix alone must not establish internal status. Diagnostics must be bounded and data-free; arbitrary payload-bearing plugin logs are not made safe by applying a generic redactor.

Configuration reads and portable exports retain typed Secret names and IDP references, never resolved values. Configurations come from accepted declarative state, not reconstructed runtime properties. Exports exclude ownership, bindings, internal UIDs, profile definitions, workload identities, plugin trust, and processing state. Import reauthorizes references in the destination instance.

## Enforcement contract

```text
Authenticated caller + action + resolved resource/scope
                         |
          Current role bindings and Owner relationship
                         |
       Allow only with applicable operation preconditions
                         |
         Commit desired state / release authorized view

Runtime work: accepted component + permanent Owner's current input authority
```

Enforce this contract across explicit-instance routes, default-instance aliases, programmatically constructed Server APIs, composite operations, plugin application endpoints, cached responses, and long-lived streams. Resolve aliases to the same exact target; they are not separate grants.

If current authority cannot be established, deny protected operations. Do not fall back to permissive plugin defaults, stale permission copies, or initial seed bindings.

The current Query attach path creates an internal application Reaction. Reading a Query stream must still require only the Query's `read-results`, not Author, Query Consumer, or a new shareable component. Any retained helper is transient, scoped to that caller's read operation, excluded from ordinary persistence/sharing, and terminated when access ends. Agents may replace the helper rather than preserve its implementation.

Runtime dependencies must be checked against accepted identities and current authority, not merely a construction-time list of readable IDs. SDK/FFI bridges must preserve bound instance/component scope and connect to host authority rather than maintain independent policy copies. Revocation must affect cached output as well as fresh fetches; checks only at Start or stream handshake are insufficient.

The UI must not fetch full configurations to render summaries or treat hidden buttons as enforcement. UI affordances may reflect effective permissions, but the server remains authoritative.

Authentication failures/unavailability follow the [authentication failure contract](01-authentication.md#http-surface-and-failure-contract): `401` for missing, malformed, expired, or invalid credentials; `503` when verification is temporarily unavailable and no usable trusted keys remain. Verification unavailability is not an authorization denial or proof that a credential is invalid. Authorization denial is `403` for a visible target and `404` for a hidden or missing target. Global/scoped actions without a hidden target may return `403`. Errors and validation failures must not disclose protected configuration, hidden dependencies, or credentials.

## Audit and security dependencies

Audit binding changes, component/profile mutations, denials, lifecycle actions, exports, sensitive data access, application mutations, and session/stream opening/closing under the shared audit contract. Include action, target identities, role/grantee for binding changes, relevant accepted-state revisions, and outcome. Attribute an edit to its actual caller, not the unchanged Owner. Automatic work uses system/component attribution; host operations use host identity.

Never include resolved credentials, tokens, full request/configuration bodies, or protected payloads/results. No durable audit record per result row or delivery is required.

During a known audit-only outage, block new audit-required operations; existing authorized pipelines may continue under normal checks. Authorized Stop and permission revocation remain available with explicit audit-failure reporting. A committed change whose audit write fails remains committed and is reported incomplete, not rolled back. Sink implementation and readiness reporting belong to the audit design.

## Threats, risks, and decisions

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Login mistaken for resource authority | Logged-in Bob edits Alice's Source | Unauthorized configuration changes | Check action, exact target, and current role/Owner authority on every protected path |
| Cross-instance or recreated-name confusion | A new `orders` Source inherits the old one's permissions | Access to a different resource | UID-bound targets/dependencies; no inheritance across deletion/recreation; aliases resolve exact scope |
| Ownership forged or transferred through configuration | Bob imports YAML naming Alice as Owner | Misattributed authority and accountability | Creator becomes permanent Owner; no client-supplied Owner, Owner binding, or transfer |
| Delegation exceeds the caller's authority | Source Owner assigns Instance Admin | Privilege escalation | Fixed valid role/target/grantee combinations and explicit delegation rules |
| Administrator lends input rights unintentionally | Pat adds a private Source to Bob's Query using Pat's rights | Processing Bob is not entitled to perform | Validate resulting inputs against Bob; require explicit ordinary bindings before editing |
| Group claims become permanent runtime authority | A saved group-membership snapshot keeps a pipeline authorized | Stale or ambiguous background rights | Ownership/subject-bound input authority; no retained JWT, membership snapshot, or special approval |
| Stream/cache outlives permission | Carol keeps receiving cached Dashboard results after revocation | Continued disclosure | Recheck current authority and terminate affected access, including idle streams and cached paths |
| Internal read adapter becomes a privilege bypass | Query Reader obtains a persistent, independently usable attach Reaction | Unintended creation or data access | Internal helper inherits only the exact read operation's scope/lifetime, not user-component authority |
| Discovery becomes configuration access | Component list loads everyone's `view=full` response | Endpoint, query, or credential-reference disclosure | Separate safe summaries from config/diagnostics/results; enforce on API, counts, graphs, and UI fetching |
| Application mutation becomes administration | A Dashboard request changes subscribed Queries or credentials | Bypass of editing/input checks | `consume` covers application state, not component/security configuration |
| Updated configuration widens an existing audience | Editor adds customer email to a shared Query | Additional data reaches existing readers | Accepted: bindings follow component revisions; editor reviews exposure, no per-revision consent |
| Shared credentials defeat assumed upstream exclusivity | Bob creates another Source against Alice's database | Equivalent external data access | Accepted trusted-instance authority; Drasi permissions protect components, not exclusive upstream data access |
| Consumer changes another consumer's application item | Carol deletes Bob's Dashboard layout | Loss of shared application content | Accepted coarse `consume`; no per-item ownership or reader/editor split |
| Offboarding leaves effective access | Author role is removed, but Bob still owns a Reaction | Continuing editing/execution authority | Explicit offboarding of remaining bindings/components; permanent ownership and grantor-independent bindings are deliberate |
| Restart revives a revoked binding | Startup reapplies an old seed role assignment | Restoration of withdrawn privilege | Seed once; committed state remains authoritative; recheck authority before execution |
| Audit failure conceals a successful mutation | API fails after a binding was saved | Misleading outcome and missing audit evidence | Report committed-but-incomplete; restrict new audit-required work while retaining authorized containment |
| Host or admitted native code is compromised | A native plugin reads process memory | Bypass of logical instance boundaries | Accepted trusted-process boundary; native admission and deployment isolation are separately owned |

## Verification and implementation boundaries

Implementers must exercise the policy as behavior, not only unit-test role names:

| Case | Required result |
|---|---|
| Each role on valid/invalid scope, type, and grantee | Only defined actions succeed; additive alternatives work; invalid combinations are rejected |
| Person/service Owner, admin editor, Operator | Distinct powers and actual-actor audit; no ownership transfer or input-right substitution |
| Group author/admin versus subject-bound input authority | Group-based requests work; background authority needs qualifying direct bindings or ownership |
| Default routes, composite calls, programmatic Server, plugin listeners | Same exact-target policy; no alternate entry-point bypass |
| Summary/config/diagnostic/result/catalogue access | Correct projections and denials; no hidden counts, full-config prefetch, payload logs, or credential values |
| Shared IDP identifier discovery | Authors, scoped admins, and Source/Reaction Owners may list identifiers within their applicable instance scope; Query ownership or `read-catalog` alone is insufficient; no profile definitions, resolved values, or vault-wide Secret listing |
| Direct Query attach versus Reaction consumption | Reader needs no authoring rights; consumer needs no upstream grants; neither gains the other's direct API authority |
| Revocation during idle/active streams and cached reads | Affected access ends; unrelated users/pipelines remain authorized; current Owner input loss blocks dependent work |
| Update, recreation, import, and restart | Sharing persists only across identity-preserving updates; no transferred authority or seed replay |
| Credential unavailability and audit outage | Apply the stated failure/containment rules without weaker authorization or misleading success |

This work adds the role-binding and ownership model, shared authorization integration, protected API/UI views, and continuing-access enforcement. It depends on authenticated caller context and authoritative control-state commits. Exact REST shapes, Rust modules, policy evaluator, synchronization, storage layout, and SDK/FFI signatures are implementation choices; they must not invent new permissions or weaken these contracts.

Processing correctness/recovery, native code admission, listener transport, provider proofs, and credential acquisition mechanics remain separately owned. No new state-reset feature, user directory, destination-grant system, or per-tenant quota framework is required by this document.

## Alternatives considered

Individually assigned actions or custom policies create a second configuration surface; fixed roles suffice. Assignable/co-owned/transferable ownership adds delegation and recovery rules; permanent ownership plus scoped administration is the selected model. Group-backed background authority needs membership-lifetime machinery; direct bindings avoid it. Requiring Dashboard consumers to hold every input permission would defeat deliberate downstream sharing.

## References

- [Drasi Server authentication](01-authentication.md)
- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
