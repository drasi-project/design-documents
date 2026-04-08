# Title

* Project Drasi - 2026-3-6 - Ahmed Kamal (@ahmed-kamal2004)

## Overview

MQTT is a widely used lightweight messaging protocol for IoT, industrial telemetry, and event-driven systems. This design adds a new Drasi Source that subscribes to one or more MQTT topics and converts incoming messages into `SourceChange` events that Drasi continuous queries can process in near real time. With this source, teams can connect devices and event publishers that already use MQTT brokers (for example, Mosquitto, EMQX, or HiveMQ) without building custom ingestion bridges.

The MQTT Source will support configurable topic subscriptions, secure broker authentication, payload decoding (JSON first), and topic-to-graph mapping so users can write expressive queries over physical hierarchies such as building/floor/room/sensor. The implementation is expected to follow the existing Drasi source architecture and patterns so that it remains maintainable, testable, and consistent with other sources in drasi-core.


## Terms and definitions

<!-- These terms are internal to your design and will not be shared publicly. -->

| Term | Definition |
|------|------------|
| MQTT | Message Queuing Telemetry Transport, a lightweight publish/subscribe protocol |
| Topic hierarchy | Multi-level MQTT topic path used to organize and route messages (e.g., `building-1/floor-1/room-2/sensor-1`) |
| SourceChange | Internal Drasi change envelope emitted by sources into the continuous query engine |

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->

### User scenarios

**IoT Platform Engineer**

An engineer operates a fleet of sensors publishing telemetry to MQTT topics and wants Drasi to detect patterns in near real time (for example, persistent over-temperature conditions). They need to configure one MQTT Source, subscribe to topic patterns, map topic segments into queryable entities, and run continuous queries without writing custom adapters.

**Data Analyst / Operations User**

An analyst wants to query sensor hierarchy and values (Building → Floor → Room → Sensor) and create reactive logic. They need stable graph labels and properties that reflect MQTT topics and payloads so query authoring is predictable.

### Goals

1. Add an MQTT Source for drasi-core that subscribes to broker topics and emits Drasi `SourceChange` records.
2. Support topic hierarchy mapping to graph elements so users can write meaningful Cypher/GQL queries.
3. Support JSON payloads as the primary format with clear extension points for future formats.
4. Support MQTT v3.1.1 as the primary protocol version, this achieves (maximum compatibility) as it understands MQTT v3.1.1 brokers and interoperable with MQTT v5 brokers.
5. Validate interoperability with target brokers (Mosquitto and HiveMQ).

### Non-Goals

<!-- Describe non-goals to identity something that we won't be focusing on immediately. We won't be expending any effort on these matters. -->

## Design requirements

<!-- Describe key technical requirements, dependencies, and constraints that impact the design.

Provide a link to the issue(s) tracking this work. -->

### Requirements

- Source implementation should align with patterns followed by other drasi-core sources.
- Topic subscription must support exact topics and wildcard forms (`+`, `#`) where broker permits.
- Topic-to-schema mapping must support hierarchical querying use cases.
- Payload decoding must support JSON object payloads and basic scalar-in-object payloads.
- Configuration should allow selecting mapping strategy and payload format.

### Dependencies

- drasi-core source implementation patterns.
- Continuous query engine expectations for node/relationship labels and stable IDs.
- Broker availability and protocol support in test environments (Mosquitto, HiveMQ).

### Out of scope

The first iteration does not include broad non-JSON codec support (for example Protobuf), or advanced broker-specific optimizations. Those can be added in follow-up iterations once production usage patterns are validated.

## Design

### High-level design

The MQTT Source connects to one configured broker, subscribes to configured topic patterns, decodes each incoming payload, then maps topic segments and payload fields into graph-oriented `SourceChange` events. These events are processed by optional middleware (JQ) before being forwarded to Drasi’s query engine.

The key design axis is topic mapping strategy. This document captures three approaches:

1. **Hierarchy model mapping (preferred for readability):** user declares semantic levels such as `Building/Floor/Room/Sensor`.
2. **Separator/regex mapping:** user declares parsing pattern (e.g., dash-separated entity/value tokens).
3. **Level mapping fallback:** no semantic config; levels become generic labels (`L0`, `L1`, `L2`).
4. **Pattern based matching:** uses Template Variables to extract metadata from the topic and determines how the JSON payload updates the graph.


### Architecture Diagram

```text
Sensors/Publishers -> MQTT Broker -> MQTT Source -> Mapping/Middleware -> Drasi Query Container -> Reactions
```

### Detail design

#### Input data examples

Example payloads received from publishers:

```json
{
	"timestamp": "2026-03-05T10:15:30Z",
	"type": "temperature",
	"value": 4.2,
	"unit": "C"
}
```

or

```json
{
	"temperature": "4.2"
}
```

or 

```
44.2
```

Example topic:

```text
building-1/floor-1/room-2/sensor-1
```
or

```
building-1/floor-1/room-2/sensor-1/temperature
```

#### Quality of service (QoS)

Quality of service can be specified per topic subscription `client.subscribe(topic, qos)`, as a default `QoS` can be set to `1` for all subscriptions, while keeping `QoS::0` and `QoS::2` as options.

With durable WAL being supported and enabled within MQTT source, `QoS::0` must be prevented and fallen back to `QoS::1`.

#### Topic hierarchy guidance

The source should support structured topic paths that align with industrial hierarchy patterns such as ISA-95 and Unified Namespace style trees. For example:

```text
Enterprise/Site/Area/ProductionLine/WorkCell/Equipment/DataPoint
```

or:

```yaml
manufacturing/
  plantA/
    sensors/
      humidity/
        sensor001
        sensor002
    inventory/
      raw_materials/
        current_stock
      finished_goods/
        current_stock
    quality_control/
      inspection/
        results
```

This allows filtered subscriptions such as:

```text
manufacturing/plantB/quality_control/testing/#
```

#### Query examples

Simple retrieval query:

```cypher
MATCH (b:Building {id: "building-1"})
			-[:HAS_FLOOR]->(f:Floor {id: "floor-2"})
			-[:HAS_SENSOR]->(s:Sensor {id: "sensor-7"})
RETURN b.id AS building, f.id AS floor, s.id AS sensor, s.value
```

Complex condition query:

```cypher
MATCH (b:Building)-[:HAS_FLOOR]->(f:Floor)-[:HAS_ROOM]->(r:Room)-[:HAS_SENSOR]->(s:Sensor)
WHERE s.type = 'temperature'

WITH
	b, f, r, s,
	drasi.changeDateTime(s) AS temperatureChangeTime

WHERE
	temperatureChangeTime != datetime({epochMillis: 0}) AND
	drasi.trueFor(
		s.value > 25,
		duration({ seconds: 15 })
	)

RETURN
	b.id AS buildingId,
	f.id AS floorId,
	r.id AS roomId,
	s.id AS sensorId,
	s.value AS temperature,
	temperatureChangeTime AS fireRiskDetectedSince
```

#### Topic mapping options

##### Option A: User-declared hierarchy model

User config:

```text
Building/Floor/Sensor
```

Topic:

```text
campus/the-first-floor/temperature
```

Mapped labels:

```text
campus -> Building
the-first-floor -> Floor
temperature -> Sensor
```

Pros: highly readable queries and explicit semantics.
Cons: requires mapping model configuration and enforcement.

##### Option B: Separator/regex mapping

User config provides naming pattern (for example `-`).

Topic:

```text
building-1/floor-3/sensor-2
```

Mapped labels:

```text
building-1 -> building
floor-3 -> floor
sensor-2 -> sensor
```

Pros: flexible and compact config.
Cons: naming convention must be strictly followed.

##### Option C: Level mapping (no semantic config)

Topic:

```text
building-1/floor-3/sensor-2
```

Mapped labels:

```text
building-1 -> L0
floor-3 -> L1
sensor-2 -> L2
```

Pros: zero config.
Cons: lower query readability and weaker domain semantics.

##### Option D: Pattern Based Matching
This provides a flexible Domain Specific Language (DSL) that uses Template Variables to extract metadata from the topic and determines how the JSON payload updates the graph.


it will have 4 sections
- `pattern`: Users define topic structures using {variable} placeholders (e.g., {building}/{floor}/sensor/{id}). The source extracts these segments to use as node IDs, labels, or properties.

1. `entity`: contains the `label` and the `id` patterns for the main node that contains data.
2. `properties`: contains ingestion modes and contains 3 options.

`mode` of `payload_as_field` and specified `field_name`: Use when the topic segment (e.g., {attribute}) defines the property name and the payload is a single value.

`mode` of `payload_spread`: Use when the payload is a JSON object; the source flattens all keys directly into the Drasi node properties.

`inject`: (can be used with any of the `payload_as_field` or `payload_spread` options) Allows the user to manually insert topic variables (like {floor}) into the node's properties alongside the payload data for better query context.

3. `nodes`: to define the hierarchical parent nodes used in relations, where every nodes requires a `label` and `id`.
4. `relations`: to define the relation between the hierarchical parent nodes defined in `nodes` and the main node defined in `entity`, where each relation needs `label`, `from` and `to`,

Note, nodes specified in `nodes` section but can't access the main entity node using any path can be ignored (whenever ignoring it increases the efficieny).

Example

```yaml
topic_mappings:
- pattern: "{building}/{floor}/{room}/thermostat/{attribute}"
	entity:
		label: "Thermostat"                 
		id: "{building}:{floor}:{room}:thermostat"    
	properties:
		mode: payload_as_field.      
		field_name: "{attribute}"                   
	nodes:                                                      
		- label: "Building"
		  id: "{building}"
		- label: "Floor"
		  id: "{building}:{floor}"
		- label: "Room"
		  id: "{building}:{floor}:{room}" 
	relationships:
        - label: "HAS_FLOOR"                                      
          from: "Building" ## node label
          to: "Floor"          ## node label
        - label: "HAS_ROOM"
		  from: "Floor"
          to: "Room"        
		- label: "HAS_THREMOSTAT"
		  from: "Room"
		  to: "Thermostat"
```

#### Behavior

Proposed sensor input shape:
```json
{
	"type": "temperature",
	"value": 4.2,
	"unit": "C"
}
```
With topic name:
```
building-1/floor-1/room-2/sensor-1
```
With configured hierarchical model:
```
Building/Floor/Room/Sensor
```

Proposed Drasi `SourceChange` records emitted after source transformation:

```rust
// Topic segments are mapped positionally onto the configured hierarchy model:
// building-1 -> Building, floor-1 -> Floor, room-2 -> Room, sensor-1 -> Sensor
query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "building-1"),
			labels: Arc::new([Arc::from("Building")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "building-1",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "floor-1"),
			labels: Arc::new([Arc::from("Floor")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "floor-1",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Node {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "room-2"),
			labels: Arc::new([Arc::from("Room")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::from(json!({
			"name": "room-2",
		}))
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "building-1-has-floor-1"),
			labels: Arc::new([Arc::from("HAS_FLOOR")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "building-1"),
		in_node: ElementReference::new("mqtt", "floor-1"),
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "floor-1-has-room-2"),
			labels: Arc::new([Arc::from("HAS_ROOM")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "floor-1"),
		in_node: ElementReference::new("mqtt", "room-2"),
	},
}).await;

query.process_source_change(SourceChange::Update {
	element: Element::Relation {
		metadata: ElementMetadata {
			reference: ElementReference::new("mqtt", "room-2-has-sensor-1"),
			labels: Arc::new([Arc::from("HAS_SENSOR")]),
			effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
		},
		properties: ElementPropertyMap::new(),
		out_node: ElementReference::new("mqtt", "room-2"),
		in_node: ElementReference::new("mqtt", "sensor-1"),
	},
}).await;
```

Main SourceChange:

```rust
query.process_source_change(SourceChange::Update {
    element: Element::Node {
        metadata: ElementMetadata {
            reference: ElementReference::new("mqtt", "sensor-1"),
            labels: Arc::new([Arc::from("Sensor")]),
            effective_from: chrono::Utc::now().timestamp_nanos_opt().unwrap() as u64,
        },
        properties: ElementPropertyMap::from(json!({
            "type": "temperature",
            "value": 4.2,
            "unit": "C",
            "topic": "building-1/floor-1/room-2/sensor-1"
        }))
    },
}).await;
```

#### Optimization

For the hierarchy model, the source must emit the hierarchical nodes and relations to create a queryable graph structure. However, emitting redundant (already stored in indexes) nodes and relations for every incoming message is inefficient. The MQTT Source can choose to implement filter (Write-absorption layer) from a set of strategies including **Bloom Filters** and **Adaptive Radix Trees (ART)** to minimize redundant emissions while maintaining correctness.

##### Problem

High-load MQTT sources emit many messages from the same sensor hierarchies. If each message triggers full node and relation emission (e.g., `Building-1`, `Floor-1`, `Room-2`, `Sensor-1` nodes plus 3 relations), repeated hierarchies sent as `SourceChange` add significant load on continuous queries and indexes, especially in deployments with thousands of sensors.

##### Solution 1: Hierarchical-based Bloom Filter approach

Maintain a **single shared Bloom Filter** and check hierarchically by progressively building the full topic path. For each incoming message, validate that all intermediate hierarchy levels have been seen before emitting nodes and relations.

**How it works:**

For topic `building-011/floor-02/room-03/sensor-01`:

1. Check Bloom Filter for `building-011` → exists?
2. Check Bloom Filter for `building-011/floor-02` → exists?
3. Check Bloom Filter for `building-011/floor-02/room-03` → exists?
4. Check Bloom Filter for `building-011/floor-02/room-03/sensor-01` → exists?

If all checks pass, the hierarchy is treated as likely known and schema emission is skipped.
The false-positive chance for a full path becomes much lower because every prefix must pass. if per-check false-positive rate is `p` and topic depth is `n`, the full-path false-positive rate is approximately `p^n`.

One edge case is a persistent false-positive key pattern. To avoid starving those paths forever, we add a very low-probability revalidation path that occasionally processes and inserts even when all checks return positive.



**Example:**

```
First message: building-011/floor-02/room-03/sensor-01
- Check "building-011": NOT found → Emit the full schema + add to filter

Second message: building-011/floor-02/room-03/sensor-01 (repeated)
- Check "building-011": FOUND
- Check "building-011/floor-02": FOUND
- Check "building-011/floor-02/room-03": FOUND
- Check "building-011/floor-02/room-03/sensor-01": FOUND
→ Skip all node/relation emissions, emit only sensor value update (emit schema with very low probability)

Third message: building-011/floor-02/room-04/sensor-02 (new room, same floor)
- Check "building-011": FOUND
- Check "building-011/floor-02": FOUND
- Check "building-011/floor-02/room-04": NOT found → Emit full schema + add to filter

```

Implementation sketch:

```python
s = "building01/floor02/room03/sensor-temperature"

# segment s
s_segments = s.split("/")

# check for each hierarchy parent if it exists in the bloom filter
hierarchy = ""
for i, segment in enumerate(s_segments):
	hierarchy = segment if i == 0 else hierarchy + "/" + segment

	if hierarchy not in bloom_filter:
		emit_to_continuos_query(s)
		return

# exiting the loop without return means hierarchy likely exists in bloom filter
# with very low probability of false positives (multi-level checks)
# do a final low-probability insertion to handle persistent false-positive inputs
small_probability = 0.0001  # 0.01%
if get_random_probability() < small_probability:
	emit_to_continuos_query(s)

return
```
when using `Pattern based matching`, checking is done againest the parents of the main entity node and the relations between them `for each edge` instead the traditional segmenting based on the topic naming pattern.

**Advantages:**
- **Lower false positives rate**: All prefixes must pass before skipping schema emission.
	- Example (independence approximation): with `p=5%` and 4 levels, `p^n = 0.05^4 = 0.00000625` (0.000625%).
- **Minimal memory overhead**: One shared Bloom Filter across all hierarchies, and the bloom filter is very memory efficient by design.
- **Strong write-absorption behavior**: Repeated hierarchies avoid redundant node/relation emissions.

**Disadvantages:**
- **Multiple checks per message**: O(depth) lookups instead of a single O(1) check, where `depth` is the number of segments in the topic name.
- **No strong (almost eventually) hierarchical consistency**: Because of false-positives probability, some hierarchies maybe classified as already exists in the database while they aren't, this problem is mitigated using the `low probability insertion mechanism` under the condition that there is a load from this hierarchy (topic name).
- **Scalability** standard bloom-filter is not scalable with increasing number of inputs, which results in increasing the rate of false positive, solution is to implement scalable size-increasing bloom filter that keeps a strict limit on the false positive rate (multiple papers can be analyzed for implementation).



##### Solution 2: ART (Adaptive Radix Tree) approach

Maintain an **Adaptive Radix Tree** and check the hierarchy existence by checking the topic name against the tree.

**How it works:**

For topic `building-011/floor-02/room-03/sensor-01`:

1. Check Tree for `building-011/floor-02/room-03/sensor-01` → exists?

If exists, schema emission is skipped, else, the schema is emitted to the query and added to the tree.

when using `Pattern based matching`, no change is expected in the checking mechanism.

**Advantages:**
- **Minimal memory overhead**: ART is highly memory efficient (but less efficient than bloom-filters).
- **Consistency**: If a specific hierarchy is identified as existed, then it is 100 % exists in the indexes.
- **O(K) check**: Where K is the number of bytes of the topic string, with path compaction mechanism of the ART, K is decreased.

**Disadvantages:**
- **Performs poorly with undefined topic naming patterns**: The strength of radix trees is path compaction for prefixes, if there is no prefix-shared between inputs, it will memory inefficient.
- **Unneeded space consumption**
	assuming we got `building-001/floor-02/room-03/sensor-01` and `building-011/floor-02/room-03/sensor-01`, the term `1/floor-02/room-03/sensor-01` will be replicated on each side of the tree.
<img width="789" height="456" alt="Screenshot from 2026-04-06 17-00-37" src="https://github.com/user-attachments/assets/d398843c-a0c8-484f-9dce-2e20b6b4bcf2" />


##### Solution 3: Segmented ART approach

Maintain one ART, HashSet(u64) for relations and atomic counter (u32).
we insert in the ART `(key, value)` where `key` is `{label}-{id}` and `value` is the current seq number (u32).
the hash-set contains (`u64`)(`parent u32``child u32`)

<img width="1207" height="551" alt="Screenshot from 2026-04-06 18-24-29" src="https://github.com/user-attachments/assets/e9e71b73-778c-49f3-ae5c-c8daeeb064a7" />


**How it works:**

For topic `building-011/floor-02/room-03/sensor-01`:


1. key: `BUILDING-building-011` value: 1(u32) (ART)
2. key: `FLOOR-floor-02` value: 2(u32) (ART)
3. key: `ROOM-room-03` value: 3(u32) (ART)
4. key: `SENSOR-sensor-01` value: 4(u32) (ART)
5. relation (1-2) (HashSet)
6. relation (2-3) (HashSet)
7. relation (3-4) (HashSet)

this gives detailed picture of what exists in the indexes, enabling fine-controlled of the emitted hierarchy schema payload (so, we don't insert what is already existing). 

when using `Pattern based matching`, we will store for each node (ids) and relation (specified in the `nodes` and `relations` sections) instead of for each segment (basically mapping is done before the segmented-ART step).

**Advantages:**
- **Deterministic existence checks**: Not probabilistic.
- **Can be more efficient with the undefined topic naming patterns**
- **Fine-Grained control**: Fine-grained control and representation for the schema.
- **Minimize unneeded replication**: because of namespace isolation.

**Disadvantages:**
- **Higher metadata overhead**: Each token maintains its counters, and each relation is represented by >= 8 bytes in memory.
- **Less time efficient**: checking relation existence can be between O(1) on average, but O(n) for worst cases, for checking set of relations, this can be inefficient.

##### Persistence vs in-memory
- the filter could start `cold`, and be `hot` while working.
- disk-persistence and synchronization can be done in next versions.

##### Configuration 

`hierarchy` key can be used to define the chosen mechanism, with values:

- `bloom` for using the first option of bloom-filter.
- `art` (default to keep performance efficient, while preserving consistency) for using the second option of standard-simple ART.
- `segmented-art` for using the third option of segmented-ART.
- `none` for none of those options, directly emitting hierarchy to the continuos query.

#### Bootstrapping

MQTT retained messages are treated by default as a **native bootstrap mechanism** for this source. On startup or re-subscription, retained messages provide an initial snapshot of current topic state without requiring an external bootstrap flow.

The source should also support Drasi's **pluggable bootstrap providers**. This keeps behavior consistent with other sources and allows operators to choose non-MQTT bootstrap strategies when needed (postgres - mssql).

No MQTT-specific `BootstrapProvider` implementation is required. The MQTT source consumes retained messages natively and uses the existing Drasi bootstrap-provider extension model when an bootstrap source is configured.

#### Data format

JSON is the primary payload format in v1. Additional formats may be added later based on explicit configuration.

#### MQTT protocol version

v3.1.1 is the preferred implementation target because it understands v3.1.1 brokers, while being interoperable with v5 brokers

#### Target brokers

- Mosquitto
- HiveMQ

<!-- This section should be detailed and through enough that another developer could implement your design and provide enough detail to get a high confidence estimate of the cost to implement the feature but isn't as detailed as the code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and describe it in enough detail that someone else could implement it. Cover ALL of the important decisions like names. Your goal is to get agreement to proceed with coding and PRs.

If there are alternative you are considering please include that in the open questions section.

If the product has a layered architecture, it's good to align these sections with the product's layers. This will help readers use their current understanding to understand your ideas.

- **Advantages of this design** - Describe what's good about this plan relative to other options. Does it feel easy to implement? Provide flexibility for future work?
- **Disadvantages** - Describe what's not ideal about this plan. If you don't point these things out other people will do it for you. This is a good place to cover risks. -->

### API Design

No new REST API is proposed in this design. The expected change is a new source type configuration surface (broker endpoint/auth, subscriptions, mapping strategy, payload format) through existing Drasi source resource flows and CLI/apply workflows.

### Alternatives Considered

- **Hierarchy model mapping** vs **separator/regex mapping** vs **generic level mapping** vs **Pattern-based matching**.
- Hierarchy model is favored for readability of user queries.
- Pattern-Based matching is favored because the extreme flexibility it gives to the users.
- Separator/regex and level mapping remain valid fallback modes for low-governance topic ecosystems.

## Security

- Use secure broker connectivity (TLS/SSL), (TCP).

```rust
pub enum MqttTransportMode {
    #[default]
    TCP,
    TLS {
        /// ca certificate
        ca: Vec<u8>,
        /// alpn settings
        alpn: Option<Vec<Vec<u8>>>,
        /// tls client_authentication
        client_auth: Option<(Vec<u8>, Vec<u8>)>, // mTLS 
    },
}
```
- Add support for identity provider authentication trait `IdentityProvider`, this unlocks the usage of `AwsIdentityProvider`, `AzureIdentityProvider`, and `PasswordIdentityProvider` (traditional username and password authentication).

- Validate and safely parse payloads to prevent malformed-input failures.

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc. -->

## Compatibility impact

- Compatible with MQTT v3.1.1 brokers and interoperable with MQTT v5 brokers.
- No breaking changes expected for existing sources; this is an additive source type.
- Query compatibility depends on selected mapping strategy (semantic labels vs generic level labels).

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here. -->

## Supportability

### Telemetry
<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature. -->

### Verification

- Unit tests for drasi source.
- Integration tests against Mosquitto and HiveMQ (needs more research).
- End-to-end test scenarios validating query outcomes for hierarchy traversal and temporal conditions (needs more research).

<!-- This includes the test plan to validate the features such as unit-test and functional tests. -->

## Development Plan


<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates. -->

## Open issues

1. Should hierarchy model mapping be mandatory in v1, or optional with auto-fallback?
2. Which additional payload formats should be prioritized after JSON?
3. Is the `durability.max_size_mb` in the Durable WAL design source configurable after source start or needs source restart to be reconfigured ?
<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design. -->

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

## Note on Drasi Redesign & Reliability:
This implementation acknowledges the upcoming architectural changes to the Drasi lib. For V1, the MQTT Source operates on an at-most-once delivery guarantee.

- Current Assumption: A local Write-Ahead Log (WAL) is not implemented in this iteration. This simplifies the initial scope and focuses on protocol interoperability and mapping logic.

- Future Integration: Once the centralized durability framework is available in drasi-lib, the MQTT Source will be integrated to support persistent state and "at-least-once" guarantees across restarts.

## References

- https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/
- https://drasi.io/drasi-kubernetes/tutorials/curbside-pickup/
- https://www.machbase.com/en/post/how-to-make-sensor-data-send-directly-to-a-database-via-mqtt
- https://drasi.io/reference/query-language/
- https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901118
- https://www.hivemq.com/blog/mqtt5-essentials-part6-user-properties/
- https://www.isa.org/standards-and-publications/isa-standards/isa-95-standard

<!-- Optional. Add the design documents and references you use for this document. -->
