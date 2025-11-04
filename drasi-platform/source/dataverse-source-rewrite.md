# Dataverse Source Rewrite

* Project Drasi - October 30, 2025 - Ruokun Niu (@ruokun-niu)

## Overview

<!-- Provide a succinct high-level description of the component or feature and where/how it fits in the big picture. The overview should be one to three paragraphs long and should be understandable by someone outside Drasi team. 

For this Dataverse source rewrite, consider explaining:
- What is the Dataverse source and what does it connect to?
- Why does it need to be rewritten/updated?
- What are the key improvements or changes this rewrite will bring?
-->

The Dataverse source in Drasi is responsible for connecting to Microsoft Dataverse, a cloud-based data platform used by Power Platform applications. This source enables Drasi to monitor changes in Dataverse tables and react to those changes in real-time through continuous queries.

Our current implementation of the Dataverse source requires a comprehensive rewrite to enhance its functionality and improve its performance. It is not implemented using the Drasi Source SDK, which limits its maintainability and extensibility. The rewrite aims to leverage the Source SDK to provide a more robust and flexible implementation.

Key improvements in this rewrite include:
- Optimized change fetch intervals to balance latency and API rate limits
- Standardized handling of Dataverse entity IDs
- Support for dual authentication methods: client secret and Entra Workload ID (managed identity)
- Enhanced handling of lookup fields and multi-table relationships
- Clear documentation of required permissions and roles for secure operation

## Terms and definitions

<!-- These terms are internal to your design and will not be shared publicly. -->

| Term | Definition |
|------|------------|
| Dataverse | Microsoft Dataverse (formerly Common Data Service) - a cloud-based data platform used by Power Platform |
| Entity | A table in Dataverse that stores data records (e.g., Account, Contact, custom entities) |

## Objectives

<!-- Describes goals/non-goals and user-scenario of this feature to understand the end-user goals.

- If the feature shares the same objectives of the existing design, link the existing doc rather than repeat the same context.
- If the feature has a scenario, UX, or other product feature design doc, link it here and summarize the important parts. -->


### User scenarios

**Data Engineer Setting Up Change Tracking**

A data engineer wants to monitor changes in Dataverse tables (such as Account, Contact, or custom entities) and react to those changes in real-time using Drasi continuous queries. They need to:
- Configure the Dataverse source to connect to their Dataverse environment
- Authenticate using either managed identity or client secret
- Specify which tables to monitor
- Create continuous queries that process the change data
- Monitor the health and status of the data pipeline

### Goals

The primary goals of this Dataverse source rewrite are:

1. **Optimize Change Fetch Interval and Rate Limiting**
   - Investigate the current interval at which the reactivator fetches changes from Dataverse
   - Research and document Dataverse API rate limits for change tracking requests
   - Determine the optimal polling frequency that balances:
     - Change detection latency (how quickly changes are detected)
     - API throttling avoidance (staying within Dataverse service limits)
     - Resource efficiency (minimizing unnecessary API calls)
   - **Implement intelligent polling strategy**: Ideally, fetch changes immediately when a change is detected, and only wait for the configured interval when no changes are present (adaptive polling)
   - Implement configurable polling intervals with sensible defaults based on findings

2. **Investigate and Standardize Dataverse ID Handling**
   - Explore how Dataverse entity IDs work and their formats:
   - Understand the relationship between:
     - Primary key field names (e.g., `accountid`, `contactid`, or custom entity primary keys)
     - The standardized ID field that appears in API responses (data.id)
   - Document how IDs should be consistently extracted and used in the Drasi source SDK.

3. **Implement and Validate Dual Authentication Methods**
   - Ensure both authentication methods are fully supported and tested:
     - **Client Secret authentication**: Application (client) ID + client secret
     - **Entra Workload ID (Managed Identity)**: Azure workload identity federation for Kubernetes
   - Document authentication setup requirements:
     - Required Dataverse permissions (e.g., read access to entities, change tracking)
     - Azure AD app registration steps for client secret method
     - Workload identity configuration for managed identity method
     - Troubleshooting guide for common authentication issues
   - Ensure secure credential handling throughout (no secrets in logs, proper rotation support)

4. **Handle Lookup Fields and Multi-Table Relationships**
   - Investigate how Dataverse lookup fields work and their impact on the source:
     - Understand the structure of lookup field data (e.g., how references to other entities are represented)
     - Determine how lookup fields relate to entity IDs (connection to Goal #2)
     - Explore how lookup values are returned in API responses (navigation properties, expanded data, etc.)
   - Design how lookup fields should be represented in Drasi's model:
     - Should lookups become edges/relationships between nodes?
     - How to handle lookup metadata (display name, type information)
   - Explore supporting multiple Dataverse tables/entities in a single source:
     - Investigate if a single source instance can monitor multiple tables simultaneously
     - Determine how to configure multi-table sources (table list, filters per table)
   - Define the approach for handling standard lookup fields (e.g., Account lookups from Contact)

5. **Define Minimum Required Permissions and Roles**
   - Investigate and document the minimum Dataverse permissions required for the source to function:
     - Read permissions on specific entities/tables
     - Change tracking permissions
     - Any additional API-level permissions needed
   - For managed identity (Entra Workload ID) specifically:
     - Determine the minimum Azure AD roles required for the workload identity
     - Document required Dataverse security roles and application user setup
     - Identify the minimum privilege level needed for each operation:
       - Initial bootstrap/data fetch
       - Change tracking queries
       - Metadata queries (schema information)
   - For client secret authentication:
     - Document required app registration permissions
     - Define minimum API permissions in Azure AD
   - Create a permissions setup guide that follows the principle of least privilege
   - Test with minimal permissions to ensure the source functions correctly without over-provisioning

### Non-Goals

The following are explicitly out of scope for this rewrite:

- **Dataverse Reaction**: This design document focuses solely on the Dataverse source component; the Dataverse reaction is out of scope.

## Design requirements

<!-- Describe key technical requirements, dependencies, and constraints that impact the design.

Provide a link to the issue(s) tracking this work. -->

### Requirements

<!-- List design goals or constraints that impact the design. Examples include extensibility points, maintainability, code reuse, considering if your feature needs to be taken down-level and coherency with related code.

For the Dataverse source rewrite, consider:
- Must implement both Reactivator and Proxy components
- Must use one of the Drasi SDKs (Rust, Java, or .NET)
- Must handle Dataverse authentication (OAuth 2.0, Service Principal)
- Must support change tracking via Dataverse Change Tracking API
- Must handle network failures and implement retry logic
- Must support configurable polling intervals
- Must transform Dataverse entity data into Drasi node/edge format
-->



### Dependencies

<!-- Describe the key dependencies that this design has on other features being developed as well as the dependencies other features have on this design.

For Sources, typical dependencies include:
- Drasi SDK version (Rust/Java/.NET)
- Query Container compatibility
- Control Plane API for Source registration
- Kubernetes Resource Provider for deployment
- GitHub Actions workflows (build-test, draft-release)
- E2E test framework

External dependencies:
- Microsoft Dataverse API
- Authentication libraries (MSAL, Azure Identity)
-->

### Out of scope

<!-- Describe the important issues or concerns NOT addressed by this proposal. 1 paragraph each (can be short). Often the "out of scope" items refer to the current state. We might already have planned work to address the "out of scope" items, mention it if that's the case. -->

## Design

### High-level design

### Architecture Diagram

<!-- Provide an architecture design if applicable. An architecture diagram should be provided when the design involves multiple parts of Drasi (Communication between RPs, Communication between CLI and RPs, Protocol changes for UCP, Communication with external systems like clouds, etc.) -->

N/A

### Detail design

#### 1. Change Fetch Interval and Rate Limiting (Goal #1)

**Research Findings**

**Dataverse API Limits Overview:**

Dataverse enforces two separate API limit systems:

**1. Short-term Service Protection Limits (Per Principal, 5-Minute Window):**
- **6,000 requests per 300 seconds (5 minutes)** - equivalent to 1,200 requests/minute maximum
- **20 minutes of execution time** per 5-minute window
- **52 concurrent requests** maximum
- **Scope**: Applies per Managed Identity (or user) in each environment
- **Behavior**: Each Managed Identity gets its own independent 6,000-request bucket
- **Throttling**: Automatic retry with `Retry-After` header on 429 responses
- **Note**: These limits cannot be increased through purchase

**2. 24-Hour Request Entitlement (Tenant-Wide Pool):**
- **Licensed users**: 40,000 requests per day per user
- **Managed Identities/Service Principals**: Do NOT have individual 40,000 request limits
- **Tenant pool**: All non-licensed principals (MIs, service principals, application users) draw from a shared tenant-wide pool
- **Pool calculation**: 25,000 base + 5,000 × (number of unique paid licensed users), capped at 10,000,000
- **Reporting**: Power Platform Admin Center → Analytics → Capacity shows combined daily usage
- **Throttling**: When pool exhausted, all principals receive 429 Too Many Requests (after 6-month grace period)
- **Add-ons**: 50,000-request capacity add-ons available for purchase to expand the pool

**Impact for Drasi with Managed Identity:**
- Each Managed Identity has dedicated 6,000 req/5-min protection limits (not shared)
- All Managed Identities share the tenant-wide 24-hour pool



**Change Tracking API Characteristics:**
- Each API call tracks **only one table/entity**
- Pagination returns max **5,000 records per response**
- Delta tokens expire after **7 days** (configurable via `ExpireChangeTrackingInDays`). This does not affect us as we are storing the delta token in the dapr state store.

**Rate Limit Analysis:**

**Short-term (5-minute) Protection Limits:**

With current 60-second interval and single entity:
- 5 requests per 5-minute window
- Far below the 6,000 request limit (<0.1%)

With multiple entities, the formula scales:
- **Polling Interval = Entity Count / 10** (using 50% safety margin of 6,000 limit)

| Entities | Safe Interval | Requests per 5-min | % of 5-min Limit |
|----------|---------------|-------------------|------------------|
| 5        | 0.5s          | 3,000             | 50%              |
| 10       | 1s            | 3,000             | 50%              |
| 50       | 5s            | 3,000             | 50%              |
| 100      | 10s           | 3,000             | 50%              |
| 500      | 50s           | 3,000             | 50%              |

- **Short-term (5-min) limits are NOT the constraint** - they're easily satisfied
- **24-hour tenant pool IS the constraint** - requires careful capacity planning
- Default polling interval should be **conservative** (e.g., 30-60 seconds) to avoid exhausting tenant pools
- Adaptive polling helps by slowing down during idle periods

**Why Webhooks Won't Work:**

Dataverse does support webhooks through the Plugin Registration system, but they are **not suitable** for the Drasi source use case:

1. **Administrative Overhead**: Webhooks must be pre-registered using the Plugin Registration Tool with Dataverse admin privileges. Users cannot dynamically add tables via YAML configuration.

2. **Not Self-Service**: Each table requires manual webhook registration, activation, and endpoint configuration before it can send events.

3. **No Historical Context**: Webhooks only fire for new changes after registration. If the source restarts or goes offline, there's no mechanism to "catch up" on missed changes.

4. **Event-by-Event Model**: Webhooks deliver individual operation events (one Create, one Update, one Delete at a time), not bulk synchronization batches needed for continuous queries.

5. **Operational Complexity**: Requires managing webhook lifecycle, public endpoint exposure, webhook authentication, and event deduplication.

**Conclusion**: The Change Tracking API with polling is the appropriate mechanism for Drasi's self-service, catch-up-capable data synchronization model.


**Proposed Design**

The source will implement an **adaptive polling strategy with exponential backoff** to balance responsiveness and API efficiency. Each worker will maintain its own polling interval that starts at a minimum value (e.g., 1 second) and doubles on each poll that returns no changes, up to a configurable maximum (e.g., 60 seconds). When the reactivator starts, it will immediately check to see if there are any changes.

When changes are detected, the interval immediately resets to the minimum, ensuring fast response during active periods while conserving API quota during quiet periods. Each entity operates independently, so active entities poll frequently while idle entities naturally slow down. HTTP 429 throttling responses will be handled using the `Retry-After` header without affecting the backoff state.



#### 2. Dataverse ID Handling (Goal #2)

**Current Implementation:**

```csharp
// For upserts
data.Id = upsert.NewOrUpdatedEntity.Id.ToString();

// For deletes
data.Id = delete.RemovedItem.Id.ToString();
```

The current implementation is correct. Dataverse entities use the `Entity.Id` property which contains a GUID primary key (e.g., `e169c80c-a8bc-4bf5-b6ec-5a9de4474d5f`). This property is entity-agnostic and works for any table type (Account, Contact, custom entities) without needing to know the specific primary key field name (accountid, contactid, etc.).

**Interaction with Lookup Fields:**

Lookup fields contain an `EntityReference` object which also has an `Id` property, but this is handled correctly because:

1. **Entity-level ID extraction happens at the top level**: `Entity.Id` extracts the ID of the **current entity** being processed (e.g., the DisasterAlert record)
2. **Lookup field IDs are nested inside attributes**: The lookup field's `EntityReference.Id` (e.g., the Employee ID) is serialized as part of the attribute value in the JSON properties, not at the entity level

**Example:**
```csharp
// Entity being processed: DisasterAlert
Entity disasterAlert = upsert.NewOrUpdatedEntity;

// This gets the DisasterAlert's ID (correct)
data.Id = disasterAlert.Id.ToString();  
// Result: "12345678-1234-1234-1234-123456789abc"

// Lookup field is serialized separately as an attribute
var employeeLookup = disasterAlert.Attributes["cr06a_employeename"] as EntityReference;
// employeeLookup.Id is "87654321-4321-4321-4321-210987654321" (Employee ID)
// This gets serialized into the properties, not confused with the entity ID
```

The two ID contexts are separate:
- **`Entity.Id`**: The primary key of the entity being synchronized (DisasterAlert)
- **`EntityReference.Id`**: The foreign key reference embedded in a lookup field (Employee)

**Conclusion:** The ID handling is correct and does not conflict with lookup fields. **No changes needed.**


#### 3. Authentication Methods (Goal #3)
Currently, the Drasi Dotnet SDK supports the following authentication methods:
- None
- MicrosoftEntraWorkloadID
- ConnectionString
- AccessKey

During the initial testing stage, we have verified that we were able to authenticate to Dataverse using `MicrosoftEntraWorkloadID` method. However, the current implementation of the Dataverse source uses OAuth 2.0 client credentials flow (client ID and secret), which is not yet supported by the Drasi Dotnet SDK. We need to verify if this authentication method is still supported in Dataverse, and implement it in the SDK.

#### 4. Lookup Fields and Multi-Table Relationships (Goal #4)

The Dataverse reactivator uses the Change Tracking API to monitor changes on a per-entity basis. This means that the reactivator will spawn a sync worker that is responsible for tracking changes for each configured entity/table, and we cannot monitor multiple entities in a single worker. 

Therefore, to support multiple tables/entities in a single source, we will allow users to specify a list of entities in the source configuration. Each entity will be tracked independently, and the reactivator will manage separate state (delta tokens, polling intervals) for each entity. This is consistent with the current design of the source.

**How Lookup Fields Are Represented:**

When a Dataverse entity contains a lookup field (a reference to another entity), it is returned as an **`EntityReference`** object with the following properties:

```csharp
// Example: DisasterAlert entity with a lookup to Employee (cr06a_employeename field)
EntityReference lookup = entity.Attributes["cr06a_employeename"] as EntityReference;

// EntityReference properties:
lookup.Id;          // Guid - ID of the referenced entity (e.g., employee ID)
lookup.LogicalName; // String - Logical name of the referenced entity (e.g., "cr06a_employee")
lookup.Name;        // String - Primary name value of the referenced entity (e.g., "John Smith")
```

**Current Code Handling:**

In your `JsonEventMapper`, lookup fields are serialized along with all other attributes:

```csharp
foreach (var attribute in upsert.NewOrUpdatedEntity.Attributes)
{
    var val = JsonSerializer.SerializeToNode(attribute.Value);                        
    props.Add(attribute.Key, val);
}
```

When `attribute.Value` is an `EntityReference`, `JsonSerializer.SerializeToNode()` will serialize it as a JSON object containing the `Id`, `LogicalName`, and `Name` properties.

**Example JSON Output:**

```json
{
  "cr06a_disasteralertid": "12345678-1234-1234-1234-123456789abc",
  "cr06a_alerttitle": "Emergency Alert",
  "cr06a_employeename": {
    "Id": "87654321-4321-4321-4321-210987654321",
    "LogicalName": "cr06a_employee",
    "Name": "John Smith"
  }
}
```

**Design Decision:**

✅ **Simple lookups are fully supported.** The current implementation correctly serializes `EntityReference` objects with `Id`, `LogicalName`, and `Name` properties. We will serialize the lookup fields as-is without modification. 

Depending on the use case, there are multiple ways that users can use the lookup field data in their continuous queries. For example:

##### Use Drasi Middleware:

Users can leverage **Drasi's `promote` middleware** to extract lookup field properties to the top level for easier querying:

```yaml
apiVersion: v1
kind: ContinuousQuery
name: my-query
spec:
    <...>
    middleware:
      - name: extract-lookup-fields
        kind: promote
        config:
          mappings:
            - path: "$.cr06a_employeename.Id"
              target_name: "employeeId"
            - path: "$.cr06a_employeename.Name"
              target_name: "employeeName"
  query: >
    MATCH (da:DisasterAlert)
    WHERE da.employeeName = 'John Smith'
    RETURN da.cr06a_alerttitle, da.employeeId, da.employeeName
```

This transforms the nested lookup structure:
```json
{
  "cr06a_employeename": {
    "Id": "87654321-4321-4321-4321-210987654321",
    "LogicalName": "cr06a_employee",
    "Name": "John Smith"
  }
}
```

Into flattened properties:
```json
{
  "cr06a_employeename": { /* original preserved */ },
  "employeeId": "87654321-4321-4321-4321-210987654321",
  "employeeName": "John Smith"
}
```


**Benefits of this approach:**
- No source code changes required
- User-configurable per continuous query
- Preserves original nested data for other queries
- Standard Drasi pattern for data transformation
- In case if the user has multiple Drasi Dataverse sources monitoring different tables, they can create relationships in their queries by matching on the lookup IDs.



### API Design

<!-- Include if applicable -- any design that changes our public REST API, CLI arguments/commands, or Go APIs for shared components should provide this section. Write N/A here if not applicable. -->


#### CLI Commands
No changes to CLI commands are required for this source rewrite.

#### Control Plane Integration
We might need to update the supported authentication methods for this source type.




### Alternatives Considered

<!-- Describe the alternative designs that were considered or should be considered. Give a justification for why alternative approaches should be rejected if possible.

Examples for a Source rewrite:
1. **Alternative SDK choice** - Why was Rust/Java/.NET chosen over the others?
2. **Change detection approach** - Polling vs. webhooks vs. change tracking
3. **State management** - In-memory vs. persistent storage
4. **Authentication method** - Service Principal vs. Interactive login
-->

## Security

<!-- Optional. Use this section to describe the security threats and its mitigations with this design---such as authenticating request, storing secrets and credentials, etc.

For the Dataverse source, consider:
- How are Dataverse credentials stored? (Kubernetes secrets)
- How is authentication performed? (OAuth 2.0, Service Principal)
- Are credentials rotated? How is rotation handled?
- Is data encrypted in transit? (TLS)
- What permissions are required on the Dataverse side?
- How is the principle of least privilege applied?
- Are there any secrets in logs or error messages?
-->

Dataverse credentials will be stored securely using Kubernetes secrets. 

## Compatibility impact

<!-- Optional. Describe the potential compatibility issues with the other components---such as incompatible with older CLI. Include breaking changes to behaviors or APIs here.

For a Source rewrite:
- Will existing Dataverse source configurations need to be migrated?
- Are there breaking changes to the configuration schema?
- Is there a migration path for users of the old source?
- Does this affect any existing continuous queries?
- Are there any changes to the data format sent to Query Containers?
-->

N/A

## Supportability

Add `agents.md` to the dataverse source directory.

### Telemetry

<!-- This includes the list of instrumentation such as metric, log, and trace to diagnose this new feature.

For a Source, consider:
- Metrics: connection status, records processed, lag time, error rates
- Logs: connection events, authentication failures, data transformation errors
- Traces: end-to-end tracing from Dataverse to Query Container
- Health check endpoint responses
-->

Perhaps we can add a health check endpoint to verify:
1. Connectivity to Dataverse.
2. Dapr sidecar status.

### Verification

<!-- This includes the test plan to validate the features such as unit-test and functional tests.

For the Dataverse source rewrite:

#### Unit Tests
- Configuration parsing and validation
- Data transformation logic
- Error handling scenarios
- Authentication mock tests

#### Integration Tests
- Connection to Dataverse sandbox environment
- Change detection accuracy
- Bootstrap process
- Reconnection after network failure

#### E2E Tests
- Create a new E2E test in the `e2e-test` directory
- Test scenario: Deploy source → Create continuous query → Verify change detection
- Consider creating a test Dataverse environment with sample data

#### Performance Tests
- Throughput testing with high-volume changes
- Memory usage under sustained load
- Latency measurements
-->

## Development Plan

<!-- This section is for planning how you will deliver your features. This includes aligning work items to features, scenarios or requirements, defining what deliverable will be checked in at each point in the product and estimating the cost of each work item. Don't forget to include the Unit Test and functional test in your estimates.

Example breakdown:
1. **Phase 1: Core Reactivator** (X weeks)
   - Implement Dataverse connection and authentication
   - Implement bootstrap logic
   - Unit tests

2. **Phase 2: Change Detection** (X weeks)
   - Implement change tracking integration
   - Implement polling mechanism
   - Integration tests

3. **Phase 3: Proxy Component** (X weeks)
   - Implement data transformation
   - Handle relationships and metadata
   - Unit tests

4. **Phase 4: Configuration & Deployment** (X weeks)
   - Define configuration schema
   - Update Control Plane integration
   - Update GitHub Actions workflows (build-test, draft-release)

5. **Phase 5: Testing & Documentation** (X weeks)
   - Create E2E test
   - Performance testing
   - Documentation updates
   - Migration guide (if applicable)

Don't forget:
- Update the `build-test` workflow
- Update the `draft-release` workflow
- Create E2E test in the `e2e-test` directory
-->

## Open issues

<!-- Describe (Q&A format) the important unknowns or things you're not sure about. Use the discussion of to answer these with experts after people digest the overall design.

-->

## Appendices

<!-- Optional. Describe additional information supporting this design. For instance, describe the details of alternative design if you have one. -->

### Research Notes

**Service Protection API Limits (Per Principal, 5-Minute Sliding Window):**
- **6,000 requests per 5-minute window** (= 1,200 requests/minute max)
- **20 minutes (1,200 seconds) of execution time** per 5-minute window
- **52 concurrent requests** maximum
- **Scope**: Per Managed Identity or user in each environment
- Each principal gets independent limit bucket
- Automatic retry with Retry-After header on 429 responses
- Only one table is tracked per retrieve changes API call

**24-Hour Request Entitlement (Tenant-Wide Pool):**

| Aspect | Detail |
|--------|--------|
| **Scope** | Tenant-wide pool for all non-licensed principals (service principals, managed identities, application users) |
| **Managed Identity behavior** | Does not have a personal 40,000 limit because it isn't a licensed user. Every request subtracts from tenant's pooled capacity |
| **Typical pool size** | 25,000 base + 5,000 × (number of unique paid licensed users) → capped at 10,000,000 |
| **Reporting** | Power Platform Admin Center → Analytics → Capacity shows combined daily usage of all MIs + service principals |
| **Throttling** | When pool exhausted, every principal (including MI) receives 429 Too Many Requests after 6-month grace period |
| **Add-ons** | Can purchase 50,000-request add-ons to expand pool |
| **Licensed users** | Each gets personal 40,000 requests/day (not from shared pool) |

**Impact of Smaller Intervals:**

With the default 60-second interval:
- You make 5 requests per 5-minute window
- Very safe - nowhere near the 6,000 request limit

If you reduce to a 5-second interval (12x faster):
- You'd make ~60 requests per 5-minute window
- Still very safe - only 1% of the limit

Even at a 1-second interval (60x faster):
- You'd make ~300 requests per 5-minute window
- Still safe - only 5% of the limit

**Additional Considerations:**

Delta Token Expiration:
- Changes are tracked for 7 days by default (configurable via ExpireChangeTrackingInDays)
- If you don't poll within 7 days, you lose the change history

Pagination:
- Each response returns max 5,000 records
- If you have more changes, pagination handles it automatically

Throttling Response:
- If throttled, you'll get a 429 Too Many Requests error with a Retry-After header
- The .Execute() call will throw an exception with retry timing

**Actual Server Ordering:**
1. All new/updated records first (sorted by version number within this group)
2. All deleted records last

Example: If the actual chronological sequence was:
- 10:00 AM - Record A updated
- 10:01 AM - Record B deleted
- 10:02 AM - Record C updated
- 10:03 AM - Record D deleted

You would receive them in this order:
1. Record A updated
2. Record C updated
3. Record B deleted
4. Record D deleted

**Dynamic Interval Calculation Based on Entity Count:**

The Formula:
```
Polling Interval (seconds) = (Number of Entities × 300) / Target Request Limit

With safety margin (recommended 50-75% of max limit):
Polling Interval = (Number of Entities × 300) / 3000  // Using 50% of 6000 limit
                 = Number of Entities / 10
```

Examples:

| Entities | Safe Interval | Request Rate per 5-min |
|----------|---------------|------------------------|
| 5        | 0.5s          | 3,000 requests         |
| 10       | 1s            | 3,000 requests         |
| 20       | 2s            | 3,000 requests         |
| 50       | 5s            | 3,000 requests         |
| 100      | 10s           | 3,000 requests         |
| 500      | 50s           | 3,000 requests         |
| 1000     | 100s (1.67m)  | 3,000 requests         |

**Implementation Approaches:**

Option 1: Calculate at Startup
- Count how many entities are configured to sync
- Calculate shared interval: interval = entityCount / 10
- All SyncWorker instances use the same calculated interval

Option 2: Configuration-Based
- User specifies number of entities in config
- Application calculates interval automatically
- Log the calculated value for transparency

Option 3: Dynamic Discovery
- Query Dataverse at startup for all tables with change tracking enabled
- Auto-calculate based on actual count
- Recalculate if new entities are added

**Benefits:**
- ✅ Simple: One calculation, no complex runtime logic
- ✅ Predictable: Guaranteed to stay under limits
- ✅ Scales automatically: More entities → longer intervals
- ✅ No throttling risk: Always within safe bounds
- ✅ Easy to reason about: Clear cause and effect

## References

### Official Documentation
- [Drasi Platform Repository](https://github.com/drasi-project/drasi-platform)
- [Drasi Platform Sources](https://github.com/drasi-project/drasi-platform/tree/main/sources)
- [Microsoft Dataverse Web API Reference](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview)
- [Dataverse Change Tracking](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/use-change-tracking-synchronize-data-external-systems)
- [Dataverse API Limits](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/api-limits)

