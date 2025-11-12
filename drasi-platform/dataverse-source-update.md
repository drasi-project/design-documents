# Dataverse Source Update

* Project Drasi - 2025-11-11 - Ruokun Niu (@ruokun-niu)

## Overview

The Dataverse Source is a critical component of Drasi that enables integration with Microsoft Dataverse, allowing continuous queries to monitor and react to data changes in Dataverse environments. The current implementation has several limitations that impact its security, performance, and maintainability. This design document outlines the updates needed to modernize the Dataverse Source implementation.

The primary improvements include: adding support for managed identity authentication to enhance security, migrating to the Drasi .NET Source SDK for better maintainability and consistency with other sources, implementing a more efficient polling algorithm to reduce unnecessary API calls, and expanding data type support to handle complex Dataverse types like lookups.

These updates will make the Dataverse Source more secure, efficient, and easier to maintain while expanding its capability to handle a broader range of Dataverse data scenarios.

## Terms and definitions

| Term | Definition |
|------|------------|
| Managed Identity | Azure Active Directory (AAD) authentication mechanism that provides an automatically managed identity for Azure resources to authenticate to services without storing credentials in code |
| Dataverse | Microsoft's cloud-based data platform that provides secure storage and management of business data |
| Lookup | A Dataverse data type that represents a reference to a record in another table, similar to a foreign key relationship |
| Polling Interval | The time period between consecutive checks for new data changes in the source system |
| Reactivator | The Drasi component responsible for detecting and processing data changes from the source |
| Proxy | The Drasi component that provides initial data and handles queries to the source system |
| Source SDK | A software development kit that provides common patterns and utilities for building Drasi sources |

## Objectives

### User scenarios

**Scenario 1: Secure Enterprise Integration**
- **Persona**: Enterprise IT Administrator
- **Goal**: Deploy Drasi with Dataverse Source in a production environment without storing credentials
- **Requirements**: The administrator needs to authenticate using Azure Managed Identity instead of storing service principal credentials in configuration files

**Scenario 2: Efficient Change Detection**
- **Persona**: DevOps Engineer
- **Goal**: Monitor Dataverse data changes with minimal API call overhead
- **Requirements**: The system should intelligently detect when changes occur and adjust polling frequency accordingly, reducing costs and improving responsiveness

**Scenario 3: Complex Data Modeling**
- **Persona**: Business Application Developer
- **Goal**: Create continuous queries that work with Dataverse entities containing lookup fields
- **Requirements**: The developer needs to query and track changes in entities that have lookup fields, with the lookup values properly extracted and available in queries

### Goals

- **Security Enhancement**: Implement managed identity authentication to eliminate the need to store credentials, improving security posture
- **SDK Migration**: Migrate to the Drasi .NET Source SDK to ensure consistency with other sources and leverage common functionality
- **Performance Optimization**: Implement an adaptive polling algorithm that reduces API calls during quiet periods while maintaining responsiveness during active periods
- **Data Type Support**: Add support for Dataverse lookup and other complex data types (currency, choice/picklist) with proper value extraction
- **Documentation**: Provide comprehensive documentation for configuration, authentication options, and data type handling

### Non-Goals

- Implementing real-time streaming from Dataverse (continue using polling-based approach)
- Supporting all Dataverse data types in the initial release (focus on lookup type specifically)
- Backward compatibility with the current implementation's configuration format
- Supporting authentication methods beyond managed identity and existing service principal authentication
- Performance optimization of the query execution itself (focus is on change detection)

## Design requirements

### Requirements

**Track issue**: https://github.com/drasi-project/drasi-platform/issues/338

**Functional Requirements**:
1. Support authentication via Azure Managed Identity
2. Maintain support for existing service principal authentication as a fallback
3. Use the Drasi .NET Source SDK for both proxy and reactivator components
4. Implement an adaptive polling algorithm that adjusts based on change frequency using Dataverse's native change tracking capability
5. Handle Dataverse complex data types (lookup, currency, choice/picklist) correctly in both initial load and change detection by extracting their values
6. Provide clear error messages for authentication and configuration issues

**Non-Functional Requirements**:
1. Maintain or improve current performance for initial data load
2. Reduce API call volume by at least 50% during periods with no changes
3. Ensure backward compatibility of the query interface (users should not need to change their continuous queries)
4. Follow Drasi security best practices for credential handling
5. Provide comprehensive logging and telemetry for troubleshooting

**Extensibility**:
- Design should allow easy addition of new Dataverse data types in the future
- Polling algorithm should be configurable to allow tuning for different environments
- Authentication mechanism should be extensible for future authentication types

### Dependencies

**Drasi .NET Source SDK**:
- The current .NET Source SDK must support all required functionality for Dataverse integration
- If the SDK lacks necessary features, it must be extended

**Azure Identity Libraries**:
- Requires Azure.Identity NuGet package for managed identity support
- Must be compatible with the .NET version used by the SDK

**Dataverse SDK**:
- Microsoft.PowerPlatform.Dataverse.Client or equivalent for Dataverse API access
- Must support managed identity and environment variable-based authentication

**Dapr**:
- Requires Dapr state store for persisting change tracking tokens
- State store name: `drasi-state`

**Control Plane**:
- Configuration schema changes for identity field structure
- Resource provider should support deploying sources with identity configuration

**Build and Release Pipelines**:
- GitHub Actions workflows are already configured to build and publish the Dataverse source
- May need to update the `default-source-provider` file if new environment variables are introduced

### Out of scope

**Real-time Change Feed**: While Dataverse provides change tracking capabilities for incremental synchronization, it does not offer real-time push notifications. This design leverages Dataverse's change tracking API (see [Use change tracking](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/use-change-tracking-synchronize-data-external-systems)) with adaptive polling to efficiently detect changes.

**Complete Data Type Coverage**: This design focuses on the most common complex data types (lookup, currency, choice/picklist). Supporting all Dataverse data types (e.g., Customer, Owner, PartyList, etc.) is deferred to future iterations.

**Multi-tenant Authentication**: This design focuses on single-tenant scenarios. Multi-tenant authentication patterns are not addressed.

**Historical Data Analysis**: The design focuses on forward-looking change detection, not analyzing historical change patterns beyond the current polling state.

## Design

### High-level design

The updated Dataverse Source will consist of two main components, both refactored to use the Drasi .NET Source SDK:

1. **Dataverse Reactivator**: Responsible for detecting changes in Dataverse and publishing them to the Drasi query container
   - Uses Dataverse's native change tracking API with adaptive polling to efficiently detect changes
   - Authenticates using managed identity or service principal
   - Tracks change tokens from Dataverse change tracking to retrieve only new/modified/deleted records
   - Extracts lookup, currency, and choice values from entity attributes

2. **Dataverse Proxy**: Provides initial data snapshots and handles node queries
   - Authenticates using the same mechanism as reactivator
   - Executes FetchXML queries against Dataverse tables
   - Extracts complex data type values (lookup, currency, choice) during initial load

Both components will share common authentication and configuration logic through the SDK.

**Key Improvements**:
- **Change Tracking with Adaptive Polling**: Leverages Dataverse's built-in change tracking to retrieve only changed records, with automatic polling interval calculation based on entity count (user can override with maxInterval)
- **Persistent State**: Uses Dapr state store to persist change tracking tokens, ensuring reliability across restarts
- **Flexible Authentication**: Supports managed identity (MicrosoftEntraWorkloadID) or environment variables for credentials (stored as Kubernetes secrets)
- **SDK-based Architecture**: Common patterns for configuration, logging, and telemetry provided by the SDK

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Drasi Query Container                    │
│                                                              │
│  ┌────────────────┐                  ┌──────────────────┐  │
│  │  Initial Load  │◄─────────────────┤ Dataverse Proxy  │  │
│  │   (Snapshot)   │                  │                  │  │
│  └────────────────┘                  └──────────────────┘  │
│         ▲                                      │            │
│         │                                      │            │
│         │                                      ▼            │
│  ┌──────┴───────┐                  ┌──────────────────┐    │
│  │   Change     │◄─────────────────┤   Dataverse      │    │
│  │   Stream     │                  │  Reactivator     │    │
│  └──────────────┘                  └──────────────────┘    │
│                                              │              │
└──────────────────────────────────────────────┼──────────────┘
                                               │
                                               │ Adaptive
                                               │ Polling
                                               ▼
                           ┌──────────────────────────────────┐
                           │   Microsoft Dataverse            │
                           │                                  │
                           │  ┌────────────┐ ┌─────────────┐ │
                           │  │  Account   │ │  Contact    │ │
                           │  │  Table     │ │  Table      │ │
                           │  └────────────┘ └─────────────┘ │
                           │                                  │
                           │  Authentication:                 │
                           │  - Managed Identity              │
                           │  - Service Principal             │
                           └──────────────────────────────────┘

SDK Components (Shared):
┌────────────────────────────────────────────────┐
│  Drasi .NET Source SDK                         │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Config       │  │ Authentication       │   │
│  │ Management   │  │ Provider             │   │
│  └──────────────┘  └──────────────────────┘   │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Logging &    │  │ Change Publisher     │   │
│  │ Telemetry    │  │ Interface            │   │
│  └──────────────┘  └──────────────────────┘   │
└────────────────────────────────────────────────┘
```

### Detail design

#### 1. Authentication Mechanism

**Managed Identity Authentication**:
```csharp
// Using Azure.Identity for managed identity
public interface IDataverseAuthProvider
{
    Task<ServiceClient> GetAuthenticatedClientAsync();
}

public class ManagedIdentityAuthProvider : IDataverseAuthProvider
{
    private readonly string _instanceUrl;
    private readonly ManagedIdentityCredential _credential;
    
    public ManagedIdentityAuthProvider(string instanceUrl, string clientId = null)
    {
        _instanceUrl = instanceUrl;
        _credential = clientId != null 
            ? new ManagedIdentityCredential(clientId)
            : new ManagedIdentityCredential();
    }
    
    public async Task<ServiceClient> GetAuthenticatedClientAsync()
    {
        var token = await _credential.GetTokenAsync(
            new TokenRequestContext(new[] { $"{_instanceUrl}/.default" }));
        
        return new ServiceClient(
            instanceUri: new Uri(_instanceUrl),
            tokenProviderFunction: async (uri) => token.Token);
    }
}

public class EnvironmentVariableAuthProvider : IDataverseAuthProvider
{
    private readonly string _instanceUrl;
    
    public EnvironmentVariableAuthProvider(string instanceUrl)
    {
        _instanceUrl = instanceUrl;
    }
    
    public async Task<ServiceClient> GetAuthenticatedClientAsync()
    {
        // Read from environment variables (can be set from Kubernetes secrets)
        var clientId = Environment.GetEnvironmentVariable("DATAVERSE_CLIENT_ID");
        var tenantId = Environment.GetEnvironmentVariable("DATAVERSE_TENANT_ID");
        var clientSecret = Environment.GetEnvironmentVariable("DATAVERSE_CLIENT_SECRET");
        
        var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
        var token = await credential.GetTokenAsync(
            new TokenRequestContext(new[] { $"{_instanceUrl}/.default" }));
        
        return new ServiceClient(
            instanceUri: new Uri(_instanceUrl),
            tokenProviderFunction: async (uri) => token.Token);
    }
}
```

**Configuration Schema**:
```yaml
# Example source configuration
kind: Source
apiVersion: v1
name: my-dataverse
spec:
  kind: Dataverse
  identity:
    type: MicrosoftEntraWorkloadID
    # Optional: specify client ID for user-assigned managed identity
    clientId: "optional-client-id"
  properties:
    instanceUrl: https://myorg.crm.dynamics.com
    
    # Polling configuration
    polling:
      maxInterval: 300s  # Optional: maximum polling interval (calculated based on entity count if not specified)
    
    # Tables to monitor
    tables:
      - name: account
      - name: contact

# Alternative: Using environment variables for authentication (no identity section needed)
# Set these as environment variables or Kubernetes secrets:
#   DATAVERSE_CLIENT_ID
#   DATAVERSE_TENANT_ID
#   DATAVERSE_CLIENT_SECRET (use secrets for sensitive values)
```

#### 2. Adaptive Polling Algorithm

The adaptive polling algorithm balances responsiveness with API efficiency by adjusting the polling interval based on change detection patterns. The algorithm automatically calculates optimal polling intervals based on the number of entities being tracked, with a configurable maximum to allow user override.

**Polling State Machine**:
```csharp
public class AdaptivePollingController
{
    private TimeSpan _currentInterval;
    private readonly TimeSpan _minInterval;
    private readonly TimeSpan _maxInterval;
    private readonly int _entityCount;
    
    public AdaptivePollingController(int entityCount, TimeSpan? userMaxInterval = null)
    {
        _entityCount = entityCount;
        _minInterval = TimeSpan.FromSeconds(5);
        
        // Calculate max interval based on entity count, or use user override
        _maxInterval = userMaxInterval ?? CalculateMaxInterval(entityCount);
        _currentInterval = _minInterval;
    }
    
    private TimeSpan CalculateMaxInterval(int entityCount)
    {
        // Algorithm: scale max interval based on entity count
        // More entities = shorter max interval to ensure timely detection
        // Fewer entities = longer max interval to reduce API calls
        if (entityCount <= 10)
            return TimeSpan.FromMinutes(10);
        else if (entityCount <= 50)
            return TimeSpan.FromMinutes(5);
        else
            return TimeSpan.FromMinutes(2);
    }
    
    public TimeSpan GetNextInterval(bool changesDetected)
    {
        if (changesDetected)
        {
            // Reset to minimum interval on change detection
            _currentInterval = _minInterval;
        }
        else
        {
            // Gradually increase interval up to max
            _currentInterval = TimeSpan.FromSeconds(
                Math.Min(
                    _currentInterval.TotalSeconds * 1.5,  // 50% increase
                    _maxInterval.TotalSeconds
                )
            );
        }
        
        return _currentInterval;
    }
}
```

**Change Detection Flow**:
1. Query Dataverse for changes since last change token
2. If changes found:
   - Publish changes to query container
   - Update change token
   - Reset polling interval to minimum
3. If no changes:
   - Increase polling interval gradually (1.5x multiplier)
   - Wait for next interval

#### 3. SDK Integration

The Dataverse Source will use the [Drasi.Source.SDK NuGet package (v0.1.8-alpha)](https://www.nuget.org/packages/Drasi.Source.SDK/0.1.8-alpha) to ensure consistency with other Drasi sources.

**Reactivator Implementation**:

To build a reactivator, implement `IChangeMonitor` which returns an `IAsyncEnumerable<ChangeNotification>` that yields changes when they occur in the data source.

```csharp
public class DataverseChangeMonitor : IChangeMonitor
{
    private readonly DataverseChangeTracker _changeTracker;
    private readonly AdaptivePollingController _pollingController;
    private readonly ILogger _logger;
    
    public async IAsyncEnumerable<ChangeNotification> GetChanges(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                var changeSet = await _changeTracker.RetrieveChangesAsync();
                var hasChanges = changeSet.NewOrUpdatedEntities.Any() || 
                                 changeSet.RemovedOrDeletedEntityReferences.Any();
                
                if (hasChanges)
                {
                    // Yield all changes
                    foreach (var entity in changeSet.NewOrUpdatedEntities)
                    {
                        yield return CreateChangeNotification(entity);
                    }
                    
                    foreach (var deleted in changeSet.RemovedOrDeletedEntityReferences)
                    {
                        yield return CreateDeleteNotification(deleted);
                    }
                }
                
                var nextInterval = _pollingController.GetNextInterval(hasChanges);
                await Task.Delay(nextInterval, cancellationToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error detecting changes");
                await Task.Delay(_pollingController.GetNextInterval(false), cancellationToken);
            }
        }
    }
}

// Build and start the reactivator
var reactivator = new SourceReactivatorBuilder()
    .UseChangeMonitor<DataverseChangeMonitor>()
    .Build();

await reactivator.StartAsync();
```

**Proxy Implementation**:

To implement a proxy, implement `IBootstrapHandler` which will be invoked when a new query bootstraps. The request will hold the element labels the query is interested in and an `IAsyncEnumerable<SourceNode>` must be returned.

```csharp
public class DataverseBootstrapHandler : IBootstrapHandler
{
    private readonly IDataverseAuthProvider _authProvider;
    private readonly DataTypeValueExtractor _valueExtractor;
    
    public async IAsyncEnumerable<SourceNode> GetBootstrapData(
        BootstrapRequest request,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var client = await _authProvider.GetAuthenticatedClientAsync();
        
        foreach (var label in request.Labels)
        {
            var tableName = MapLabelToTable(label);
            var fetchXml = BuildFetchXmlQuery(tableName);
            var results = await client.RetrieveMultipleAsync(
                new FetchExpression(fetchXml));
            
            foreach (var entity in results.Entities)
            {
                var node = await _valueExtractor.ConvertEntityToNodeAsync(entity);
                yield return node;
            }
        }
    }
}

// Build and start the proxy
var proxy = new SourceProxyBuilder()
    .UseBootstrapHandler<DataverseBootstrapHandler>()
    .Build();

await proxy.StartAsync();
```

#### 4. Complex Data Type Handling

**Incoming Dataverse Entity Structure**:

Dataverse entities returned from the API have the following structure:
```json
{
  "accountid": "00000000-0000-0000-0000-000000000001",
  "name": "Contoso Corp",
  "revenue": {
    "Value": 1000000.00  // Money type
  },
  "primarycontactid": {  // Lookup type (EntityReference)
    "Id": "00000000-0000-0000-0000-000000000002",
    "LogicalName": "contact",
    "Name": "John Doe"
  },
  "industrycode": {  // Choice/OptionSetValue
    "Value": 1
  },
  "accountcategorycode": [  // Multi-select choice (OptionSetValueCollection)
    { "Value": 1 },
    { "Value": 2 }
  ]
}
```

**Data Type Value Extraction**:
```csharp
public class DataTypeValueExtractor
{
    public object ExtractValue(string attributeName, object attributeValue)
    {
        return attributeValue switch
        {
            // Extract lookup value - store the referenced entity details
            EntityReference lookup => new
            {
                Id = lookup.Id.ToString(),
                LogicalName = lookup.LogicalName,
                Name = lookup.Name ?? string.Empty
            },
            
            // Extract money/currency value
            Money money => money.Value,
            
            // Extract choice/picklist value (OptionSetValue)
            OptionSetValue choice => new
            {
                Value = choice.Value,
                // Label can be retrieved from metadata if needed
            },
            
            // Extract multi-select choice value
            OptionSetValueCollection multiChoice => multiChoice
                .Select(o => o.Value)
                .ToArray(),
            
            // Default: return as-is
            _ => attributeValue
        };
    }
    
    public async Task<Node> ConvertEntityToNodeAsync(Entity entity)
    {
        var node = new Node
        {
            Id = entity.Id.ToString(),
            Labels = new[] { entity.LogicalName }
        };
        
        foreach (var attribute in entity.Attributes)
        {
            node.Properties[attribute.Key] = ExtractValue(attribute.Key, attribute.Value);
        }
        
        return node;
    }
}
```

**Dataverse Change Tracking Integration**:
```csharp
public class DataverseChangeTracker
{
    private readonly ServiceClient _client;
    private readonly DaprClient _daprClient;
    private const string STATE_STORE_NAME = "drasi-state";
    
    public async Task<ChangeSet> RetrieveChangesAsync(string tableName)
    {
        // Retrieve data version token from Dapr state store
        var stateKey = $"dataverse-token-{tableName}";
        var dataVersion = await _daprClient.GetStateAsync<string>(STATE_STORE_NAME, stateKey);
        
        // Use Dataverse's RetrieveEntityChanges API
        var request = new RetrieveEntityChangesRequest
        {
            EntityName = tableName,
            Columns = new ColumnSet(allColumns: true),
            DataVersion = dataVersion
        };
        
        var response = (RetrieveEntityChangesResponse)await _client.ExecuteAsync(request);
        
        // Persist version token to Dapr state store for next retrieval
        await _daprClient.SaveStateAsync(
            STATE_STORE_NAME, 
            stateKey, 
            response.EntityChanges.DataToken);
        
        var changeSet = new ChangeSet
        {
            NewOrUpdatedEntities = response.EntityChanges.Changes
                .OfType<NewOrUpdatedItem>()
                .Select(c => c.NewOrUpdatedEntity)
                .ToList(),
                
            RemovedOrDeletedEntityReferences = response.EntityChanges.Changes
                .OfType<RemovedOrDeletedItem>()
                .Select(c => c.RemovedItem)
                .ToList(),
                
            HasMoreRecords = response.EntityChanges.MoreRecords,
            PagingCookie = response.EntityChanges.PagingCookie
        };
        
        return changeSet;
    }
}
```

#### Advantages of this design
- **Security**: Managed identity eliminates credential storage and rotation concerns; environment variables can use Kubernetes secrets
- **Performance**: Dataverse change tracking combined with adaptive polling significantly reduces API calls during quiet periods while maintaining responsiveness
- **Accuracy**: Native change tracking ensures no changes are missed and deleted records are properly detected
- **Maintainability**: SDK integration ensures consistency with other sources and reduces code duplication; Dapr state store provides reliable token persistence
- **Extensibility**: Data type extraction pattern can be easily extended to support additional Dataverse types
- **Simplicity**: Automatic calculation of polling intervals based on entity count reduces configuration complexity

#### Disadvantages
- **Complexity**: Adaptive polling and change tracking add state management complexity
- **Dapr Dependency**: Requires Dapr state store for token persistence
- **Migration**: Existing deployments will need configuration updates
- **Change Tracking Setup**: Requires change tracking to be enabled on Dataverse tables (must be configured by administrators)
- **Testing**: Adaptive polling and change tracking behavior requires comprehensive testing across different change patterns

### API Design

**Configuration API Changes**:

The source configuration schema uses the `identity` field (at the same level as `kind` and `properties`) for authentication. Managed identity can be configured using the `MicrosoftEntraWorkloadID` type, or credentials can be provided via environment variables for scenarios where secrets are stored in Kubernetes.

```yaml
# Identity section (at same level as kind and properties)
identity:
  type: MicrosoftEntraWorkloadID
  clientId?: string  # Optional: user-assigned managed identity

# Polling configuration section (under properties)
polling:
  maxInterval: duration  # Optional: maximum polling interval
                         # If not specified, calculated based on entity count:
                         # - <= 10 entities: 10 minutes
                         # - <= 50 entities: 5 minutes  
                         # - > 50 entities: 2 minutes

# Environment variables option (no identity section needed)
# These can be set as Kubernetes secrets:
#   DATAVERSE_CLIENT_ID
#   DATAVERSE_TENANT_ID
#   DATAVERSE_CLIENT_SECRET
```

**CLI Impact**:

No CLI command changes are required. The existing `drasi apply` command will support the new configuration schema through the standard source YAML files.

Example:
```bash
drasi apply -f dataverse-source.yaml
```

### Alternatives Considered

#### Alternative 1: Webhook-based Change Notification

**Description**: Use Dataverse webhooks to receive push notifications instead of polling.

**Advantages**:
- Real-time change notification
- No API call overhead for checking changes
- More efficient for high-frequency changes

**Disadvantages**:
- Requires exposing an HTTP endpoint accessible from Dataverse
- Complex networking setup in Kubernetes environments
- Webhook delivery reliability concerns
- Initial load still requires polling pattern
- Not all Dataverse deployments support webhooks

**Justification for rejection**: The complexity and infrastructure requirements outweigh the benefits, especially since adaptive polling significantly reduces API overhead. This could be reconsidered in the future.

#### Alternative 2: Continue Without SDK Migration

**Description**: Update authentication and polling without migrating to SDK.

**Advantages**:
- Less refactoring required
- Faster initial implementation

**Disadvantages**:
- Misses opportunity for consistency with other sources
- Duplicate code for logging, telemetry, configuration
- Harder to maintain long-term
- No benefit from SDK improvements

**Justification for rejection**: The SDK migration is a key requirement that provides long-term maintainability benefits and consistency across sources.

#### Alternative 3: Fixed Interval with Rate Limiting

**Description**: Keep fixed polling interval but implement rate limiting to reduce API calls.

**Advantages**:
- Simpler implementation
- Predictable behavior

**Disadvantages**:
- Still makes unnecessary API calls during quiet periods
- Doesn't reduce API costs as effectively
- Less responsive during active periods

**Justification for rejection**: Adaptive polling provides better balance between responsiveness and efficiency.

## Security

### Threat Analysis

**Threat 1: Credential Exposure**
- **Risk**: Service principal credentials stored in configuration could be exposed
- **Mitigation**: 
  - Prefer managed identity authentication (no stored credentials)
  - When service principal is required, store credentials in Kubernetes secrets
  - Support Azure Key Vault integration for credential storage
  - Log warnings when using service principal authentication

**Threat 2: Unauthorized Data Access**
- **Risk**: Overly permissive authentication could allow access to unintended data
- **Mitigation**:
  - Document principle of least privilege for managed identity permissions
  - Managed identity should only have read permissions to specified tables
  - Implement table-level filtering in configuration
  - Log all authentication attempts and data access

**Threat 3: Token Theft**
- **Risk**: Access tokens could be intercepted or stolen
- **Mitigation**:
  - Use Azure.Identity library which handles token caching and renewal securely
  - Tokens are short-lived and automatically refreshed
  - Communication with Dataverse uses HTTPS only
  - No logging of tokens or sensitive authentication data

**Threat 4: Man-in-the-Middle Attacks**
- **Risk**: Network traffic could be intercepted
- **Mitigation**:
  - All communication with Dataverse uses HTTPS/TLS
  - Certificate validation enabled by default
  - No option to disable certificate validation

### Security Best Practices

1. **Default to Managed Identity**: Documentation should recommend managed identity as the primary authentication method
2. **Audit Logging**: All authentication events and configuration changes should be logged
3. **Minimal Permissions**: Sample configurations should demonstrate minimal required permissions
4. **Secret Management**: Clear documentation on using Kubernetes secrets or Azure Key Vault for any stored credentials

## Compatibility impact

### Breaking Changes

This is a complete rewrite of the Dataverse Source, so backward compatibility is not maintained:

1. **Configuration Schema**: The authentication configuration format is new and incompatible with the existing format
   - **Migration Path**: Provide documentation and examples for converting old configuration to new format
   - **Tool Support**: Consider providing a configuration migration script

2. **Lookup Data Format**: Lookup values will be represented differently in node properties
   - **Impact**: Existing continuous queries may need updates if they reference lookup fields
   - **Mitigation**: Document the new format and provide query migration examples

3. **Change Tracking Requirement**: Requires change tracking to be enabled on Dataverse tables by administrators

## Supportability

### Telemetry

**Metrics to Collect**:
- `dataverse_polling_interval_seconds`: Current polling interval (gauge)
- `dataverse_changes_detected_total`: Count of polling cycles with changes (counter)
- `dataverse_api_calls_total`: Total API calls to Dataverse (counter)
- `dataverse_authentication_failures_total`: Authentication failure count (counter)
- `dataverse_data_type_extractions_total`: Data type value extraction count by type (counter)
- `dataverse_entities_processed_total`: Total entities processed (counter)
- `dataverse_change_tracking_version`: Current change tracking version/token by table (gauge)

**Logs to Generate**:
- Authentication method used and success/failure
- Polling interval adjustments and reasons
- Change detection results (summary, not full data)
- Change tracking status and version updates
- Data type extraction operations
- Configuration validation errors
- API errors and retry attempts

**Traces**:
- End-to-end request tracing for change detection
- Authentication flow tracing
- Data type value extraction tracing
- Change tracking API request tracing

### Verification

**Unit Tests**:
- Authentication provider tests (managed identity and service principal)
- Adaptive polling algorithm tests with various change patterns
- Data type value extraction tests (lookup, currency, choice)
- JSON mapping tests for converting Dataverse entities to Drasi nodes
- Configuration parsing and validation tests

**Integration Tests**:
- JSON mapping validation with sample Dataverse entity payloads (no live Dataverse required)
- Mock Dataverse change tracking responses to test change detection logic
- Authentication with managed identity (in Azure environment)
- Authentication with service principal
- Change tracking with Dataverse RetrieveEntityChanges API (requires live Dataverse)
- Change detection across different intervals
- Complex data type handling (lookup, currency, choice)
- Error handling and recovery

**E2E Tests**:
- Deploy source with managed identity in Azure Kubernetes Service
- Deploy source with service principal in any Kubernetes cluster
- Verify initial data load with complex data types
- Verify change detection using Dataverse change tracking
- Verify adaptive polling behavior over time
- Test configuration updates and source restart

**Performance Tests**:
- Measure API call reduction with adaptive polling and change tracking
- Initial load performance with large datasets
- Change processing latency
- Resource consumption (CPU, memory)

## Development Plan

### Phase 1: SDK Migration (2 weeks)
- Refactor reactivator to use .NET Source SDK
- Refactor proxy to use .NET Source SDK
- Migrate existing authentication to SDK patterns
- Update unit tests
- **Deliverable**: Working source using SDK with existing features

### Phase 2: Managed Identity Authentication (1 week)
- Implement ManagedIdentityAuthProvider
- Add configuration schema for authentication options
- Update documentation
- Add authentication tests
- **Deliverable**: Source supporting both managed identity and service principal

### Phase 3: Dataverse Change Tracking Integration (1.5 weeks)
- Implement DataverseChangeTracker using RetrieveEntityChanges API
- Add change tracking validation and error handling
- Integrate adaptive polling with change tracking
- Add telemetry for change detection
- Add change tracking tests
- **Deliverable**: Source using native Dataverse change tracking with adaptive polling

### Phase 4: Complex Data Type Support (1.5 weeks)
- Implement DataTypeValueExtractor for lookup, currency, and choice types
- Update initial load to extract data type values
- Update change detection to handle data type values
- Add data type handling tests
- **Deliverable**: Source with full support for lookup, currency, and choice data types

### Phase 5: Integration and Testing (1 week)
- Integration testing with real Dataverse instance
- E2E testing in Azure and non-Azure environments
- Performance testing and tuning
- Documentation completion
- **Deliverable**: Production-ready source

### Phase 6: Build Pipeline Updates (0.5 weeks)
- Update `default-source-provider` file if new environment variables are needed
- Test automated builds and releases with existing workflows
- **Deliverable**: Automated CI/CD for updated source

**Total Estimated Effort**: 7.5 weeks

### Work Items
1. SDK migration for reactivator
2. SDK migration for proxy
3. Managed identity authentication implementation
4. Service principal fallback implementation
5. Dataverse change tracking integration using RetrieveEntityChanges API
6. Adaptive polling algorithm implementation
7. Data type value extractor implementation (lookup, currency, choice)
8. Configuration schema updates
9. Unit test suite
10. Integration test suite
11. E2E test suite
12. Documentation updates
13. Build pipeline updates (if needed for environment variables)
14. Sample configurations and tutorials

## Open issues

**Q1: Should we support user-assigned managed identities in addition to system-assigned?**
- **Discussion**: User-assigned identities provide more flexibility but add configuration complexity. Should we support both or start with system-assigned only?
- **Recommendation**: Support both, as user-assigned identities are common in enterprise scenarios

**Q2: What should the default polling intervals be?**
- **Discussion**: Need to balance responsiveness vs API costs. Different environments may have different needs.
- **Recommendation**: Start with conservative defaults (min: 5s, max: 5m, initial: 30s, increment: 10s) and allow configuration

**Q3: How should we handle tables where change tracking is not enabled?**
- **Discussion**: Change tracking must be enabled on Dataverse tables by administrators. Should we fall back to polling all records or fail with an error?
- **Recommendation**: Validate at startup and provide clear error messages directing users to enable change tracking. Include links to documentation.

**Q4: Should we implement any caching for choice/picklist labels?**
- **Discussion**: Choice labels are retrieved from metadata. Caching could reduce API calls but adds complexity and staleness concerns.
- **Recommendation**: No caching in initial implementation; store only the numeric value. Revisit based on performance metrics if label resolution is needed.

**Q5: How should we handle Dataverse throttling?**
- **Discussion**: Dataverse has API rate limits. Should we implement backoff or rely on SDK built-in handling?
- **Recommendation**: Implement exponential backoff for throttling responses, coordinated with adaptive polling

**Q6: Should the SDK be extended or should we work around limitations?**
- **Discussion**: If SDK lacks features, should we extend it or implement workarounds in the source?
- **Recommendation**: Extend SDK when features are generally useful, workaround for Dataverse-specific needs

**Q7: How should we handle large change sets that exceed paging limits?**
- **Discussion**: Dataverse change tracking supports paging. Should we process all pages in one cycle or spread across multiple polling intervals?
- **Recommendation**: Process all pages in a single cycle to maintain consistency, with configurable timeout for very large change sets

## Appendices

### Appendix A: Dataverse Change Tracking Overview

Dataverse provides native change tracking capabilities through the `RetrieveEntityChanges` API that maintain a history of data changes. The change tracking feature:
- Tracks create, update, and delete operations
- Provides a data version token to query incremental changes since the last request
- Retains changes for a configurable retention period (default 90 days)
- Returns both the changed entities and information about deleted entities
- Must be explicitly enabled on each table by administrators (see [Enable change tracking](https://learn.microsoft.com/en-us/power-platform/admin/enable-change-tracking-control-data-synchronization))
- Works with the Web API through the `RetrieveEntityChanges` function

Key benefits over polling-based approaches:
- Only retrieves changed records, not all records
- Detects deletions that would otherwise be missed
- Provides guaranteed consistency through version tokens
- More efficient use of API calls and bandwidth

### Appendix B: Sample Configuration Migration

**Old Configuration**:
```yaml
kind: Source
name: my-dataverse
spec:
  kind: Dataverse
  properties:
    url: https://myorg.crm.dynamics.com
    clientId: xxxxx
    clientSecret: xxxxx
    tenantId: xxxxx
```

**New Configuration (using environment variables)**:
```yaml
kind: Source
name: my-dataverse
spec:
  kind: Dataverse
  properties:
    instanceUrl: https://myorg.crm.dynamics.com
    polling:
      maxInterval: 300s  # Optional

# Set these as Kubernetes secrets:
# DATAVERSE_CLIENT_ID=xxxxx
# DATAVERSE_CLIENT_SECRET=xxxxx
# DATAVERSE_TENANT_ID=xxxxx
```

**New Configuration (using managed identity)**:
```yaml
kind: Source
name: my-dataverse
spec:
  kind: Dataverse
  identity:
    type: MicrosoftEntraWorkloadID
  properties:
    instanceUrl: https://myorg.crm.dynamics.com
    polling:
      maxInterval: 300s  # Optional
```

## References

1. [Drasi Platform Repository](https://github.com/drasi-project/drasi-platform)
2. [Drasi .NET Source SDK (NuGet)](https://www.nuget.org/packages/Drasi.Source.SDK/0.1.8-alpha)
3. [Microsoft Dataverse Web API Reference](https://docs.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview)
4. [Azure Managed Identity Documentation](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
5. [Use change tracking to synchronize data with external systems](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/use-change-tracking-synchronize-data-external-systems)
6. [Enable change tracking to control data synchronization](https://learn.microsoft.com/en-us/power-platform/admin/enable-change-tracking-control-data-synchronization)
7. [Issue #338: Update the Dataverse Source](https://github.com/drasi-project/drasi-platform/issues/338)
