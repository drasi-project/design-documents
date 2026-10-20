# Drasi Server Audit

* Project Drasi - 2026-10-19 - Aman Singh (@amansinghoriginal)

**Status:** Draft

## Overview

<!-- Provide a succinct high-level description of the component or feature and where/how it fits in the big picture. The overview should be one to three paragraphs long and should be understandable by someone outside Drasi team. -->

V1 starts with a simple local audit file and explicit operation outcomes. We will expand this approach when justified by user demand, rather than introduce additional audit infrastructure in anticipation of it. Backward compatibility, user migration, and data migration are not requirements.

Drasi Server will record who attempted security-sensitive operations, which resources and accepted state were involved, and what happened. Audit is separate from operational logging, application output, and the authoritative control store. It must not contain credentials or protected data.

This design defines target behavior, not the current implementation. It completes the shared audit contract referenced by the other control-plane designs without adding roles, an audit-query API, or a second authority store.

## Terms and definitions

<!-- These terms are internal to your design and will not be shared publicly. -->

| Term | Definition |
|---|---|
| Audit record | A bounded structured description of an operation, decision, outcome, or security-relevant transition |
| Audit-required operation | An operation whose record must be written under the coverage and ordering rules below |
| Write acknowledgement | Confirmation that a complete record was written to the local file, not merely queued or accepted by a logger |
| Known audit outage | The audit facility cannot currently provide that acknowledgement within its supported bounds |
| Correlation | Host-established identifiers linking an operation, its constituent changes, and later outcomes; not authentication or authority |

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->

### User scenarios

<!-- Describe the user scenarios for this design. Ensure that you define the roles and personas in these user scenarios when it requires API design. -->

- A component Owner or scoped administrator changes accepted configuration and can distinguish a rejected change from one committed with incomplete runtime or audit completion.
- An authorized reader requests results or a configuration export; Drasi records admission before releasing protected data.
- A host administrator configures local audit storage and performs initialization or access recovery with host-attributed evidence.
- During an audit-only outage, authorized callers can still Stop components or revoke permission, while existing authorized pipelines remain subject to normal security checks.

### Goals

<!-- Describes goals to define why we need, where we go, and how we know whether we will be successful when we deliver this feature. -->

- Record bounded, correctly attributed security evidence independently of ordinary logging.
- Define honest local-write acknowledgement, operation outcomes, and audit-outage behavior.
- Preserve credential and data confidentiality without introducing another authority store.

### Non-Goals

<!-- Describe non-goals to identity something that we won't be focusing on immediately. We won't be expending any effort on these matters. -->

No new audit-reader role, audit database, mandatory per-record durable flush, persistent outbox, remote delivery guarantee, cryptographic history chain, or processing-recovery mechanism is required.

## Design requirements

<!-- Describe key technical requirements, dependencies, and constraints that impact the design.

Provide a link to the issue(s) tracking this work. -->

Related design: [Control-plane security overview](00-overview.md).

### Requirements

<!-- List design goals or constraints that impact the design. Examples include extensibility points, maintainability, code reuse, considering if your feature needs to be taken down-level and coherency with related code. -->

The contract applies to Server-managed API, runtime, and host-administration operations. SDK integrations must carry the context needed by those operations; ordinary embedded DrasiLib applications are not required to adopt Server auditing.

### Dependencies

<!-- Describe the key dependencies that this design has on other features being developed as well as the dependencies other features have on this design. For example, supporting AWS resource has multiple feature dependencies with Bicep types, UCP, CoreRP, LinkRP, and CLI. This section summarizes such dependencies with the design document links. -->

| Concern | Owning contract |
|---|---|
| Verified caller identity, development identity, and credential lifetime | [Authentication](01-authentication.md) |
| Operation permissions, visibility, revocation, and continuing runtime authority | [Roles and authorization](02-roles-and-authorization.md) |
| Credential references, provider lifecycle, and value protection | [Instance credentials](03-instance-credentials.md) |
| Host settings, initialization/recovery, committed state, and runtime outcomes | [Security state and administration](04-security-state-and-administration.md) |
| Export, template deployment, and constituent Apply mutations | [Templates and portability](05-templates-and-portability.md) |
| Audit coverage, local recording, acknowledgement, and audit availability | This design |

### Out of scope

<!-- Describe the important issues or concerns NOT addressed by this proposal. 1 paragraph each (can be short). Often the "out of scope" items refer to the current state. We might already have planned work to address the "out of scope" items, mention it if that's the case. -->

Other security planes define their own operations and integrate with this audit contract where required. This document does not decide native plugin admission, listener authentication methods, delivery semantics, or processing recovery.

## Design

### High-level design

<!-- Provide a high-level description, using diagrams as appropriate, and top-level explanations to convey the architectural/design overview. Don't go into a lot of details yet but provide enough information so that the relationship between these components and other components. Call out or highlight new components that are not part of this feature (dependencies). This diagram generally treats the components as black boxes. Provide a pointer to a more detailed design document, if one exists. If you have the alternatives for your design, describe alternative solution too. -->

Drasi Server owns one audit facility per deployment. Its operation-facing interface must report successful recording or failure. It must be independent of `RUST_LOG`, component log levels, optional processing storage, and whether anyone subscribes to diagnostics.

Produce records where the security decision and operation outcome are known. Do not reconstruct them from HTTP status codes, arbitrary log messages, or lifecycle broadcasts. A failed HTTP request may already have committed a change; a runtime status change does not identify the person who requested it.

Host initialization and offline access recovery use the same record and file contract under 04's exclusive deployment access. This is shared code, not a requirement to run a separate audit service. The audit writer does not acquire authority to perform the operation it describes.

### Architecture Diagram

<!-- Provide an architecture design if applicable. An architecture diagram should be provided when the design involves multiple parts of Drasi (Communication between RPs, Communication between CLI and RPs, Protocol changes for UCP, Communication with external systems like clouds, etc.) -->

```text
Authenticated API operations     Host maintenance     Managed runtime/security hooks
              |                         |                         |
              +-------------------------+-------------------------+
                                        |
                         Server-owned audit facility
                       bounded metadata + acknowledged write
                              |                     |
                        Local audit file     Availability state
                              |                     |
                    Optional host collector  Admission and readiness
```

### Detail design

<!-- This section should be detailed and through enough that another developer could implement your design and provide enough detail to get a high confidence estimate of the cost to implement the feature but isn't as detailed as the code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and describe it in enough detail that someone else could implement it. Cover ALL of the important decisions like names. Your goal is to get agreement to proceed with coding and PRs.

If there are alternative you are considering please include that in the open questions section.

If the product has a layered architecture, it's good to align these sections with the product's layers. This will help readers use their current understanding to understand your ideas.

- **Advantages of this design** - Describe what's good about this plan relative to other options. Does it feel easy to implement? Provide flexibility for future work?
- **Disadvantages** - Describe what's not ideal about this plan. If you don't point these things out other people will do it for you. This is a good place to cover risks. -->

#### Host configuration and local file

The host may configure an audit-file path through the startup-setting contract in 04. The setting is optional; auditing is not. It applies to Server-managed production and private development deployments.

| Configuration | Behavior |
|---|---|
| Path omitted | Use `<deployment-directory>/audit/audit.jsonl` |
| Relative path | Resolve against the deployment directory, not the process working directory |
| Explicit path | Use the host-selected destination; API/component settings cannot redirect it |
| File or required directory absent | Create with restrictive OS access appropriate to the platform |
| Destination cannot be created or written | Apply the startup/outage rules; no fallback to stdout, another path, or disabled audit |

Illustrative host startup setting, not a prescribed overall YAML layout:

```yaml
audit:
  path: /var/lib/drasi/audit/audit.jsonl
```

This is not seed-once resource authority. Changing it through host startup configuration takes effect on restart; it does not require deployment reinitialization or change resources and bindings. A deployment may keep startup settings and seed input in one file or separate files under 04.

Write structured, newline-delimited JSON records with safe encoding and bounded fields. Append without overwriting existing history or interleaving concurrent records. The audit file is not an approved application-output destination or the control-state database. Do not introduce an API to change the path, disable audit, clear history, or read audit records.

The host protects the directory and files and manages capacity, rotation, retention, and optional collection. Rotation/reopening must preserve the append and acknowledgement contract; an unavailable writer follows the same outage rules. Existing history is not silently deleted when the configured path changes.

##### What acknowledgement guarantees

A complete record must reach the local file through a successful write before it is acknowledged. Acceptance into a memory queue, a tracing call, or a userspace buffer alone is insufficient. Internal asynchronous writing is allowed if the operation receives the actual write outcome.

V1 does not require a durable flush for each record. A host/power failure may lose an acknowledged tail; interrupted writes may leave an incomplete tail. A crash can also occur between a control-state commit and its audit record. This is not an atomic, crash-complete, or exactly-once audit history.

Write failures, partial writes, and exceeded writer bounds must not be acknowledged as successful audit. Bound pending records, bytes, and write-wait time; required records cannot be silently dropped to keep the facility looking healthy.

Before appending another record after a partial or interrupted write, the serialized writer must establish a line boundary. If the existing tail is not newline-terminated, append a delimiter without overwriting or truncating the damaged bytes. That delimiter does not complete or acknowledge the damaged record. Subsequent records, including startup and recovery records, must be independently readable, newline-terminated JSON records. If the boundary or the new record cannot be written, do not acknowledge recovery or reopen admission.

The local file remains useful after component deletion. Runtime cleanup must not remove its audit history. It is not tamper-proof against an administrator or native code that controls the host.

#### Audit coverage

Record semantic operations rather than every internal function call. The following table defines required coverage; it does not grant permission to perform any operation.

| Event category | Required record |
|---|---|
| Authentication failure or unavailability | Missing, invalid, expired, or unverifiable credentials and unavailable verification, with safe reason categories; repeated unauthenticated failures may be aggregated as described below |
| Authorization denial | Denied action and known target/scope, including hidden-target denials; normal caller-facing disclosure restrictions still apply |
| Resource mutation | Instance/component create, update, and delete, including rejected or failed attempts and actual commit/completion outcome |
| Binding mutation | Assignment/revocation, exact role and grantee selector, target, decision, and outcome |
| Credential configuration | Store/IDP creation, change/replacement, deletion, disable/re-enable, and component credential-reference changes; record identities/revisions, not values or configuration bodies |
| Lifecycle and containment | Requested Start/Stop, reset-state if exposed, and their outcomes; security-driven blocking/resumption and relevant provider-availability transitions |
| Sensitive read | Query/application result access, snapshots, and configuration export; record the access scope, not returned data |
| Protected sessions and streams | Opening/admission and closing, including expiration, revocation, disconnect, or failure when observed |
| Application mutation | Supported security-sensitive application-data changes, such as Dashboard edits, with target and outcome but no application body |
| Composite operations | Constituent mutations and partial outcomes for Apply/import/template deployment, correlated to the parent request |
| Host administration | Initialization and additive access recovery, including failure and committed-but-incomplete outcomes, attributed to the observed host actor |
| Audit facility | Startup/readiness establishment, detected recording failure through safe operational reporting, and successful recording of recovery |

Routine successful metadata/catalogue reads, health checks, and read-only management inspection do not require a record per read. This includes permission-gated accepted-configuration/binding inspection and data-free diagnostics needed to understand an operation's outcome. It does not exempt export or permit results to be exposed as diagnostics.

Ordinary successful event ingestion, individual result rows, reaction deliveries, heartbeats, metrics, and routine processing/retry events do not require durable audit records. Use bounded operational telemetry instead. An application-data mutation is not automatically equivalent to ordinary event ingestion.

For automatic work, record security-relevant transitions rather than repeatedly auditing the same blocked condition on every retry. Audit does not require observing every credential fetch, external identity selection, or processing step.

An internal helper created to serve a Query read remains part of that read's scope and lifetime under 02. Do not turn it into an apparent user-created/owned Reaction in audit.

#### Record content and attribution

Use a defined structured envelope, not arbitrary dictionaries copied from requests or plugins. Exact wire names are implementation choices; these meanings must be available:

| Information | Required meaning |
|---|---|
| Record identity and time | Host-assigned event identity, schema version, and timestamp; payload timestamps are not security time |
| Correlation | Operation/session identity and parent operation where applicable; do not rely on untrusted client identifiers for uniqueness |
| Actor | Actor kind and trusted identity/context as defined below |
| Action and target | Semantic action, deployment, and relevant instance/component/provider UIDs and types when known |
| Decision | Allowed, denied, or unavailable security decision, with a bounded safe reason |
| Accepted state | Relevant accepted-state revisions where applicable; mutation commit status is not applicable to operations that do not mutate control state |
| Completion | Actual operation/runtime outcome and failed phase; distinguish not attempted, pending, completed, and failed where applicable |
| Binding-specific context | Fixed role and exact Principal/group selector for a binding operation |

Unavailable information must remain explicitly absent. An unauthenticated request or failed initialization may not have a resolved component or deployment UID; do not invent one. Diagnostic readable names may supplement, but never replace, authoritative identities.

| Actor kind | Attribution |
|---|---|
| Authenticated API caller | Validated issuer and subject from 01, including a scoped administrator editing another Owner's component |
| Private development caller | Explicit fixed development identity, distinguishable from a production OIDC login |
| Unauthenticated caller | Unauthenticated/unknown, never an asserted subject decoded from an unverified token |
| Verified provider integration | Bound integration/component context supplied by the owning ingress contract, not an invented OIDC Principal or the Source Owner |
| Host maintenance | Observed OS identity under protected deployment access, not identity supplied through a user argument or environment label |
| Automatic work | System/component attribution and relevant accepted state; the reconciler does not impersonate the permanent Owner |

A binding's grantee or group selector is not necessarily the actor. Accepted profile/host-option references do not prove which external identity an SDK discovery chain actually selected.

Never record tokens, authorization codes, PKCE verifiers, passwords, resolved Secrets, authentication certificates/private keys, signatures, credential-bearing headers/URLs, full request/configuration bodies, raw parameter maps, query text, or protected payloads/results. Use safe reason categories rather than raw exception strings or provider responses. Encode and bound permitted identifiers so caller-controlled strings cannot inject extra records.

Audit is host-protected evidence, not another component discovery feed. Its records must not be exposed through component logs/events or generic diagnostic APIs. A Log Reaction's intentional result output is likewise not audit.

#### Operation ordering and completion

##### Mutations and composite work

Use the authorization and commit boundary from 02/04. Check audit availability before admitting new audit-required work. A healthy check is not a reservation that guarantees a later write.

Commit the authorized change, apply permitted runtime effects, and record the actual outcomes before reporting the operation fully complete. An asynchronous runtime effect may have a correlated later outcome; do not claim activation succeeded just because intent committed. No control-store transaction may remain open across audit I/O.

If the audit write fails after commitment, preserve the committed state and report committed-but-incomplete with the actual runtime outcome. Do not undo a grant/revocation, claim rollback, or imply that blindly repeating the request is safe. A client disconnect must not itself cancel the responsibility to record a committed operation while the Server continues running; this is not a guarantee across process crashes.

For Apply, follow 05's per-entry commitment and stop-on-failure contract. Each constituent operation needs its own target/outcome evidence; a top-level success/failure record alone is insufficient. Use correlation to connect the entries, without recording the bundle or template parameter values.

##### Reads, denials, and sessions

For an authorized sensitive read/export or protected session opening, write its admission record before releasing protected data. If that write fails, do not release the response, cached results, or initial stream data. Recheck current authority as required by 02; a written audit record does not reserve access after revocation.

An access record means Drasi admitted the access, not that every byte reached the client. Record observed failures or closure without asserting delivery or recall guarantees.

Authentication/authorization failures remain rejected even when their audit attempt fails. Never permit the request because the denial cannot be recorded, and do not recursively generate audit events about every failed audit write.

Close protected access on expiration, revocation, disconnect, or lost eligibility regardless of writer availability. Attempt to record the observed close; failure must not hold the session open. Abrupt termination may leave an opening record without a corresponding close.

Automatic enforcement of lost authority must likewise proceed even if its audit record cannot be written. Recording failure cannot delay security-driven blocking or turn withdrawn permission back into usable authority.

#### Audit availability and recovery

This is an audit-only outage policy: it never relaxes authentication, authorization, credential, or control-store requirements.

| Situation | Required behavior |
|---|---|
| Normal startup or initialization | Require a usable destination and a successful startup/readiness record before normal protected operation or initialization commitment; no disabled-audit fallback |
| New audit-required operation during a known outage | Reject admission before its mutation, sensitive data release, or new protected session |
| Previously admitted pipelines and streams | May continue under all normal authority/lifetime checks; an existing connection does not exempt new audit-required requests within it |
| Authorized Stop or permission revocation | Remain available; attempt audit and explicitly report recording failure and actual saved/runtime outcomes |
| Provider mutation, new grant, Start, export, or access recovery | No additional outage exception, including actions described as maintenance or containment |
| Authorized safe management inspection | Remains available within 02's visibility rules; no results or export disguised as inspection |
| Recording fails after a committed mutation | Report committed-but-incomplete, mark audit unavailable, and restrict subsequent admissions |
| Recording becomes usable again | Write a successful health/recovery record, then automatically reopen audit admission if other security dependencies also permit it |

The Stop/revocation exception still requires valid authority and a successful control-state commit under 04. It does not permit unsaved changes, bundle in new grants, or bypass unrelated operation preconditions. Offline additive Server Admin recovery grants authority and therefore requires normal audit admission.

Audit failure degrades readiness, not process liveness. Expose aggregate public readiness only; authorized diagnostics may show safe audit-health reasons without raw errors, credentials, or host-path disclosure. Keep permitted inspection/containment reachable rather than terminating the process merely because a writer fails. Whether startup waits or exits before becoming ready is an implementation choice.

Use bounded failure reporting and recovery attempts. A pathname existing or a successful open alone is not proof of recovery; a complete health/recovery record must be written. Record the known gap and available failure counts without claiming to reconstruct missing operation records.

No persistent audit replay queue, retrospective reconstruction from control state, or additional acknowledgement from a remote collector is required. Graceful shutdown should finish bounded pending writes; forced termination is subject to the stated incomplete-history limit.

#### Bounded failure recording

Unauthenticated callers can generate unlimited missing/invalid-token requests. Retain bounded individual detail and explicit aggregate counts for repeated unauthenticated failures. Aggregates must identify the safe failure category and covered time interval; aggregation must not silently claim each attempt has an individual record.

Bound counters, grouping keys, records, bytes, and writer work. Do not create unlimited buckets from attacker-supplied subjects, tokens, URLs, or addresses. The requests remain denied regardless of aggregation.

Do not sample accepted sensitive operations, binding mutations, authenticated administrative outcomes, or authenticated authorization denials. Their required-record failures invoke the outage policy. An authenticated flood can still reduce availability within these bounds; this is not a per-user quota or adversarial resource-isolation guarantee.

Counters and pending summaries need not survive a crash. If the sink is unavailable, bounded health/operational reporting must reveal the loss rather than silently present a complete history. Numeric limits, aggregation windows, backpressure, and scheduling are implementation choices.

#### External collection and future destinations

An operator-managed collector may forward the local file to a remote service. Drasi acknowledges the local write only; successful local audit does not prove collection, remote delivery, indexing, or retention. A collector outage alone is not an audit-writer outage.

Keep record generation separate from writing so a later direct remote destination can be considered without redefining every audited operation. V1 does not add remote sink configuration, a public sink-plugin framework, credential provisioning for a collector, or a delivery subsystem. Direct remote acknowledgement, buffering, and outage behavior would require an explicit later decision.

### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable.

- Describe the REST APIs in detail for new resource types or updates to existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components. -->

This design adds no audit REST endpoint or user-facing audit CLI command. Existing Server and host-administration operations must apply the following admission and completion behavior.

| Surface | Required contract |
|---|---|
| Existing mutation responses and host-command outcomes | Preserve [04's commitment/completion distinctions](04-security-state-and-administration.md#completion-and-audit), including explicit post-commit audit failure; do not report rollback |
| Sensitive reads, exports, and protected session admission | Acknowledge the admission record before releasing protected data; an existing audit record does not replace current authorization |
| Readiness and authorized management inspection | Report aggregate readiness and permitted safe audit-health details; do not expose the file or record contents |
| Host startup configuration | Use the optional audit path and mandatory-audit behavior defined in [Host configuration and local file](#host-configuration-and-local-file) |
| Internal operation/runtime integration | Supply trusted semantic context to the shared facility and receive the actual write result and availability state, not a fire-and-forget logging acknowledgement |

Rust interfaces, SDK/FFI plumbing, record field spelling, and error serialization remain implementation choices. The Server, clients, and host tooling must use a consistent contract without requiring another design of permissions.

### Alternatives Considered

<!-- Describe the alternative designs that were considered or should be considered. Give a justification for why alternative approaches should be rejected if possible. -->

Scraping logs or lifecycle events loses actor and commit semantics and inherits best-effort filtering/drop behavior. Writing through a Reaction would make the evidence depend on the workload being audited. An audit database/outbox or atomic state-and-audit transaction adds durability guarantees not selected here. Requiring remote acknowledgement would couple administration to a separate service and delivery policy. Unconditionally continuing new sensitive work during failure defeats required audit; stopping all work would unnecessarily interrupt previously authorized pipelines and containment.

## Security

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc. -->

| Threat / risk | Simplified example | What is at risk | Mitigation or accepted limit |
|---|---|---|---|
| Ordinary logging is mistaken for audit | `RUST_LOG=off` or a full broadcast buffer discards a grant-change log | Missing security evidence without admission failure | Independent acknowledged writer; log settings and subscribers do not control audit |
| Actor is forged or confused with the Owner | A rejected token names Alice, or Pat edits Bob's component | False attribution | Trusted actor kinds; no unverified claims or Owner impersonation |
| Response status hides a committed change | A binding commits, then audit fails and HTTP returns an error | Unsafe retry and inaccurate history | Separate decision, commitment, runtime, and audit outcomes; no rollback claim |
| Sensitive data leaves before audit succeeds | Export bytes are returned before the writer reports disk failure | Unrecorded new disclosure | Acknowledge admission before releasing sensitive read/session data |
| Audit outage blocks containment | A full audit disk prevents an administrator from revoking access | Continued unwanted access | Narrow authorized Stop/revocation exception; keep normal state/permission checks |
| Maintenance becomes an outage bypass | Recovery or profile reconfiguration proceeds while audit is unavailable | Unaudited new authority | No extra exemption for grants, provider mutations, or additive recovery |
| Existing stream becomes an authority bypass | A stream survives token expiry because its closing record cannot be written | Unauthorized continued disclosure | Close independently of audit; existing-stream continuity never overrides 01/02 |
| Credentials or payloads enter evidence | An error record copies a provider response or request body | Sensitive-data disclosure | Allowlisted bounded metadata and safe reasons; no raw bodies/errors |
| Audit itself becomes a disclosure endpoint | Component log subscribers receive deployment-wide audit records | Cross-resource identity or operation disclosure | Host-protected file, separate from logs/diagnostics; no audit-query API |
| Attacker exhausts audit capacity | Repeated invalid tokens fill queues or disk | Control-plane availability | Bounded detail and explicit unauthenticated-failure aggregates; writer failure is visible |
| Local acknowledgement is overstated | Power fails after an OS write but before durable storage | Lost audit tail | Accepted: no per-record crash durability or atomic state-and-audit guarantee |
| False recovery reopens admission | File exists but writes fail, or a new record joins an unterminated fragment | New unaudited sensitive work | Establish a line boundary and require an independently readable recovery record before reopening admission |
| Host or native code alters audit | Privileged code deletes or rewrites the file | Incomplete or falsified history | OS/deployment protection; accepted trusted-host limit, no tamper-proof claim |
| Remote collection is mistaken for guaranteed delivery | Local writes succeed while a collector is disconnected | Missing remote evidence | Explicit local-only acknowledgement; collection/retention remain operator responsibilities |

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

Backward compatibility, user migration, and data migration are not requirements. Server, client, and host-tool changes must consistently implement the operation outcomes described above.

## Supportability

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->

Use the bounded health reporting, failure counts, and recovery recording defined in [Audit availability and recovery](#audit-availability-and-recovery) and [Bounded failure recording](#bounded-failure-recording). Operational telemetry remains separate from required audit and must not expose credentials, raw errors, or protected payloads.

### Verification

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

| Case | Required result |
|---|---|
| Omitted/custom/relative path, missing directory, inaccessible destination | Stable deployment-relative default, host-only selection, protected creation, no alternate-path or disabled fallback |
| Log filtering, full component-log buffers, no subscribers | Required audit remains independent and acknowledged |
| Caller, administrator editor, development, unauthenticated, host, and automatic work | Correct distinct attribution without raw token claims or Owner impersonation |
| Successful/denied/failed mutation, concurrent operations, recreated resource name | Exact identities and correlated outcomes; no interleaved records or status-only inference |
| Commit failure versus post-commit audit/runtime failure | Honest 04 outcomes; no false success, rollback, or automatic replay |
| Export, sensitive read, Query attach, and session opening | Admission recorded before data release; helper activity does not invent user-component ownership |
| Audit failure during active streams, expiry, and revocation | Existing authorized access follows the table; closure never waits for audit recovery |
| Known outage with Stop, revocation, profile changes, and host recovery | Only the stated exceptions proceed; no permission or durable-state bypass |
| Partial Apply with a constituent audit failure | Correlated per-entry records and 05's honest partial completion; later entries not admitted |
| Authentication flood and writer saturation | Bounded work, visible aggregation/loss, no sampling of protected mutation outcomes |
| Partial writes, blocked writer, rotation, shutdown, and recovery | No false acknowledgement; degraded readiness and bounded recovery; no invented crash-completeness |
| Unterminated tail followed by another record, startup, or reopening | Preserve the damaged bytes, establish a line boundary, and write an independently readable subsequent record; boundary/write failure must not reopen admission |
| Nested secrets, error strings, multiline identifiers, and diagnostic streams | No credential/payload leakage, record injection, or unintended audit exposure |

## Development Plan

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

Current Server shared handlers, authentication/authorization context, the new control-state boundary, host CLI, and Server-managed runtime/SDK hooks are integration points. DrasiLib's `ComponentLogRegistry` and `ComponentEventHistory` provide operational streams, not authoritative security evidence. Their filtering, bounded retention, and fire-and-forget callbacks must not be repurposed as the required audit sink.

| Work item | Deliverable and verification |
|---|---|
| Shared writer and availability | Implement bounded encoding, serialized append, acknowledgement, outage/recovery, and framing; unit-test write failure, saturation, and interrupted tails |
| Semantic operation integration | Connect API, host, runtime, and session boundaries to the shared facility; test attribution, admission ordering, and committed-but-incomplete outcomes |
| Deployment and client integration | Wire startup configuration, readiness, permitted inspection/containment, and caller-visible outcomes; exercise the cross-component failure matrix above |

## Open issues

<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design. -->

**Which details remain for implementation?** Rust/SDK interfaces, record field names, error serialization, file-opening/rotation integration, and numeric bounds remain implementation choices within this contract. They must not introduce new permissions, stronger durability promises, or remote-delivery requirements implicitly.

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

None.

## References

<!-- Optional. Add the design documents and references you use for this document. -->

- [Drasi Server](https://github.com/drasi-project/drasi-server)
- [DrasiLib](https://github.com/drasi-project/drasi-core/tree/main/lib)
