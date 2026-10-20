# Drasi Server Instance Credentials

**Status:** Draft

## Overview

V1 starts with a simple trusted-administrator model. Host-enabled production discovery, shared host authentication options for Instance Admins, and supported token-target overrides are deliberate simplicity choices. More restrictive controls are deferred until justified by concrete user or scenario demand, not promised for a later version. Backward compatibility, user migration, and data migration are not requirements.

Drasi Server will give each DrasiLib instance its own credential context: one optional Secret Store and multiple named, shared workload identity-provider configurations. Instance Authors and existing Source/Reaction Owners may use the credentials made available to their instance. They are trusted integration developers, not mutually adversarial tenants.

The host establishes available authentication mechanisms. Instance Admins configure instance providers using those mechanisms. Components reference Secrets or named identity providers; API users do not gain permission to read resolved credential values. This document defines the target behavior, not the current implementation.

## Terms and boundaries

| Term | Meaning |
|---|---|
| Workload identity | The identity used by Drasi to authenticate to an external system, not the human configuring the component |
| Host authentication option | A host-provided, named configuration selecting an explicit identity mechanism or a supported SDK credential-discovery chain |
| Credential-discovery chain | An SDK-defined selection process that obtains credentials from supported sources, such as environment credentials, an existing login, or the hosting platform |
| Provider kind | An available implementation, such as Azure Key Vault, file Secret Store, or Azure IDP; configuring it does not install native code |
| Secret Store | A provider that resolves an external Secret name to a value |
| Shared IDP configuration (profile) | A named, instance-scoped configuration that supplies workload credentials through `IdentityProvider::get_credentials()` |
| Secret reference | A typed reference to a name in the instance's Secret Store, not a Drasi-owned Secret resource |

[Authentication](01-authentication.md) establishes API caller identity. [Roles and authorization](02-roles-and-authorization.md) defines resource permissions and the permanent Owner's continuing input authority. Neither is replaced by workload credentials.

Native plugin admission remains in the Code & Plugin Plane. Host/cloud provisioning, listener protection, destination policy, and general processing/recovery remain separately owned. The host and admitted native plugins are trusted and unsandboxed; instance scoping is not process isolation.

## Initial authority and administrator trust

### Host-provided authentication choices

Trusted deployment configuration supplies the authentication options available to Drasi. Initial configuration may use seed YAML; initialization commits accepted state, and normal startup must not replay seed definitions over subsequent control-state changes. Host maintenance interfaces are outside this document.

An option may select a particular managed/federated identity or explicitly enable a supported SDK discovery chain. The host may offer explicit identities only, discovery only, or both. Drasi must not enable discovery implicitly.

Instance Secret Store and shared IDP configurations may reference these options, but must not override their identity-selection settings or enable an authentication mechanism the host has not provided. Sources and Reactions select typed Secret references or shared IDP references within their own instance; they must not reference host authentication options directly.

Enabling a chain accepts its documented credential selection and error behavior. Environment changes or a different developer login may change the identity it selects without changing a Drasi profile. This is an accepted host-controlled tradeoff, including for production deployments; a selected chain is not a fixed-identity guarantee.

The host/cloud operator establishes the underlying identity's usability and external permissions. Drasi does not provision cloud identities or grant cloud IAM access. An identity identifier alone is not proof that the deployment can use it.

All Instance Admins may select from the same available host options. There is no per-instance identity allowlist, assignment approval, or uniqueness requirement. Two instances may intentionally use the same identity, external vault, or scope. An administrator may configure a new profile using external authority also used by another instance; this does not grant direct access to that instance's Drasi profiles or components.

Separate profiles or external scopes do not isolate administrators who can select the same host-provided identities. Use separately configured deployments when administrators must not share that external authority.

API Server Admin is not Host Admin. Server Admin may create instances, assign trusted Instance Admins, and administer profiles across instances, but cannot introduce or redefine host authentication options through that API role. Trusted initialization may establish the initial Server Admin Principal/group as specified by administration and authorization.

### Independent bootstrap

The authentication option used to open a Secret Store must work independently of that store. It must not directly or indirectly require a Secret from the store it is opening.

| Mechanism | Source of initial authority |
|---|---|
| Managed or federated workload identity | Hosting platform or independently established federation |
| Explicitly enabled SDK discovery | The configured SDK's credential sources and selection behavior |
| Supported client-secret authentication | Initial credential independently supplied by trusted host provisioning |
| File or OS keyring store | The process's configured OS access to the host-provided backing store |

Host provisioning may use protected files or environment values where required. Remote callers must not submit the initial credential value, choose arbitrary process-environment references, or cause resolved values to be saved in Drasi control state.

Other shared IDP configurations may use Secrets from the instance store once it is available. Do not introduce recursive store chaining or a general credential-dependency framework.

```text
Host-provided authentication option / OS access
                       |
             Instance Secret Store
                       |
              Named Secret values
                       |
       Components and Secret-backed IDP configurations

Host authentication options may also support shared IDPs directly.
```

The runtime obtains authentication credentials, not the creator's human login token. A failed selected mechanism must not cause Drasi to switch to an unrelated option, another instance's provider, or anonymous access. Internal selection within an explicitly chosen SDK chain follows that chain's contract.

## Instance provider configuration

### One optional Secret Store

Each instance has its own resolver context and may configure one Secret Store. An instance without Secret dependencies does not need one. Every Secret reference resolves through that instance's configured store; there is no process-global, cross-instance, or second-store fallback.

Instance Admin selects an available provider kind, its backing store/scope, and the appropriate host authentication option. File/keyring providers use their OS access arrangement rather than an artificial cloud identity.

The host supplies or approves local secret-file locations and keyring scopes. Enabling a local provider is not permission for API users to select arbitrary host files. File stores are not development-only. Operators may choose a plaintext secret file and are responsible for permissions and storage protection; no encrypted-file format or new key-management service is required.

Secrets must already exist in the selected backend. Drasi does not generate external Secret keys, provision Secret values, expose a value-write API, or create per-author Secret resources.

An illustrative component reference is:

```yaml
password:
  kind: Secret
  name: orders-db-password
```

The name is part of the accepted component configuration. It is not a permission grant or a globally unique identifier. The same name in two instances can resolve to different values.

### Multiple shared IDPs

Instance Admin may create multiple named workload IDP configurations. Source and Reaction configuration may select one through a reference such as `identityProvider: orders-login`. Profiles belong to the instance, not their creator's account; there are no author-owned IDPs or per-author credential-sharing grants.

Each IDP defines its provider-specific settings and uses an available host authentication option where external identity acquisition is needed. A password-based IDP may obtain its password through the instance Secret Store. Configuration is immutable; replacement creates a new profile and requires explicit reference updates. Availability may change without changing that configuration.

Retain provider-specific configuration instead of inventing a cross-cloud scope language. Components may supply supported request context, such as a token's target service or a database hostname/port. Context must not replace the selected host authentication option, its identity-selection settings, or a configured assumed role.

For Azure, the configured token scope is a default that supported component context may override, not a Drasi-enforced resource-access boundary. This is token-request context, not permission to change the Secret Store's scope or the administrator-controlled identity settings above. External IAM and destination authorization determine actual permissions. Host/cloud administrators are responsible for appropriate privileges; Instance Admins deliberately expose that usable authority to their Authors.

## Permissions and exposed operations

These operations use the existing authorization model, not new credential-specific roles or individually assignable permissions.

| Operation | Permitted actor and conditions |
|---|---|
| Establish host authentication options or local backing-store access | Trusted host authority; not granted by an API Server Admin or Instance Admin role |
| Browse available host authentication choices and provider kinds for administration | Scoped administrator; expose non-secret selection information, never credential values |
| Read instance provider definitions, status, and affected dependencies | Scoped administrator with `manage-credentials`; Server Admin across instances, Instance Admin within its instance |
| Configure/change/delete/disable/enable the instance Secret Store; create/delete/disable/enable shared IDPs | Scoped administrator with `manage-credentials`, subject to the lifecycle rules below; no native-code admission or external IAM modification |
| Discover available shared IDP identifiers | Instance Author, existing Source/Reaction Owner, or scoped administrator in the relevant instance; not full profile definitions or host authentication configuration |
| Select credential references when creating a Source/Reaction | Instance Author or scoped administrator with ordinary `create` and required input authority |
| Change references on an existing Source/Reaction | Owner or scoped administrator with `update`; target and operational dependents stopped; resulting input `use` checked against the unchanged Owner |
| Start/stop a component using its accepted references | Owner, scoped administrator, or Operator; Start also requires current Owner input authority and credential availability |
| Read a component's accepted Secret names/IDP references | Component Owner or scoped administrator through `read-config`; identifiers are not resolved values |
| Read or write resolved credential values through the API | Nobody, including Server Admin; external provisioning and runtime authentication are separate paths |

Query components do not acquire credentials directly. Bootstrap provider configuration uses its parent Source's instance credential context and configuration authority, not a separate credential ownership system.

All available instance credentials may be exercised by authorized Authors and existing Source/Reaction Owners. Removing Instance Author does not remove an existing Owner's editing or credential-selection authority. Credential availability grants no input `use`, direct Query reads, Reaction `consume`, or access to another component's configuration.

Operators and application consumers may indirectly exercise the credentials of accepted components through their permitted operations. They cannot change credential selections or inspect values. Discovery does not require a vault-wide Secret listing or an API for testing/extracting arbitrary Secret values.

The management interface must support the operations above with exact instance/profile references and current authorization at commit. Reject unknown or cross-instance references, including in Server Admin requests. Validate that each component kind supports its selected authentication mode; a supplied IDP must not be silently ignored or replaced by inline credentials.

## Configuration, runtime, and lifecycle

### Commit references before using credentials

Create/update validates configuration, reference syntax, and supported authentication choices, then commits desired state. It must not resolve Secrets, acquire tokens, construct provider/component runtimes, or connect externally. External credential availability is checked when needed for activation, not through side effects of configuration validation.

Preserve accepted declarative configuration and references as the source for configuration reads, persistence, and export. Never reconstruct them from resolved plugin properties. Export/import follows authorization: references may travel, but provider definitions, host identity configuration, credential values, ownership, and grants do not.

Remote configuration must use typed Secret and IDP references for credentials, including administrative and development requests. Reject plaintext authentication material and process-environment references. Ordinary non-secret parameters, such as hostnames or public client IDs, may remain literals.

Supported plugins must identify and validate credential-bearing inputs, including authentication headers, connection strings, and bootstrap settings where applicable. An opaque string/map field is not an exception to credential handling. Validation errors, runtime errors, properties, and diagnostics must not disclose resolved values; generic redaction after exposure is not sufficient.

### Provider changes

Dependence is determined from saved configuration, not only currently running objects. Include indirect dependencies through Secret-backed IDPs and Source bootstrap providers.

| Operation | Required behavior |
|---|---|
| Configure a new store or IDP | Authorized administrator commits valid declarative settings; no credential acquisition during the mutation |
| Change/replace the instance Secret Store | Require affected components and operational dependents explicitly Stopped; commit the new settings without changing component ownership, grants, or Secret names; leave them Stopped |
| Delete the Secret Store configuration | Reject while any saved component or IDP configuration depends on it, even if stopped or disabled |
| Change an IDP configuration | Reject in-place changes; create a replacement profile and explicitly update references through normally authorized operations |
| Delete an IDP configuration | Reject while saved component/provider references remain, even if stopped or disabled |
| Disable a store or IDP | Allow despite dependencies; mark unavailable, deny subsequent acquisition through existing and new handles, and stop/block affected components without deleting their configuration |
| Re-enable an unchanged provider | Recheck current authority and availability; retained Running intent may resume, while explicit Stopped intent remains Stopped |
| Rotate an external Secret under the same name | Stop/start or Server restart must pick up the current value; an internal reconnect may retain an already-held value |
| Missing/inaccessible required Secret or failed credential acquisition | Fail closed and block affected activation/work; do not select another identity/store or weaker authentication |

An unavailable or Blocked component is not equivalent to explicitly stopping it for a configuration change. Replacement must not silently retarget a live pipeline.

Changing stores can change the meaning of every existing Secret name. The administrator prepares the destination store; Drasi does not guarantee equivalent values or fetch every Secret during the mutation. Missing values fail at activation.

Refresh or reconstruct providers as needed to honor stop/start rotation, including file-store caches and Secret-backed password IDPs. Dynamic `get_credentials()` continues to obtain credentials through the selected profile, subject to its current availability.

Disabling a provider must gate subsequent acquisition even through a previously issued provider handle, and contain components holding cached values. It cannot recall credentials already supplied or guarantee termination of external sessions. External access changes may be detected only on subsequent interactions; there is no continuous external-IAM polling or instantaneous external-revocation guarantee.

### Protected values and audit

Resolved passwords, tokens, authentication certificates, private keys, and Secret values belong only in the runtime paths that need them. Never expose them through API responses, configuration, properties, exports, templates, logs, diagnostics, or audit, or persist them in Drasi control/processing state.

This does not prohibit intended Secret Store backends or protected host-provisioning facilities from holding their required values. Backend storage protection remains the operator's responsibility.

Audit provider mutations, credential-reference changes, denials, and security-relevant availability transitions under the shared audit contract. Record the actual API actor, instance/profile/component identities, operation, relevant accepted-state revision, and outcome. Automatic work uses system/component attribution; host setup uses host attribution, never an invented OIDC caller.

Do not log credentials, full configuration/request bodies, or raw credential-provider responses. Diagnostics expose bounded safe reasons, not secrets. Existing audit-outage and containment rules in [02](02-roles-and-authorization.md#audit-and-security-dependencies) apply; this document does not add a new outage exemption for profile mutations.

## Threats, risks, and decisions

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Global resolver selects the wrong instance's store | Two instances use `db-password`; one receives the other's value | Cross-instance credential disclosure/use | Bind every resolution path to the component's instance; no global fallback or temporary global-provider switching |
| Administrator recreates another instance's external access | An Instance Admin selects the same host identity and vault used elsewhere | Assumed isolation of external authority | Accepted: trusted admins share the host-provided options; external IAM applies, no per-instance identity allowlist |
| Author exercises powerful shared credentials | An Author creates a new Source using a broadly privileged credential | External access beyond what another component exposes | Accepted trusted-author model; administrators control available profiles and external permissions, no per-author credential grants |
| Discovery selects an unintended identity | New environment credentials take precedence over the VM identity | Unplanned privileges or changed external attribution | Host must explicitly enable the SDK chain; its environment-dependent identity selection is an accepted risk, not a fixed-identity guarantee |
| Bootstrap depends on itself | Vault login requires a Secret inside that vault | Unusable credential acquisition or insecure workaround | Independently provision initial authority; reject self-dependent authentication and do not add fallback credentials |
| Plaintext or ENV enters through another configuration field | An API request embeds a token in an HTTP header or references a host variable | Persisted/exposed credentials or unintended host authority | Typed credential inputs and reference validation, including nested/plugin-specific settings; host bootstrap exception is not an API bypass |
| Runtime inspection reveals resolved values | A plugin serializes its password into `properties()` | Disclosure through configuration, export, or diagnostics | Committed unresolved configuration is authoritative; credential-safe runtime properties/errors and no value-return API |
| Token scope is mistaken for an access restriction | A component requests a different Azure service token | External authority broader than a profile's default suggests | Accepted: scope is not generic Drasi authorization; retain configured identity/mechanism, rely on external permissions |
| Store replacement changes a running workload's authority | A profile switches from a limited file store to a broader vault | Uncontrolled live credential changes | Require explicitly stopped affected workloads and deliberate admin mutation; no guarantee of equivalent external values |
| Deleted provider leaves live or saved consumers | An IDP is removed while Sources still reference it | Broken references or stale credential use | Reject referenced deletion; explicit replacement and separate disablement |
| Cached values survive rotation or disablement | A file cache returns the old password after restart of a component | Unexpected old credentials or continued acquisition | Fresh values on stop/start; availability checks on existing handles; contain dependents, but do not promise recall of issued credentials |
| File backend exposes secrets through host access | Another OS account reads the configured JSON file | Disclosure of stored values | Operator-protected file/keyring access; plaintext file storage is an accepted host choice, not a promise of encrypted-at-rest protection |
| Local provider becomes arbitrary host-file access | Remote configuration substitutes a different filesystem path | Host data or credentials outside intended backing stores | Host-provided/approved local locations; provider selection does not confer arbitrary OS access |
| Credential failure triggers a different identity | Vault access fails and a plugin tries an unrelated developer login | Unauthorized authority substitution | Fail closed; only the explicitly chosen mechanism's documented discovery behavior is permitted |
| Host or admitted native plugin is compromised | Native code reads another provider's process memory | Bypass of logical scoping and credential confidentiality | Accepted trusted-process boundary; no sandbox or adversarial multi-tenancy guarantee |

## Implementation boundaries and verification

Reuse DrasiLib's Secret Store and workload IDP abstractions. Replace process-global resolution with explicit instance-bound context across Server construction, built-in providers, statically linked plugins, SDK/FFI factories, and Source bootstrap providers. Seeded and API-created components must use the same contract.

The current Server startup/factory paths, SDK config resolver, runtime credential contexts, and configuration persistence are integration points. Code admission, caller OIDC, role evaluation, and general processing-state recovery are not reimplemented here. Embedded DrasiLib applications need not adopt Server's remote configuration or role model; Server-managed execution must not use that distinction to bypass this contract.

| Verification case | Required result |
|---|---|
| Two instances resolve the same Secret name concurrently | Each uses its own configured store across static, dynamic, built-in, and bootstrap paths |
| No configured store, unknown profile, or cross-instance reference | No default/global provider fallback; reject invalid references and fail required acquisition |
| Host options versus admin/Author/Owner/Operator operations | Only the defined actor may configure or select the relevant authority; no credential-value API |
| Explicit identity versus host-enabled discovery | Selected mechanism is used; unconfigured discovery cannot activate; chain selection does not imply a fixed identity |
| Configuration validation and persistence | No external acquisition during create/update; accepted references survive restart/export without resolved values or seed replay |
| Store change, referenced deletion, IDP replacement, disable/re-enable | Enforce saved dependencies and stopped preconditions; preserve ownership/grants/intent; no silent reference rewriting or stale-handle bypass |
| Rotation with file-store and password-IDP caching | Stop/start obtains current Secret values; reconnect behavior does not falsely promise refresh |
| Provider context and authentication mode | Supported service-specific context works without substituting identity/mechanism/role; unsupported IDP selections are not ignored |
| Nested credentials, provider errors, properties, logs, and audit | Reject disallowed inputs and expose no credential values, including on failure paths |

Host-option representation, cloud SDK integration, Rust types, FFI layouts, caching, synchronization, and dependency indexing remain implementation choices. They must implement the operation and permission contracts above, not introduce new roles, identity allowlists, credential-binding resources, value-management APIs, or a credential broker.

### API Design

One implementation task must define and document a single shared API contract for provider management and identifier discovery, including HTTP methods, versioned paths, request/response schemas, and error responses. Server, UI, and other clients must use that contract rather than independently inventing interfaces.

These wire details will be settled during implementation, not in this informal specification. The shared contract must preserve the operations, permissions, lifecycle preconditions, and failure behavior defined here; it does not introduce additional policy decisions.

## Alternatives considered

Per-author credential ownership and bindings conflict with the selected trusted-instance model. Per-instance identity allowlists add an administration boundary we intentionally do not require. A mandatory cloud vault excludes legitimate file/keyring deployments. Mandatory fixed identities sacrifice host-selected SDK discovery convenience; unconditional discovery hides the delegation decision. Generic cross-cloud scope enforcement would duplicate provider-specific authorization.

Multiple Secret Stores per instance, arbitrary provider-dependency graphs, automatic identity provisioning, and live credential retargeting are not required. One optional store, shared immutable IDPs, explicit references, and controlled lifecycle operations suffice for V1.

## References

- [Drasi Server authentication](01-authentication.md)
- [Drasi Server roles and authorization](02-roles-and-authorization.md)
- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
