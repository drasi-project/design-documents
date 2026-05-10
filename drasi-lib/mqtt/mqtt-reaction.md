# MQTT Reaction Design

* Project Drasi - April 7, 2026 - Ahmed Kamal (@ahmed-kamal)

## Overview

The MQTT Reaction in Drasi publishes continuous query result changes to MQTT brokers so external systems can consume Drasi output using a lightweight pub/sub protocol. This enables integration with IoT systems, edge gateways, automation platforms, and event-driven services that already rely on MQTT topics.

This design defines a v1 MQTT Reaction built on the Drasi Reaction SDK. It focuses on reliable publish behavior for username/password brokers, predictable topic and payload mapping, and explicit error and lifecycle semantics so failures surface quickly rather than freezing pipelines silently.

## Terms and definitions

| Term | Definition |
|------|------------|
| MQTT | Message Queuing Telemetry Transport, a lightweight publish/subscribe protocol |
| Broker | MQTT server endpoint that receives published messages and routes them to subscribers |
| Topic | Hierarchical routing key used by MQTT for message distribution |
| QoS | MQTT Quality of Service level controlling delivery guarantees |
| Retain | MQTT flag that asks the broker to keep the last message on a topic for new subscribers |

## Objectives

### User scenarios

**IoT engineer publishing query results to IoT consumers**

An IoT engineer deploys a Drasi continuous query and wants every result change to be published to MQTT so downstream IoT devices and services can react immediately. They need to:

- Configure broker connectivity and authentication.
- Control topic naming by query and event type.
- Choose delivery guarantees per route (QoS, retain, message expiry).

### Goals

1. Add an MQTT Reaction for drasi-core that subscribes to queries and emits result changes to MQTT brokers.
2. Support MQTT v5 (default) and MQTT v3.1.1, selected explicitly via a `protocol_version` config field. There is no in-CONNACK auto-fallback; rumqttc constructs separate clients per protocol version at startup.
3. Support per-query configuration of topic, payload template, QoS, retain, and v5 message expiry.
4. TLS/SSL transport support for v1 (server-cert verification only). mTLS / client-certificate authentication is deferred to v2 (no shipping `IdentityProvider` constructs `Credentials::Certificate` today).
5. v1 supports username/password authentication via `PasswordIdentityProvider`. AWS IoT and Azure IoT broker auth (token / mTLS) are deferred to v2; the shipping `AwsIdentityProvider` and `AzureIdentityProvider` emit database-scope tokens, not MQTT-broker tokens, and need purpose-built MQTT providers.
6. Validate interoperability with Mosquitto (and HiveMQ self-hosted / Cloud where username/password is sufficient).
7. Implement a testing suite covering unit tests and integration tests against a real broker via testcontainers.

### Non-Goals

v1 explicitly does not address (these are tracked for v2 or beyond):

- mTLS / client-certificate authentication. No shipping `IdentityProvider` constructs `Credentials::Certificate` (`lib/src/identity/mod.rs:94-101`). v2 must add a file-based cert provider (preferred, in `components/identity/`) or a sanctioned inline-cert escape hatch.
- MQTT-broker token authentication (Azure SAS, AWS SigV4). Needs purpose-built MQTT identity providers including a credential-refresh story (rumqttc caches credentials in `MqttOptions`; rotation needs deliberate disconnect-and-reconnect).
- AWS IoT Core, Azure IoT Hub, and other cert-only or token-only brokers. Direct consequence of the two items above.
- Adaptive batching. MQTT publishes one message per PUBLISH packet, each potentially to a different topic with its own QoS and retain flag. The Drasi adaptive-batching framework is designed to fold N events into one wire call, which MQTT does not support; the framework is not applicable to this reaction.
- `{{drasi.sequence}}` template variable. Reserved for a future Drasi version. v1 templates must not synthesize a process-local substitute under this name.
- Cross-restart exactly-once dedup helpers (correlation IDs derived from sequence). Same reason.
- Last Will and Testament (LWT) and birth/online presence messages. Purely additive when added.
- MQTT v5 user properties (per-publish key-value metadata). Purely additive; v1 publish path goes through rumqttc's `PublishProperties` so the v2 addition is a one-line diff.
- Bootstrap-driven retained-message seeding. v1 does not seed the broker on `bootstrapCompleted`.
- Metrics infrastructure. No Drasi reaction emits metrics today; v1 observability is logs only.

## Design requirements

### Requirements

- Reaction implementation aligns with patterns followed by other drasi-core reactions (`routes` plus `default_template`, `TemplateSpec<T>` plus `QueryConfig<T>`, `IdentityProvider` from `lib/src/identity`).
- Payload format defaults to JSON; templates are arbitrary Handlebars and may produce any byte sequence.
- Reaction supports a global config (broker URL, auth, TLS) and per-query plus per-operation route templates.

### Dependencies

- drasi-core reaction implementation patterns.
- `rumqttc` crate for MQTT v3.1.1 and v5 client implementations.
- `url` crate for broker-URL parsing.
- Broker availability and protocol support in test environments (Mosquitto via testcontainers).

### Out of scope

Same as Non-Goals above.

## Design

### High-level design

The MQTT Reaction receives query result diffs from the Drasi Query and transforms each diff into an MQTT publish composed of:

- Target MQTT topic (rendered Handlebars template).
- Payload (rendered Handlebars template, default raw JSON of the diff).
- MQTT publish options (QoS, retain, optional v5 message expiry).

The reaction maintains a single long-lived rumqttc `AsyncClient` plus its cooperating `EventLoop` task for the entire reaction lifetime. rumqttc owns reconnection with autonomous exponential backoff and jitter; the reaction stays in `Running` while reconnects are in progress and only transitions to `Error` on enumerated terminal CONNACK or Disconnect reason codes.

### Architecture diagram

```text
   Drasi Query Container
            |
            | result change events (insert/update/delete)
            v
       MQTT Reaction
            |
            | MQTT publish
            v
       MQTT Broker
            |
            v
   Subscriber Applications
```

### Detail design

#### Startup validation

Startup validation only catches structural errors that cannot resolve at runtime:

- Each key in `routes` must reference a query that appears in `queries`. Targeting a non-existent query is a startup error.
- Each `MqttExtension.topic` template must be a non-empty string (a static check on the template, not the rendered output).
- The `url` field must parse as a valid MQTT broker URL with a supported scheme (`mqtt`, `mqtts`, `ws`, `wss`); an unsupported scheme is a startup error.
- When the URL scheme is `mqtts` or `wss`, `tls` must be `Some(...)`; mismatch is a startup error. Conversely, a plain scheme with `tls: Some(...)` is also a startup error (defensive, catches misconfiguration).
- When `tls.client_auth` is `Some(...)`, startup fails with a clear "deferred to v2" error.
- `keep_alive`, when set, must be greater than 5 (rumqttc's documented floor).
- `protocol_version` must be `V5` or `V3_1_1`.

Startup does not require every query in `queries` to have a configured route. A query with no `routes` entry and no `default_template` still starts cleanly; it simply produces no MQTT publishes (each diff is dropped with a WARN log per the runtime resolution chain below).

#### Per-query configuration

Relying on the existing `QueryConfig<T>` and `TemplateSpec<T>` framework from `drasi_lib::reactions::common`, the reaction defines an `MqttExtension` extension type carrying the per-publish MQTT-specific fields:

```rust
#[derive(Default, Clone, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "snake_case")]
pub struct MqttExtension {
    /// Target MQTT topic. Handlebars template, rendered against the same context as `template`
    /// (`after`, `before`, `query_name`, `operation`, `timestamp`).
    pub topic: String,

    /// QoS level. Default: `AtLeastOnce` (1). Only `AtMostOnce` (0) and `AtLeastOnce` (1) are valid;
    /// the enum has no `ExactlyOnce` variant. See "QoS limits" below.
    #[serde(default = "default_qos")]
    pub qos: MqttQoS,

    /// Retain flag. Default: false.
    #[serde(default)]
    pub retain: bool,

    /// Publish a zero-byte payload regardless of `template`. Default: false.
    /// Pair with `retain: true` on a `deleted` config to clear a retained state-topic message.
    /// Disambiguates from `template: ""`, which means "use the raw-JSON default".
    #[serde(default)]
    pub empty_payload: bool,

    /// MQTT v5 message expiry interval in seconds. Default: None (broker default, never expires).
    /// Silently omitted on v3.1.1 connections.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub message_expiry_interval: Option<u32>,
}

fn default_qos() -> MqttQoS {
    MqttQoS::AtLeastOnce
}

#[derive(Default, Clone, Copy, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum MqttQoS {
    AtMostOnce,
    #[default]
    AtLeastOnce,
}
```

The payload template is the `template` field of the surrounding `TemplateSpec<MqttExtension>`. Per-query routing follows the standard log/SSE/storedproc pattern: `routes: HashMap<String, QueryConfig<MqttExtension>>` plus `default_template: Option<QueryConfig<MqttExtension>>`. The reaction implements `TemplateRouting<MqttExtension>` to get `get_template_spec(query_id, operation)` for free.

Example for an alert state-topic, with the standard MQTT delete-clears-retained pattern:

```rust
let spec_add = TemplateSpec::<MqttExtension> {
    template: "[{{query_name}}] + {{after.floor}}: {{after.temp}}".to_string(),
    extension: MqttExtension {
        topic: "alerts/{{after.machine_id}}".to_string(),
        qos: MqttQoS::AtLeastOnce,
        retain: true,
        empty_payload: false,
        message_expiry_interval: None,
    },
};

let spec_update = TemplateSpec::<MqttExtension> {
    template: "[{{query_name}}] ~ {{after.floor}}: {{before.temp}} {{after.temp}}".to_string(),
    extension: MqttExtension {
        topic: "alerts/{{after.machine_id}}".to_string(),
        qos: MqttQoS::AtLeastOnce,
        retain: true,
        empty_payload: false,
        message_expiry_interval: None,
    },
};

// Standard MQTT delete-clears-retained pattern: empty payload + retain=true tells the
// broker to drop the retained message for this state topic.
let spec_delete = TemplateSpec::<MqttExtension> {
    template: String::new(), // ignored when empty_payload is true
    extension: MqttExtension {
        topic: "alerts/{{before.machine_id}}".to_string(),
        qos: MqttQoS::AtLeastOnce,
        retain: true,
        empty_payload: true,
        message_expiry_interval: None,
    },
};

let query_config = QueryConfig::<MqttExtension> {
    added: Some(spec_add),
    updated: Some(spec_update),
    deleted: Some(spec_delete),
};

// Insert the `query_config` into the MQTT reaction config `routes` map keyed by query id.
```

**Retained state clearing on delete.** For state topics where each publish overwrites prior retained state on the same topic (one retained message per entity), a delete event must explicitly clear the broker's retained copy or stale state lingers indefinitely. The reaction does not auto-apply this; users opt in by setting `empty_payload: true` and `retain: true` on the `deleted` config (see the example above). The reaction publishes a zero-byte payload with the retain flag set; brokers interpret this as "drop the retained message on this topic." For event-style topics where each publish is its own message and `retain: false` everywhere, no clearing is needed.

**`template == ""` does not produce a zero-byte payload.** Under existing reaction convention (SSE, log) it means "use the raw-JSON default for the result diff." Only `empty_payload: true` produces zero bytes.

**Runtime resolution chain.** For each diff:

1. **Spec resolution**: try `routes[query_id][operation]` first; if absent, try `default_template[operation]`; if neither resolves, drop the diff with a WARN log (cannot publish without a topic). Do not auto-fabricate a topic.
2. **Payload rendering** (within a resolved spec): if `template` is non-empty, render via Handlebars. If `template == ""`, publish the raw JSON serialized representation of the `ResultDiff`, matching the SSE and log convention.

These are two distinct fallback layers: the spec chain can drop with WARN; the payload chain falls back to raw JSON within an already-resolved spec.

**Operation handling rules.**

- `Add`, `Update`, `Delete`: route through their matching template.
- `Aggregation { before, after }` (queries with GROUP BY): route through the `updated` template with `operation = "AGGREGATION"`. Both `before` (optional, may be absent on first emission) and `after` are populated in the template context.
- `Noop`: skip silently with a DEBUG log.
- Empty `QueryResult`s carrying control signals (`bootstrapStarted`, `bootstrapCompleted`, `running`): skip silently with a DEBUG log.

#### Topic rendering and validation

The `topic` field on `MqttExtension` is a Handlebars template, rendered per-publish against the same context as `template`. After rendering, the topic is validated:

- Reject if it contains MQTT wildcard characters `#` or `+` (subscribe-side only).
- Reject if it contains an embedded null byte (`\0`); the MQTT UTF-8 string spec prohibits these.
- Reject if it contains empty segments (e.g. `foo//bar`).
- Reject if the UTF-8 byte length exceeds 65535.
- **Reject-on-slash interpolation**: at startup, record the number of `/` characters in each template literal. On every publish, after rendering, recount `/` characters in the rendered string. If the rendered count exceeds the template-literal count, the difference came from interpolation; reject.

On any rejection, log a WARN with the rendered topic and the failure reason, skip the diff, continue to the next event. Do not transition to `Error`. Bad templates produced by transient data values must not stop the pipeline.

Example of the reject-on-slash rule:

```
Template:   alerts/{{after.machine_id}}/new      (2 literal slashes)
machine_id: "site-A/machine-3"
Rendered:   alerts/site-A/machine-3/new          (3 slashes)
Result:     rejected with WARN; subscriber expecting `alerts/+/new` is not silently broken
```

Implementation note: this is a count comparison, not a structural diff. The template literal's `/` count is a strict floor for the rendered topic's `/` count under Handlebars (the engine inserts character sequences but cannot remove literal characters from the template), so any excess slashes came from value interpolation. The check is one extra `str::matches('/').count()` per publish.

The v1 stance is deliberately strict; loosening to accept-and-document is non-breaking and can be revisited if real users surface a hierarchical-interpolation requirement.

#### Template helpers and JSON safety

handlebars-rust HTML-escapes raw `{{ }}` interpolation by default (escapes `"`, `<`, `>`, `&`, and others). HTML escaping is fine for plain-text payloads but unsafe inside JSON syntax: a value containing a `"` becomes `&quot;` and breaks the JSON structure.

The reaction registers a `{{json ...}}` helper that writes raw via `out.write` and produces JSON-valid output (quoted strings, numbers, nulls, arrays, objects). Recommended patterns:

- **Full-payload JSON of the diff**: `template: '{{json after}}'` (or `{{json before}}` for delete).
- **Hand-built JSON template**: `'{"id": {{json after.id}}, "temp": {{json after.temp}}}'`.
- **Plain text**: `'{{after.id}} -> {{after.temp}}'`. Raw `{{ }}` is fine here.

**Do not** write hand-built JSON with raw `{{ }}` inside string values (e.g., `'{"id": "{{after.id}}"}'`). This silently corrupts the JSON when `after.id` contains any of `" < > &`.

#### Template context variables

| Variable | Type | Populated for |
|---|---|---|
| `after` | object | ADD, UPDATE, AGGREGATION |
| `before` | object | UPDATE, DELETE, AGGREGATION (may be absent on first AGGREGATION emission) |
| `query_name` | string | always |
| `operation` | string ("ADD", "UPDATE", "DELETE", "AGGREGATION") | always |
| `timestamp` | RFC 3339 / UTC string | always (the `QueryResult.timestamp` at which the diff was emitted) |

Reserved for a future Drasi version: `{{drasi.sequence}}` (monotonic per-emission). v1 templates must not synthesize a substitute under this name.

#### Idempotency and downstream consumers

v1 does not provide a stable cross-restart dedup key. After a Drasi restart, the same diff may be republished; downstream MQTT subscribers must handle this themselves. The recommended pattern is **last-write-wins keyed on `element.id`** (available as `{{after.id}}` for ADD/UPDATE and `{{before.id}}` for DELETE). State-topic patterns naturally converge under last-write-wins; event-topic consumers need their own dedup ledger if exact-once is required.

A monotonic sequence ID is planned for a future Drasi version and will be exposed as `{{drasi.sequence}}`. v1 templates must not depend on this name; in particular, do not synthesize a process-local counter under `{{drasi.sequence}}`, as that collides with the future implementation and gives downstream consumers a false sense of dedup safety.

#### QoS limits

Only QoS 0 (`AtMostOnce`) and QoS 1 (`AtLeastOnce`) are supported. There is no `ExactlyOnce` variant on `MqttQoS`. Configurations that attempt QoS 2 fail to deserialize. True end-to-end exactly-once across process crashes is impossible without external coordination, and Drasi does not provide it.

Downstream consumers requiring exactly-once must apply their own dedup; see "Idempotency and downstream consumers."

#### MQTT v5 publish properties (hardcoded defaults)

Every publish on a v5 connection sets `content_type = "application/json"` and `payload_format_indicator = 1` (UTF-8). These defaults match the HTTP reaction's opinionated `Content-Type: application/json` behavior and are not user-configurable in v1.

On v3.1.1 connections, both properties are silently omitted (the protocol does not carry them).

Trade-off acknowledgment: zero-byte deletes (`empty_payload: true`) are mislabeled as `application/json` under this rule. This is acceptable for v1 because the overwhelmingly common case is JSON, and downstream consumers parsing zero bytes will fail uniformly regardless of the labeled content type. v2 may make these configurable.

Implementation note: every v5 publish goes through rumqttc's `PublishProperties` (not the bare `client.publish(...)` overload) so that adding MQTT v5 user properties later (deferred to v2) is a one-line addition. Do not call bare `client.publish(...)` without a properties argument; that path silently drops the v5 property infrastructure.

#### Bootstrap and retained state

Drasi's bootstrap path does not dispatch per-element diffs to reactions; the reaction only receives a single `bootstrapCompleted` control signal once all sources have completed bootstrap. v1 therefore does not seed the broker with retained messages for the initial snapshot; retained state accumulates only as live diffs arrive after `bootstrapCompleted`.

For state-topic patterns (`retain: true`, one entity per topic), this means a fresh broker appears empty on first reaction startup, even though the underlying query has a fully populated result set. Retained state populates lazily as updates arrive.

Bootstrap-driven retained-message seeding is deferred to a later version (out of scope for v1 per "Non-Goals").

#### MQTT v5 and v3.1.1

The reaction supports MQTT v5 (default) and MQTT v3.1.1, selected explicitly via the `protocol_version` field on `MqttReactionConfig`. rumqttc has separate `AsyncClient` constructors for each version selected at construction time; there is no in-CONNACK auto-fallback. Users targeting older brokers configure `protocol_version: V3_1_1` once at startup.

The connection is managed by [`rumqttc`](https://github.com/bytebeamio/rumqtt). A single long-lived `AsyncClient` of the configured protocol version is created at startup and reused for all publish calls. rumqttc's `EventLoop::poll()` owns reconnection (see "Reconnect behavior" below).

The broker connection is configured through `MqttReactionConfig`:

```rust
#[derive(Clone, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "snake_case")]
pub struct MqttReactionConfig {
    /// Broker URL. Examples:
    ///   `mqtt://broker.example.com:1883`
    ///   `mqtts://broker.example.com:8883`
    ///   `ws://broker.example.com:80/mqtt`
    ///   `wss://broker.example.com:443/mqtt`
    /// Scheme selects transport: `mqtt` = plain TCP, `mqtts` = TLS over TCP,
    /// `ws` = WebSocket, `wss` = TLS over WebSocket. Default ports per scheme: 1883 / 8883 / 80 / 443.
    /// Parsed at startup; an unsupported scheme is a startup error.
    pub url: String,

    /// Optional client ID. Defaults to `drasi-mqtt-{reaction_id}` for deterministic
    /// session identity across restarts. Required by AWS IoT and Azure IoT Hub;
    /// required for resuming persistent sessions when `clean_start: false`.
    /// Do NOT default to a random UUID; that orphans broker sessions on every restart.
    #[serde(default)]
    pub client_id: Option<String>,

    /// MQTT protocol version. Default: V5. Selected once at startup; no auto-fallback.
    #[serde(default)]
    pub protocol_version: MqttProtocolVersion,

    /// Per-query routing: query_id -> per-operation template config.
    /// Matches the convention in log, SSE, HTTP, and storedproc reactions.
    #[serde(default)]
    pub routes: HashMap<String, QueryConfig<MqttExtension>>,

    /// Default template fallback when a query has no entry in `routes`.
    #[serde(default)]
    pub default_template: Option<QueryConfig<MqttExtension>>,

    /// Identity provider for authentication. Wired via `with_identity_provider(...)` builder.
    /// In v1, supply a `PasswordIdentityProvider` (from `lib/src/identity/password.rs`)
    /// for username/password brokers. Token and certificate providers are deferred to v2.
    #[serde(skip)]
    pub identity_provider: Option<Box<dyn IdentityProvider>>,

    /// TLS tuning. Required when the URL scheme is `mqtts` or `wss`; ignored otherwise.
    /// Startup validation: scheme says TLS but `tls: None` is an error;
    /// scheme is plain but `tls: Some(...)` is also an error (defensive).
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub tls: Option<MqttTlsConfig>,

    /// Capacity of the rumqttc internal channel between AsyncClient and EventLoop.
    /// Default: 100. Sized for sustained throughput; see "Backpressure".
    #[serde(default = "default_event_channel_capacity")]
    pub event_channel_capacity: usize,

    /// Maximum outgoing inflight QoS 1 messages. Default: rumqttc default.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub max_inflight: Option<u16>,

    /// Keep-alive interval in seconds (PingReq cadence). Default: 60.
    /// Lower for IoT-over-NAT scenarios where idle connections get reaped.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub keep_alive: Option<u64>,

    /// Clean session start. Default: true.
    /// Set false plus `client_id` to resume a persistent broker-side session;
    /// note that v1 has no client-side in-flight persistence (see "Shutdown semantics").
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub clean_start: Option<bool>,

    /// MQTT v5 connection timeout in milliseconds. Default: rumqttc default (5000).
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub conn_timeout: Option<u64>,

    /// MQTT v5 session expiry interval. Meaningful only with `clean_start: false`.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub session_expiry_interval: Option<u32>,
}

#[derive(Default, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum MqttProtocolVersion {
    #[default]
    V5,
    V3_1_1,
}

fn default_event_channel_capacity() -> usize {
    100
}
```

#### Reconnect behavior

Reconnect is owned entirely by rumqttc's `EventLoop::poll()`, which performs unbounded retries with exponential backoff and jitter. The reaction stays in `Running` while reconnects are in progress; transient network outages, broker restarts, and DNS blips are not reaction errors. The reaction only transitions to `Error` on terminal CONNACK or Disconnect reason codes (see "CONNACK and Disconnect classification" below).

`start()` returns `Ok` and lets the EventLoop task drive connectivity. An unreachable broker at startup is a transient outage, not a terminal cause; rumqttc's `poll()` does not distinguish "first connect" from "reconnect."

#### EventLoop task lifecycle

rumqttc separates `AsyncClient::publish()` (enqueue-only, returns immediately) from `EventLoop::poll()` (network I/O, ack handling, keepalive, reconnect). These run in two cooperating tasks; fusing them is impossible. `ReactionBase` only tracks the main `processing_task`, so the EventLoop task is a bespoke per-reaction concern. The closest precedent is the SSE reaction's `task_handles` pattern (`components/reactions/sse/src/sse.rs:80-85`).

- **Storage**: `MqttReaction` holds `eventloop_task: Arc<tokio::sync::Mutex<Option<JoinHandle<()>>>>` for the spawned poll loop.
- **Shutdown**: `stop()` aborts `eventloop_task` after `base.stop_common()` returns; sequence mirrors SSE (`sse.rs:580-599`). On restart there is no rumqttc in-flight persistence, so any QoS 1 publishes that left Drasi's priority queue but had not yet been ACKed are dropped (see "Shutdown semantics").
- **Transient versus terminal poll errors**: see "CONNACK and Disconnect classification" below for the enumeration. Transient errors continue the loop; terminal errors transition the reaction to `Error` then exit the task cleanly.
- **Liveness**: the task spawns with a panic-catching wrapper that, on panic or unexpected exit, transitions the reaction to `Error` with a descriptive message. Without this, a panicked EventLoop fills rumqttc's internal channel, `publish().await` blocks indefinitely, priority-queue backpressure cascades upstream, and the pipeline freezes silently.

#### CONNACK and Disconnect classification

| Bucket | Classification | rumqttc surface |
|---|---|---|
| Auth and protocol failures | Terminal (transition to `Error`) | `ConnectReasonCode::NotAuthorized`, `BadUserNameOrPassword`, `BadAuthenticationMethod`, `Banned`, `ProtocolError`, `MalformedPacket`, `UnsupportedProtocolVersion` |
| Network failures | Transient (stay `Running`, rely on rumqttc reconnect) | `ConnectionError::Io`, `Timeout`, `NetworkTimeout`, TCP refusal, DNS failure |
| Server side: capacity | Transient (stay `Running`) | `ConnectReasonCode::ServerUnavailable`, `ServerBusy`, `QuotaExceeded`. Broker is reachable and responding; capacity will recover. |
| Server side: deliberate eviction | Transient with informational log | `DisconnectReasonCode::AdministrativeAction`, `SessionTakenOver`. Broker may bring us back; if not, network reconnect retries cover it. |

Rationale on borderline cases:

- `ServerUnavailable` and `ServerBusy` are graceful overload signals; reconnect is the right answer.
- `QuotaExceeded` could indicate either a transient burst or a config-level limit; transient is the safer default. Operators see WARN logs and can intervene.
- `AdministrativeAction` and `SessionTakenOver` are intentional broker-side decisions, not config errors. Logging is sufficient; the reaction recovers naturally.

On terminal causes, the EventLoop task logs ERROR with the rendered reason code, transitions the reaction status to `Error`, and exits cleanly. The `processing_task` is then woken via the shutdown channel so `stop_common()` semantics apply.

**v3.1.1 reason codes.** The table above covers v5; v3.1.1's CONNACK carries a smaller, fixed reason-code set. Terminal causes: `RefusedProtocolVersion`, `BadClientId`, `BadUserNamePassword`, `NotAuthorized`. Transient (rely on reconnect): `ServerUnavailable` (broker is alive but overloaded). v3.1.1 has no `DisconnectReasonCode` equivalent; broker-initiated disconnects surface as `ConnectionError::Io` and are handled as transient network failures.

#### Backpressure

The reaction relies on Drasi's existing channel-mode backpressure: query priority queue plus subscriber channel buffers. The processing loop awaits `publish().await` naturally, so a full rumqttc outbound buffer or unreachable broker propagates back through the priority queue. The blocking point is rumqttc's internal channel between `AsyncClient` and `EventLoop`, sized by `event_channel_capacity` (default 100).

No timeout-and-drop wrapper is added around `publish().await`. That would break at-least-once semantics for QoS 1 (a slow broker would silently lose messages) without giving the priority queue a chance to backpressure correctly.

**QoS-asymmetric backpressure**: the mechanism above works as designed for QoS 1, where in-flight ACK limits cause `publish().await` to block once the inflight window fills. For QoS 0 there are no ACKs and the publish only backpressures at the OS socket buffer level; this is best-effort by protocol, not a Drasi limitation. Operators choosing QoS 0 accept this asymmetry implicitly.

#### Per-publish error handling

On `publish().await` returning `Err` (packet-too-large, broker reject under v5 reason code, payload encoding error, or any other publish-scoped failure), the reaction logs a WARN with the rendered topic and the error reason, then continues to the next diff. The reaction does not:

- Retry the publish inline. Retries belong to rumqttc's reconnect path, not the application loop.
- Validate payload size upfront. rumqttc's `max_packet_size` and the broker's MQTT 5 max-packet negotiation already surface oversize publishes; duplicating the check is dead code.
- Transition to `Error`. Single-publish failures are not reaction errors; only terminal connection-level failures are, per "CONNACK and Disconnect classification."

This mirrors the HTTP reaction's per-publish policy (`components/reactions/http/src/http.rs:443-460` for precedent).

#### Shutdown semantics

`stop()` aborts the EventLoop task after `base.stop_common()`. Publishes that have left Drasi's priority queue but not yet been ACKed by the broker (sitting either in rumqttc's internal AsyncClient-to-EventLoop channel, or in QoS 1 in-flight tracking) are dropped. v1 has no persistence layer for rumqttc's in-flight state.

Even `clean_start: false` with a server-side preserved session does not give cross-restart QoS 1 redelivery: the rumqttc client does not persist its in-flight tracking across process boundaries, so after restart the client does not know what was in flight and cannot retransmit. This is consistent with the "no stable cross-restart dedup key" position in "Idempotency and downstream consumers."

Operators who need higher delivery durability should run with QoS 1 plus a downstream subscriber that handles duplicates, and accept that a Drasi restart will lose any publishes that were in flight at shutdown.

#### Logging

Uses the `log` crate macros, matching existing reaction convention (HTTP, SSE, log reactions all do this; `components/reactions/http/src/http.rs:20` for precedent). Every log message is prefixed with `[{reaction_id}]` for cross-reaction grep-ability.

| Level | When |
|---|---|
| INFO | Connect, disconnect, status transitions (Starting, Running, Stopping, Stopped, Error) |
| WARN | Per-publish errors, invalid topic after rendering (including reject-on-slash), transient `EventLoop::poll()` errors |
| DEBUG | Each successful publish (rendered topic, payload size in bytes, qos), control-signal skips |
| ERROR | Terminal failures that transition to `Error` (auth rejection, malformed config, terminal v5 reason codes) |

Metrics infrastructure (Prometheus, OpenTelemetry, etc.) is out of scope for v1; no Drasi reaction emits metrics today. Logs are the v1 observability surface.

#### Security

Security splits into two independent axes: transport security (how bytes travel) and authentication (who is connecting). They combine freely.

##### Transport: `MqttTlsConfig`

Transport selection is driven entirely by the URL scheme on `MqttReactionConfig.url`. There is no separate transport config field, no `MqttTransportMode` enum: the URL scheme is the single source of truth. The reaction parses the URL at startup, picks the rumqttc transport constructor accordingly (TCP, TLS, WebSocket, WebSocket-over-TLS), and routes through the appropriate `MqttOptions`.

When the URL scheme is `mqtts` or `wss`, TLS tuning comes from `MqttTlsConfig`:

```rust
#[derive(Default, Clone, Serialize, Deserialize)]
#[serde(deny_unknown_fields, rename_all = "snake_case")]
pub struct MqttTlsConfig {
    /// Custom CA bundle (PEM). `None` (default) uses the system CA store, which is the right
    /// answer for HiveMQ Cloud and any public-CA broker.
    #[serde(default)]
    pub ca: Option<Vec<u8>>,

    /// Optional ALPN protocol list (e.g. `vec![b"mqtt".to_vec()]` for HiveMQ Cloud).
    /// Without it, some HiveMQ Cloud handshakes silently fail.
    #[serde(default)]
    pub alpn: Option<Vec<Vec<u8>>>,

    /// Reserved for v2: mTLS client cert + key (PEM). v1 rejects `Some(...)` at startup
    /// with a "deferred to v2" error; the field is in the struct so the v2 addition is non-breaking.
    #[serde(default)]
    pub client_auth: Option<(Vec<u8>, Vec<u8>)>,

    /// Dev-only: skip broker certificate verification. Default `false`. When `true`,
    /// the reaction emits a loud WARN log on every connect. For local Mosquitto without
    /// proper certs; never use in production.
    #[serde(default)]
    pub accept_invalid_certs: bool,
}
```

##### Authentication: delegate to Drasi's `IdentityProvider`

MQTT application-layer authentication (CONNECT username and password) is supplied by Drasi's existing `IdentityProvider` trait at `lib/src/identity/mod.rs`. The reaction does not redefine the trait, embed raw credentials in its config schema, or implement custom credential-resolution logic. v1 supports username/password brokers (Mosquitto, EMQX, HiveMQ self-hosted and Cloud); users wire `PasswordIdentityProvider` (`lib/src/identity/password.rs`) via the standard `with_identity_provider(...)` builder, exactly as the storedproc reactions do (see `components/reactions/storedproc-postgres/src/reaction.rs` for precedent).

`get_credentials(&self, &CredentialContext)` returns a `Credentials` enum with variants `UsernamePassword`, `Token`, and `Certificate`. v1 only consumes `Credentials::UsernamePassword`; encountering `Token` or `Certificate` returns a deferred-to-v2 error. Token and certificate variants exist in the enum today but no shipping `IdentityProvider` constructs `Credentials::Certificate`, and the shipping `AwsIdentityProvider` and `AzureIdentityProvider` emit database-scope tokens, not MQTT-broker tokens. v2 will introduce purpose-built MQTT identity providers.

`get_credentials` is called on every new connection or reconnection attempt, so providers that rotate credentials (future Azure SAS, AWS SigV4) work cleanly once the v2 providers exist.

### API Design

The expected `MqttReaction` structure:

```rust
pub struct MqttReaction {
    base: ReactionBase,
    config: MqttReactionConfig,
    eventloop_task: Arc<tokio::sync::Mutex<Option<tokio::task::JoinHandle<()>>>>,
    // additional internal state populated during implementation:
    // rumqttc AsyncClient handle, parsed broker URL, derived transport selection, etc.
}
```

The `MqttReactionBuilder` follows the conventions used by HTTP, log, and the storedproc builders, including the `with_identity_provider(...)` method.

## Compatibility impact

No compatibility impact for existing components. v1 is a new reaction type; no schema break, no existing API surface affected. Configuration loaders treat the new YAML keys as optional in any deployment that does not declare an MQTT reaction.

## Supportability

### Telemetry

Metrics infrastructure (Prometheus, OpenTelemetry, etc.) is out of scope for v1; no Drasi reaction emits metrics today. v1 observability is logs only (see "Logging" subsection in Detail design). When metrics infrastructure lands at the framework level, the MQTT reaction will emit a standard set covering publish counts, publish failures by reason, reconnection counts, and per-publish latency histograms.

### Verification

**Integration tests** spin up a real Mosquitto broker via testcontainers (Eclipse Mosquitto image), matching the existing reaction test pattern (`components/reactions/storedproc-mssql/tests/mssql_helpers.rs` for the testcontainers idiom; `components/reactions/sse/tests/integration_tests.rs` for the mock-source-driven full-stack shape).

Required integration coverage:

- **Connect success** against Mosquitto with username/password.
- **Connect with auth rejection** transitions the reaction to `Error` (verifies CONNACK terminal-code classification).
- **Publish at QoS 0** end to end.
- **Publish at QoS 1** end to end with ACK round-trip.
- **Retain-flag round-trip**: publish with `retain: true`, new subscriber connects and reads it; then publish with `empty_payload: true` plus `retain: true` clears the retained message (new subscriber reads nothing).
- **Reconnect after broker restart**: stop the broker container, verify reaction stays `Running`; restart it, verify publishes resume.
- **TLS handshake** at least once (Mosquitto with self-signed cert; reaction with custom CA bundle).
- **Template rendering edge cases**: missing fields in `after`, slash inside templated value rejects with WARN and processing continues (verifies the reject-on-slash rule), oversize payload (rumqttc surfaces the error, reaction logs WARN and continues).

**Unit tests** in `src/tests.rs`:

- Topic validation (wildcards, null bytes, empty segments, length cap, reject-on-slash).
- Template resolution chain (per-query route -> default template -> drop with WARN).
- QoS deserialization rejects QoS 2.
- Config startup validation (URL scheme valid, TLS required when scheme is `mqtts`/`wss`, `client_auth: Some(...)` rejected with deferred-to-v2 error).
- `topic` non-empty check.

## Development plan

To be filled in by the implementer. Suggested phases:

1. Reaction trait skeleton plus config struct (URL parsing, TLS config, IdentityProvider wiring, validation).
2. EventLoop task lifecycle (spawn, panic guard, stop semantics).
3. Publish path: topic rendering plus validation (including reject-on-slash), payload templating, `{{json}}` helper, hardcoded v5 publish properties, QoS 0/1 routing.
4. CONNACK and Disconnect classification, status transitions, logging at all four levels.
5. Integration tests against Mosquitto via testcontainers; cover the matrix in "Verification."
6. Documentation: README plus YAML config examples for Mosquitto, EMQX, HiveMQ Cloud (username/password only).

## Open issues

To be populated by the implementer with any unresolved questions raised during prototyping.

## Appendices

(none)

## References

- The MQTT Source design document: https://github.com/drasi-project/design-documents/blob/main/drasi-lib/Mqtt-Source/mqtt-source-design.md
- The upcoming Shell Reaction design document.
- rumqttc: https://github.com/bytebeamio/rumqtt
- MQTT v5 specification: https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
