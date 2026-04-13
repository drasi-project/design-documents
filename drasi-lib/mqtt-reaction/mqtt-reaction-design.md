# MQTT Reaction Design

* Project Drasi - April 7, 2026 - Ahmed Kamal (@ahmed-kamal)

## Overview

The MQTT Reaction in Drasi publishes continuous query result changes to MQTT brokers so external systems can consume Drasi output using a lightweight pub/sub protocol. This enables integration with IoT systems, edge gateways, automation platforms, and event-driven services that already rely on MQTT topics.

This design defines a production-ready MQTT Reaction built on the Drasi Reaction SDK. It focuses on reliable publish behavior, predictable topic and payload mapping and secure authentication and transport.

## Terms and definitions

| Term | Definition |
|------|------------|
| MQTT | Message Queuing Telemetry Transport protocol used for lightweight pub/sub messaging |
| Broker | MQTT server endpoint that receives published messages and routes to subscribers |
| Topic | Hierarchical routing key used by MQTT for message distribution |
| QoS | MQTT Quality of Service level (0, 1, 2) controlling delivery guarantees |
| Retain | MQTT flag that keeps the last message on a topic for new subscribers |

## Objectives

### User scenarios

**IoT Engineer Publishing Query Results to IoT Consumers**

An IoT engineer deploys a Drasi continuous query and wants every result change to be published to MQTT so downstream IoT devices and services can react immediately. They need to:
- Configure broker connectivity and authentication
- Control topic naming by query and event type
- Choose delivery guarantees (QoS/retain/inflight messages..etc)

### Goals

1. Add an MQTT Reaction for drasi-core that subscribes to queries and emits result changes to MQTT brokers.
2. Configurable MQTTCallSpec (topic - QoS - retain) based on per query and operation.
3. Support MQTT v5 and v3.1.1 (fallback option).
4. Validate interoperability with target brokers (Mosquitto and HiveMQ).

### Non-Goals

## Design requirements

### Requirements

- Reaction implementation should align with patterns followed by other drasi-core reactions.
- Payload decoding must support JSON object payloads, basic scalar-in-object payloads or any other body format.
- Reaction implementation must support mqtt-reaction-wide configuration via reaction spec parameters.
- Reaction implementation must support QoS and retain configuration per query and operation.
- Must avoid breaking existing reaction contract semantics.
- Must support TLS/SSL transport and identity-based authentication.

### Dependencies

- drasi-core reaction implementation patterns.
- Broker availability and protocol support in test environments (Mosquitto, HiveMQ).

### Out of scope

- Retry mechanism for unknown failures.
- This first iteration doesn't include `DLQ` (Dead-letter topic used to store failed publish events after retry exhaustion).

## Design

### High-level design

The MQTT Reaction receives query result events from the Drasi Query and transforms each event into a publish request composed of:
- target MQTT topic
- payload
- MQTT publish options (QoS, retain)

The reaction maintains a broker connection with reconnect logic (relying on rumqttc `AsyncClient`). Events are  processed then published to the broker.

### Architecture Diagram

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

#### 1. Query results Configuration.

Per Query MQTT configuration `MqttQueryConfig` for each operation ( `ADD` - `Update` - `Delete` )

```rust
pub struct MqttQueryConfig {
    /// MQTT call specification for ADD operations (new rows in query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub added: Option<MqttCallSpec>,

    /// MQTT call specification for UPDATE operations (modified rows in query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub updated: Option<MqttCallSpec>,

    /// MQTT call specification for DELETE operations (removed rows from query results).
    #[serde(skip_serializing_if = "Option::is_none")]
    pub deleted: Option<MqttCallSpec>,
}
```

Introduction of new struct `MqttCallSpec` that includes the target topic name `topic`, the message template `template`, retaining method `retain` and the Quality of Service `qos`.


```rust
pub struct MqttCallSpec {
    /// MQTT topic.
    pub topic: String,

    /// Handlebars template.
    /// If empty, sends the raw JSON data.
    #[serde(default)]
    pub template: String,

    /// MQTT message retain policy.
    #[serde(default)]
    pub retain: RetainPolicy,

    /// QoS level for MQTT messages.
    #[serde(default)]
    pub qos: QualityOfService,
}
```

Example for using `MqttQueryConfig`

```rust
let default_template = MqttQueryConfig {
        added: Some(MqttCallSpec {
            topic: "t/added".to_string(),
            template: "[{{query_name}}] + {{after.symbol}}: ${{after.price}}".to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
        updated: Some(MqttCallSpec {
            topic: "t/updated".to_string(),
            template: "[{{query_name}}] ~ {{after.symbol}}: ${{before.price}} -> ${{after.price}}"
                .to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
        deleted: Some(MqttCallSpec {
            topic: "t/deleted".to_string(),
            template: "[{{query_name}}] - {{before.symbol}} removed".to_string(),
            retain: RetainPolicy::NoRetain,
            qos: drasi_reaction_mqtt::config::QualityOfService::ExactlyOnce,
        }),
    };
```

Query result change processing rules:

1. Determine operation type from the incoming result change event (`ADD`, `Update`, `Delete`).
2. Select the matching `MqttCallSpec` from `MqttQueryConfig`.
3. If the operation-specific spec is missing, fall-back to the default configuration.
4. Generate message body based on the configured `template` template.
5. Publish to `topic` with the configured `qos` and `retain` settings.

Template context available to use within message body templates:

- `query_name`: Continuous query name.
- `before`: Value image before change (for `updated` and `deleted` when available).
- `after`: Value image after change (for `added` and `updated` when available).

This approach keeps topic routing explicit and stable while still allowing per-query customization.

#### 2. MQTT v5 and v3.1.1

The reaction targets **MQTT v5** and **MQTT v3.1.1** (fallback option) as main protocols. 

The connection is managed by the [`rumqttc`](https://github.com/bytebeamio/rumqtt) crate, which implements both MQTT v3.1.1 and v5. A single long-lived `AsyncClient` (v5 or default "v3.1.1" ) is created at startup and reused for all publish calls. `rumqttc` handles reconnection.

The broker connection is configured through `MqttReactionConfig`:

```rust
/// MQTT reaction configuration
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct MqttReactionConfig {
    /// MQTT broker host (default: "localhost")
    #[serde(default = "default_broker_addr")]
    pub broker_addr: String,

    /// MQTT broker port (default: 1883)
    #[serde(default = "default_port")]
    pub port: u16,

    /// What transport mode to use when connecting to the MQTT broker (default: TCP)
    #[serde(default)]
    pub transport_mode: MqttTransportMode,

    /// Request timeout in seconds
    #[serde(default = "default_keep_alive")]
    pub keep_alive: u64,

    /// Clean or Persistent session for MQTT connection (default: true)
    #[serde(default = "default_clean_session")]
    pub clean_session: bool,

    /// Maximum incoming packet size (verifies remaining length of the packet)
    #[serde(default = "default_max_incoming_packet_size")]
    pub max_incoming_packet_size: usize,

    /// Maximum outgoing packet size (only verifies publish payload size)
    #[serde(default = "default_max_outgoing_packet_size")]
    pub max_outgoing_packet_size: usize,

    /// Max internal request batching
    #[serde(default = "default_max_request_batch")]
    pub max_request_batch: usize,

    /// Request (publish, subscribe) channel capacity
    #[serde(default = "default_request_channel_capacity")]
    pub request_channel_capacity: usize,

    /// Capacity of the async channel
    #[serde(default = "default_event_channel_capacity")]
    pub event_channel_capacity: usize,

    /// Minimum detlay time between consecutive outgoing packets
    /// While retransmitting pending packets.
    #[serde(default = "default_pending_throttle")]
    pub pending_throttle: u64,

    /// Connection timeout
    #[serde(default = "default_connection_timeout")]
    pub connection_timeout: u64,

    /// Maximum number of outgoing inflight messages
    #[serde(default = "default_inflight")]
    pub max_inflight: u16,

    /// Base topic for MQTT requests
    #[serde(default = "default_topic")]
    pub default_topic: String,

    /// Query-specific call configurations
    #[serde(default)]
    pub query_configs: HashMap<String, MqttQueryConfig>,

    /// Last will that will be issued on unexpected disconnect
    pub last_will: Option<MqttLastWill>,

    /// Adaptive batching configuration (flattened into parent config)
    #[serde(flatten)]
    pub adaptive: Option<AdaptiveBatchConfig>,

    /// Identity Provider
    pub identity_provider: Arc<dyn IdentityProvider>,
}
```

#### 3. Security

Security is split into two independent axes: **transport security** (how bytes travel) and **identity / authentication** (who is connecting). They can be combined freely.

##### Transport — `MqttTransportMode`

The `transport_mode` field in `MqttReactionConfig` controls the socket layer:

```rust
pub enum MqttTransportMode {
    /// Plain TCP — no encryption (default)
    #[default]
    TCP,
    /// TLS-encrypted transport
    TLS {
        /// CA certificate used to verify the broker's identity
        ca: Vec<u8>,
        /// Optional ALPN protocol list
        alpn: Option<Vec<Vec<u8>>>,
        /// Optional mTLS client certificate and private key.
        client_auth: Option<(Vec<u8>, Vec<u8>)>,
    },
}
```

The `ca` field is required for TLS so the reaction always verifies the broker's certificate. Skipping broker verification is not supported to prevent man-in-the-middle attacks. `client_auth` enables mutual TLS (mTLS), where the reaction presents its own certificate to the broker.

##### Authentication — `IdentityProvider`

MQTT application-layer authentication (CONNECT username / password) is handled through a pluggable `IdentityProvider` trait rather than a static config field. This allows to deal with credentials from different sources (AWS - Azure - Simple Username-Password) without changing the core publish logic:

```rust
/// Trait for identity providers that supply authentication credentials.
#[async_trait]
pub trait IdentityProvider: Send + Sync {
    /// Fetch credentials for authentication.
    /// Called once during connection setup 
    async fn get_credentials(&self) -> Result<Credentials>;

    /// Clone the provider into a boxed trait object.
    fn clone_box(&self) -> Box<dyn IdentityProvider>;
}
```

`get_credentials` is called on new connection attempt.


#### 4. Adaptive Batching

The MQTT Reaction can operate in two modes: **basic** (publish immediately) or **adaptive** (batch and publish intelligently). 

**Important:** MQTT does not support protocol-level message batching. The adaptive batching strategy implemented here is **application-level only**. Data results are accumulated in the reaction's memory, then each result is published individually with its own MQTT publish message. This differs from transport-level batching and means each result still generates a separate MQTT `PUBLISH` frame, but the reaction can control the timing of these publishes.

Adaptive batching is configured by the `AdaptiveBatchConfig` struct in `MqttReactionConfig.adaptive`:

```rust
pub struct AdaptiveBatchConfig {
    /// Minimum batch size (events per batch) used during idle/low traffic
    pub adaptive_min_batch_size: usize,
    
    /// Maximum batch size (events per batch) used during burst traffic
    pub adaptive_max_batch_size: usize,
    
    /// Window size for throughput monitoring
    pub adaptive_window_size: usize,
    
    /// Maximum time to hold a batch (milliseconds)
    pub adaptive_batch_timeout_ms: u64,
}
```

##### How Adaptive Batching Works

When `adaptive` is configured:

1. **Accumulation phase**: Incoming query results are held in memory and accumulated into a batch. Results from the same query are grouped together.
2. **Intelligent release**: The batcher measures the incoming event rate and decides when to release the batch based on size and timeout thresholds.
3. **Processing on release**: Once a batch is ready (min size reached, max size hit, or timeout expires), all results in the batch are processed. For each result, a template is rendered and an individual MQTT `PUBLISH` message is sent. Each result gets its own publish call.
4. **Throughput**: While each result still generates a separate MQTT message (batching cannot combine multiple results into a single MQTT message due to protocol design), the application-level batching allows the reaction to control the rate and time of publishes (which could result in efficiency within the TCP and OS level).

##### Basic vs. Adaptive Processing

Both modes handle the same publish logic (template rendering, QoS, retain). The only difference is when publishes are triggered.

#### 5. Reaction Builder API

The `MqttReactionBuilder` (this is an initial schema, expected to be changed during development) provides an API for constructing MQTT reactions.

##### Builder Pattern
max_request_batch
```rust
pub struct MqttReactionBuilder {
    id: String,
    queries: Vec<String>,
    broker_addr: String,
    port: u16,
    transport_mode: MqttTransportMode,
    keep_alive: u64,
    clean_session: bool,
    max_ingoing_packet_size: usize,
    max_outgoing_packet_size: usize,
    max_request_batch: usize,
    request_channel_capacity: usize,
    event_channel_capacity: usize,
    pending_throttle: u64,
    connection_timeout: u64,
    max_inflight: u16,
    default_topic: String,
    query_configs: HashMap<String, MqttQueryConfig>,
    last_will: Option<MqttLastWill>,
    adaptive_config: Option<AdaptiveBatchConfig>,
    identity_provider: Arc<dyn IdentityProvider>,
}
```

##### Usage Examples

**Example 1**
```rust
let reaction = MqttReactionBuilder::new("my-mqtt")
    .with_query("user_events")
    .with_broker_addr("mqtt.example.com")
    .with_default_topic("drasi/events")
    .build()?;
```

**Example 2**
```rust
let reaction = MqttReactionBuilder::new("mqtt-secure")
    .with_queries(vec!["orders".to_string(), "inventory".to_string()])
    .with_broker_addr("mqtt.example.com")
    .with_port(8883)
    .with_transport_mode(MqttTransportMode::TLS { 
        ca: load_ca_cert()?,
        alpn: None,
        client_auth: Some((load_cert()?, load_key()?)),
    })
    .with_keep_alive(60)
    .with_default_topic("drasi/production")
    .build()?;
```

**Example 3**
```rust
let mut query_configs = HashMap::new();

let reaction = MqttReactionBuilder::new("mqtt-events")
    .with_broker_addr("mqtt.local")
    .with_adaptive_config(AdaptiveBatchConfig {
        adaptive_max_batch_size: 100,
        ..Default::default()
    })
    .build()?;
```

##### Sample Key Methods

**Connection Settings:**
- `with_broker_addr(addr)` — MQTT broker hostname or IP
- `with_port(port)` — Broker port (default: 1883)
- `with_transport_mode(mode)` — TCP or TLS
- `with_keep_alive(secs)` — MQTT keep-alive heartbeat
- `with_connection_timeout(ms)` — Connection establishment timeout

**Queries & Topics:**
- `with_query(name)` — Subscribe to one query
- `with_queries(vec)` — Subscribe to multiple queries
- `with_default_topic(name)` — Fallback topic when no per-query config exists
- `with_query_config(name, spec)` — Set routing for one query
- `with_query_configs(map)` — Set routing for multiple queries

**Performance & Buffering:**
- `with_request_channel_capacity(n)` — Internal MQTT request queue size
- `with_event_channel_capacity(n)` — Incoming result queue size
- `with_max_inflight(n)` — MQTT inflight window (QoS 1/2)
- `with_pending_throttle(ms)` — Backpressure on retransmissions

**Adaptive Batching:**
- `with_adaptive_config(config)` — Full adaptive batch settings
- `with_min_adaptive_batch_size(n)` — Minimum results per batch
- `with_max_adaptive_batch_size(n)` — Maximum results per batch
- `with_adaptive_window_size(n)` — Rate measurement window
- `with_adaptive_batch_timeout_ms(ms)` — Maximum batch hold time

**Finalization:**
- `build()` — Construct and return the `MqttReaction`


