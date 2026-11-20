# Drasi Server Control-Plane Security Overview

* Project Drasi - 2026-10-19 - Aman Singh (@amansinghoriginal)

**Status:** Draft

## Overview

<!-- Provide a succinct high-level description of the component or feature and where/how it fits in the big picture. The overview should be one to three paragraphs long and should be understandable by someone outside Drasi team. -->

Drasi Server hosts continuous data processing through Sources, Queries, and Reactions. Its control plane governs who may configure, operate, and access those resources, which credentials they may use, and how accepted authority persists independently of the runtime.

This document is the starting point for the control-plane behavioral specifications in this directory. It explains how their contracts fit together and records the constraints that apply across the design. They describe target behavior, not the current implementation. Each focused specification owns its detailed rules; this overview is a map, not a competing policy or a replacement for reading those rules.

## Terms and definitions

<!-- These terms are internal to your design and will not be shared publicly. -->

| Term | Definition |
|------|------------|
| Principal | The caller identity established by authentication; distinct from the permissions it receives |
| Owner | The permanent Principal associated with a Source, Query, or Reaction, not an assignable role |
| Host administrator | A person or automation trusted through deployment/OS authority, not a Drasi API role |
| Control state | Committed configuration, resource identities, ownership, bindings, references, and execution intent |
| Workload credentials | Credentials used by a component to access an external system, not its creator's API login token |

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->

### User scenarios

<!-- Describe the user scenarios for this design. Ensure that you define the roles and personas in these user scenarios when it requires API design. -->

A maintainer reviews the system's authority boundaries before reviewing an individual specification. A developer or coding agent uses the specification map to identify all contracts affected by a change, rather than implementing authentication, credentials, persistence, or audit in isolation.

### Goals

<!-- Describes goals to define why we need, where we go, and how we know whether we will be successful when we deliver this feature. -->

Provide a shared system model, clear specification ownership, and a consistent basis for implementation and cross-component verification.

### Non-Goals

<!-- Describe non-goals to identity something that we won't be focusing on immediately. We won't be expending any effort on these matters. -->

This overview introduces no new permission, API, service, or guarantee. It does not duplicate permission matrices, provider lifecycle tables, audit ordering rules, or implementation plans from the focused specifications.

## Design requirements

<!-- Describe key technical requirements, dependencies, and constraints that impact the design.

Provide a link to the issue(s) tracking this work. -->

### Requirements

<!-- List design goals or constraints that impact the design. Examples include extensibility points, maintainability, code reuse, considering if your feature needs to be taken down-level and coherency with related code. -->

These constraints apply to the entire control-plane design:

- **Backward compatibility, user migration, and data migration are not requirements.** Existing APIs, configuration shapes, and implementation paths are not behavior that must be preserved.
- **Choose the simpler approach for V1.** Additional complexity needs a concrete requirement justified by demand, not a hypothetical future scenario. Simplicity does not permit bypassing an agreed security boundary.
- **Specify behavior before machinery.** Define authority, observable outcomes, failure behavior, and accepted limits. Libraries, storage layouts, SDK/FFI signatures, and wire details remain implementation choices where the owning specification leaves them open.
- **Keep one owner for each contract.** Reuse the focused specifications and shared security authority; do not add alternative policy stores or reinterpret a deferred interface as permission to introduce new policy.

### Dependencies

<!-- Describe the key dependencies that this design has on other features being developed as well as the dependencies other features have on this design. For example, supporting AWS resource has multiple feature dependencies with Bicep types, UCP, CoreRP, LinkRP, and CLI. This section summarizes such dependencies with the design document links. -->

| Specification | Owns |
|---|---|
| [01 - Authentication](01-authentication.md) | API caller identity, production login, browser credentials, private-development authentication, and token/stream lifetime |
| [02 - Roles and authorization](02-roles-and-authorization.md) | Fixed permissions and roles, permanent ownership, sharing, visibility, current input authority, and revocation |
| [03 - Instance credentials](03-instance-credentials.md) | Host authentication options, instance Secret Stores/shared IDPs, typed references, provider lifecycle, and credential-value protection |
| [04 - Security state and administration](04-security-state-and-administration.md) | Authoritative state, configuration sources, initialization, consistent mutations, runtime intent, and host access recovery |
| [05 - Templates and portability](05-templates-and-portability.md) | Host-published recipes, safe parameters, configuration export/import, and sequential Apply outcomes |
| [06 - Audit](06-audit.md) | Required evidence, actor attribution, local-write acknowledgement, operation ordering, and audit availability |

The numbers provide navigation, not a requirement to implement six independent subsystems in sequence. Changes often span several contracts. In particular, authentication establishes identity, authorization depends on current control state, and audit participates in admission and completion.

### Out of scope

<!-- Describe the important issues or concerns NOT addressed by this proposal. 1 paragraph each (can be short). Often the "out of scope" items refer to the current state. We might already have planned work to address the "out of scope" items, mention it if that's the case. -->

Native artifact admission, preparation, loading, and activation belong to the separate **Code & Plugin Plane**. Component-management permissions do not confer native-code admission authority.

Listener transport, provider-webhook proofs, protocol integration, outbound delivery safeguards, and host input/output policy belong to separate **ingress and egress designs**. Those integrations must honor the control-plane identity and authorization contracts; this overview does not approve or implement their separate policies. In particular, 02 retains submission-only Source `ingest`; broader Source application access requires an explicit amendment.

Processing correctness, checkpointing, replay, delivery guarantees, general state recovery, and backup/restore are outside this security design. Protecting authoritative control state from unauthorized changes or permissive fallback remains in scope under 04.

## Design

### High-level design

<!-- Provide a high-level description, using diagrams as appropriate, and top-level explanations to convey the architectural/design overview. Don't go into a lot of details yet but provide enough information so that the relationship between these components and other components. Call out or highlight new components that are not part of this feature (dependencies). This diagram generally treats the components as black boxes. Provide a pointer to a more detailed design document, if one exists. If you have the alternatives for your design, describe alternative solution too. -->

The host establishes deployment trust and initial authority. API callers authenticate and operate through scoped permissions. Accepted mutations are committed before runtime effects; DrasiLib is the runtime projection of that state, not the source from which configuration or permissions are reconstructed.

Sources and Reactions obtain workload credentials through their instance context. That context neither identifies an API caller nor grants permission to use another component's data. Server-managed runtime and SDK paths use the host's current security authority; ordinary embedded DrasiLib applications are not required to adopt Server login, roles, or its control store.

### Architecture Diagram

<!-- Provide an architecture design if applicable. An architecture diagram should be provided when the design involves multiple parts of Drasi (Communication between RPs, Communication between CLI and RPs, Protocol changes for UCP, Communication with external systems like clouds, etc.) -->

```text
API caller -> Caller authentication -> Current authorization
                                              |
                         +--------------------+--------------------+
                         |                                         |
                   Protected reads                       Validated mutations
                         |                                         |
                   Authorized views                       Commit desired state
                                                                   |
                                                       Reconcile permitted runtime
                                                                   |
                                                    Sources -> Queries -> Reactions

Sources/Reactions -> Instance credential context -> Secret Store / external IAM
API/runtime/host operation boundaries -> Audit -> Host-protected local file
```

This shows responsibilities, not a protocol topology or universal operation order. Current authorization comes from committed ownership and bindings. Audit participates at the operation boundaries defined in 06: its ordering differs for mutation outcomes and sensitive data release.

### Detail design

<!-- This section should be detailed and through enough that another developer could implement your design and provide enough detail to get a high confidence estimate of the cost to implement the feature but isn't as detailed as the code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and describe it in enough detail that someone else could implement it. Cover ALL of the important decisions like names. Your goal is to get agreement to proceed with coding and PRs.

If there are alternative you are considering please include that in the open questions section.

If the product has a layered architecture, it's good to align these sections with the product's layers. This will help readers use their current understanding to understand your ideas.

- **Advantages of this design** - Describe what's good about this plan relative to other options. Does it feel easy to implement? Provide flexibility for future work?
- **Disadvantages** - Describe what's not ideal about this plan. If you don't point these things out other people will do it for you. This is a good place to cover risks. -->

The following summarize cross-cutting invariants; the owning specifications define their exact permissions, exceptions, and outcomes.

| Invariant | Relationship across the design |
|---|---|
| Authentication is not authorization | A valid caller identity supplies no resource access by itself. Workload IDPs are a separate credential-acquisition mechanism. |
| Actor and Owner are distinct | Administrative edits preserve the permanent Owner and check that Owner's input authority. Audit records the actual editor, not an impersonated Owner. |
| Shared credentials are not component grants | Trusted Authors and existing Source/Reaction Owners may exercise available instance credentials; access to another component still requires the applicable permission. |
| Committed state governs runtime | Create/update commits references without constructing runtimes or acquiring credentials. Seed state is not replayed, and runtime inspection cannot recreate authority. |
| Continuing access needs current authority | Prior admission, cached results, Running status, and established streams are not permanent grants. Revocation and credential lifetime continue to apply; pipelines do not retain human login tokens. |
| Portability does not carry privilege | Templates and imports use ordinary destination-side authority. References may travel, but ownership, bindings, credential values, and host authority do not. |
| Completion must be honest | A later runtime or audit failure does not undo a committed change. Audit acknowledgement, outage restrictions, and permitted containment follow 06, not a blanket success or shutdown rule. |
| Development is explicit, not a fallback | Private development substitutes a stable caller identity through the normal policy. It does not waive credential protection, control-state persistence, or auditing. |

### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable.

- Describe the REST APIs in detail for new resource types or updates to existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components. -->

N/A: this overview adds no interface. Follow the operation contracts in 01-06. Where wire details are deferred, define one shared contract for the Server, UI, CLI, and SDK integrations that need it, rather than independent interpretations.

### Alternatives Considered

<!-- Describe the alternative designs that were considered or should be considered. Give a justification for why alternative approaches should be rejected if possible. -->

A monolithic specification duplicates the focused contracts and makes policy drift harder to detect. Disconnected documents leave shared assumptions implicit. A short overview plus linked, individually owned specifications provides orientation without another source of competing rules.

## Security

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc. -->

The host, trusted identity infrastructure, and admitted native plugins are trusted; native plugins are not sandboxed. Logical instances scope authorization and credential resolution, not adversarial process isolation. Instance Admins share the host-provided authentication choices under 03, so distinct profiles do not guarantee distinct external authority.

The individual specifications document accepted limits, including trusted-author credential use, downstream redistribution, permanent ownership, host-enabled identity discovery, and audit history that is not crash-complete. This overview neither strengthens those promises nor relaxes their safeguards. Main-API security alone does not secure every listener or external effect; the separate planes must implement the relevant boundary contracts.

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

Backward compatibility, user migration, and data migration are not requirements. Server, client, and host-tool changes must still agree on the new contracts they implement.

## Supportability

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->

Use 06 for security evidence and audit health, 02 for data-free diagnostic access, and 04 for accepted-state versus runtime outcomes. Do not treat application output, operational logs, audit, and control state as interchangeable.

### Verification

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

Use the verification cases in each owning specification as behavioral acceptance criteria. Cross-document verification must cover authenticated operation admission, Owner-preserving edits, credential references through persistence and activation, revocation of active/cached access, and commit/audit failures with the permitted inspection and containment paths. Verify the shared contract across Server-managed entry points, not only isolated helpers.

## Development Plan

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

This change adds only the overview. Implementation work items should link the relevant sections of 01-06, identify shared dependencies, and include their verification cases. Define shared interfaces once and reuse them; document narrower implementation choices without introducing additional policy or importing assumptions from superseded drafts.

## Open issues

<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design. -->

**Does this overview introduce another policy round?** No. The focused specifications identify their remaining implementation choices. A concrete contradiction or unenforceable boundary should be resolved in the owning specification, not silently settled by an implementation or by expanding this overview.

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

None.

## References

<!-- Optional. Add the design documents and references you use for this document. -->

- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
- [Specification map](#dependencies)
