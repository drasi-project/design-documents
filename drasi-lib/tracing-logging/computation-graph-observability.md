# Observability for drasi-lib's Computation Graph - Overview

* Project Drasi - October 22, 2026 - Ruokun Niu (@ruokun-niu)
* Status: proposed replacement design; not implemented.

## Overview

<!-- Provide a succinct high-level description of the component or feature and where/how it fits in the big picture. The overview should be one to three paragraphs long and should be understandable by someone outside Drasi team. -->

**Instrument the existing ComputationGraph once.** Keep `tracing` and `metrics` as emission APIs, with the embedding application (host) owning subscribers, recorders, filtering, exporters, and shutdown.

- [Tracing](computation-graph-tracing.md): operation spans, context propagation, logs, and completed plugin spans.
- [Metrics](computation-graph-metrics.md): counts, identities, durations, and recorder callbacks.
- [Server integration](../../drasi-server/computation-graph-observability-integration.md): configuration, export, and process metrics.

Names, defaults, bridge details, and delivery slices remain proposals. The original documents remain unchanged for comparison.

## Terms and definitions

| Term | Definition |
|------|------------|
| Host | Drasi Server or another application embedding drasi-lib. |
| Native plugin | A component loaded through the computation ABI|

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->

Help developers and operators understand how work flows through ComputationGraph, identify bottlenecks, and diagnose failures:

- **Connected tracing:** follow a change across sources, transformers, queries, and sinks, including asynchronous and native-plugin boundaries.
- **Operational visibility:** monitor throughput, latency, queue pressure, errors, and component health.
- **Reusable instrumentation:** give components common tracing and metrics, with optional internal detail and host-owned collection and export.

## Design requirements


### Requirements

<!-- List design goals or constraints that impact the design. Examples include extensibility points, maintainability, code reuse, considering if your feature needs to be taken down-level and coherency with related code. -->


1. **Facade-only:** drasi-lib does not install a `tracing::Subscriber` or `metrics::Recorder`.
2. **Connected traces:** context crosses every async task and dynamic-plugin boundary. For a source change event, we should observe the full end-to-end trace that flows from Sources through Lib and to Reactions.
3. **Stable identity:** spans and metrics carry component identifiers as fields or attributes.
4. **Backend-neutral metrics:** names follow OpenTelemetry conventions under `drasi.`; units and
   instrument type remain metadata.
5. **Compatible logging:** `ComponentLogLayer`, `ComponentLogRegistry`, and the `log` bridge remain
   functional.
6. **Host control:** the host controls filtering, sampling, export, and shutdown flush.

## Design

### High-level design

<!-- Provide a high-level description, using diagrams as appropriate, and top-level explanations to convey the architectural/design overview. Don't go into a lot of details yet but provide enough information so that the relationship between these components and other components. Call out or highlight new components that are not part of this feature (dependencies). This diagram generally treats the components as black boxes. Provide a pointer to a more detailed design document, if one exists. If you have the alternatives for your design, describe alternative solution too. -->

- **Trace context travels with events and operation calls** so related spans share a trace.
- **Completed spans, log events, and metric updates go to host collection**, separately from event delivery.


### Detail design

#### 1. What changes

| Keep | Adapt |
|------|-------|
| Graph-owned execution | Instrument shared component/pipe boundaries, not a fixed five-task pipeline. |
| Logs, inspection, existing measurements | Preserve them; add explicit instance and queue identity. |
| Native ABI extensions | One shared SDK telemetry bridge for all component roles and both signals. |
| Host collection/export | Keep subscribers, recorders, and OTLP/Prometheus ownership unchanged. |

#### 2. Proposed shared profiles

| Profile | Collection |
|---------|------------|
| `off` | No new traces/metrics; ordinary logs and inspection continue. |
| `basic` | Core operation spans and operational metrics, excluding deferred instruments. |
| `debug` | Basic plus detailed pipe/query spans and permitted diagnostics. |
| `persistence` | Basic plus supported recovery/storage observations. |

Profiles select filters and collection flags, not missing capabilities. Sampling applies only to traces.

### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable.

- Describe the REST APIs in detail for new resource types or updates to existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components. -->

Proposed `TelemetryProfile::resolve()` returns trace directives, metric exclusions, and collection flags. `DrasiLibBuilder::with_telemetry(...)` and direct graph builders receive settings without installing globals. 

Initialization and SDK contracts are detailed in [tracing](computation-graph-tracing.md#api-design) and [metrics](computation-graph-metrics.md#api-design).

### Alternatives Considered

<!-- Describe the alternative designs that were considered or should be considered. Give a justification for why alternative approaches should be rejected if possible. -->

Ordinary-adapter-only instrumentation misses native graph work. Plugin-owned exporters duplicate configuration and bypass host controls.

## Security

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc. -->

No payloads, query text, or secrets in framework telemetry. Validate plugin records; the host owns export security.

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

Preserve logs, inspection, and business formats. The tracing design covers initialization migration; the metrics design identifies catalogue changes. Native and legacy ABIs remain distinct.


### Verification

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

Verify the [tracing](computation-graph-tracing.md) and [metrics](computation-graph-metrics.md) contracts, including multi-instance isolation, replay, shutdown, and disabled overhead.

## Development Plan (TO BE UPDATED)

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

| Slice | Deliverable | Relative complexity |
|-------|-------------|---------------------|
| A | Composable logging, profiles, identity, lifecycle/node/pipe metrics, basic host export | Medium |
| B | In-process event tracing, transient context, monotonic timing, query detail, replay-safe behavior | High |
| C | Native SDK/Host bridge, protocol context, plugin-internal parity, valid handled-branch timing | High |

Proposed slices include tests/documentation. Profiles move into the baseline; end-to-end timing waits for validated completion.


## References

<!-- Optional. Add the design documents and references you use for this document. -->

- [Original observability proposal](https://github.com/drasi-project/design-documents/pull/7)
- Original [overview](00-observability-overview.md), [tracing](01-tracing.md), and [metrics](02-metrics.md)
