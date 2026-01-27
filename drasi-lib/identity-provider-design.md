# Identity Provider for Drasi-lib

* Project Drasi - January 26, 2026 - Ruokun Niu (@ruokun-niu)

## Overview

This design document outlines the implementation of a flexible **Identity Provider** pattern in drasi-core to support multiple authentication methods for database connections (PostgreSQL, MySQL, etc.) and external services. The current Drasi reactions and sources use hardcoded username/password strings for authentication, which limits support for cloud-native authentication mechanisms like Azure Managed Identity and AWS IAM.

The proposed solution introduces a trait-based identity provider abstraction that allows users to define authentication once and reuse it across multiple reactions and sources. This approach centralizes token-fetching logic, reduces code duplication across components, and provides a clean, type-safe API for end users while minimizing the burden on reaction/source developers.

## Terms and definitions

| Term | Definition |
|------|------------|
| Identity Provider | A component that encapsulates authentication logic and provides credentials on demand |
| Managed Identity | Azure's feature that provides an automatically managed identity in Azure AD for applications |
| DefaultAzureCredential | Azure SDK class that tries multiple authentication methods in sequence |
| IAM Authentication | AWS Identity and Access Management authentication for RDS database connections |
| Credential Fetcher | A component responsible for actively retrieving authentication tokens |

## Objectives

### User scenarios
 
**Persona: Drasi Application Developer**

A developer building event-driven applications using Drasi needs to connect reactions and sources to cloud databases. They want to:

#### Scenario 1: Azure Managed Identity Authentication

A developer wants to connect PostgreSQL reactions to Azure Database for PostgreSQL using Managed Identity, avoiding storing passwords in configuration or code.

```rust
use drasi_lib::identity::AzureIdentityProvider;

// Create Azure identity provider using DefaultAzureCredential
// This automatically uses Managed Identity when running in Azure
let azure_identity = AzureIdentityProvider::default();
// Or with explicit managed identity client ID for user-assigned identity
let azure_identity = AzureIdentityProvider::with_managed_identity(
    "user@tenant.onmicrosoft.com",
    "03bbedd2-cce5-45ab-9414-1c1cb82361f0"  // client_id
);

// Use the identity provider with a PostgreSQL reaction
let postgres_reaction = PostgresStoredProcReaction::builder("order-processor")
    .with_hostname("mydb.postgres.database.azure.com")
    .with_database("orders")
    .with_identity_provider(azure_identity)
    .build()
    .await?;
```

#### Scenario 2: Reusable Authentication

A developer needs to connect multiple reactions to different databases using the same Azure credentials, reducing configuration duplication.

```rust
use drasi_lib::identity::AzureIdentityProvider;

// Define identity provider ONCE at the application level
let azure_identity = AzureIdentityProvider::default();

// Reuse across multiple reactions - just clone the provider!
let orders_reaction = PostgresStoredProcReaction::builder("orders-reaction")
    .with_hostname("orders-db.postgres.database.azure.com")
    .with_database("orders")
    .with_identity_provider(azure_identity.clone())  // Clone and reuse
    .build()
    .await?;

let inventory_reaction = PostgresStoredProcReaction::builder("inventory-reaction")
    .with_hostname("inventory-db.postgres.database.azure.com")
    .with_database("inventory")
    .with_identity_provider(azure_identity.clone())  // Same identity, different DB
    .build()
    .await?;

let analytics_reaction = PostgresStoredProcReaction::builder("analytics-reaction")
    .with_hostname("analytics-db.postgres.database.azure.com")
    .with_database("analytics")
    .with_identity_provider(azure_identity.clone())  // Reuse again!
    .build()
    .await?;
```

#### Scenario 3: AWS IAM Database Authentication

A developer wants to connect to Amazon RDS databases using IAM authentication tokens, integrating with AWS credential chain.

```rust
use drasi_lib::identity::AwsIdentityProvider;

// Create AWS IAM identity provider
// Automatically uses AWS credentials from environment (AWS_ACCESS_KEY_ID, etc.)
// or instance profile when running on EC2/ECS
let aws_identity = AwsIdentityProvider::new(
    "mydbuser",                      // IAM database username
    "us-west-2",                     // AWS region
    "mydb.rds.amazonaws.com",        // RDS endpoint
    5432                             // Port
).await?;

// Use with PostgreSQL reaction connecting to RDS
let rds_reaction = PostgresStoredProcReaction::builder("rds-processor")
    .with_hostname("mydb.rds.amazonaws.com")
    .with_database("myapp")
    .with_identity_provider(aws_identity)
    .build()
    .await?;
```

#### Scenario 4: Simple Password Authentication

A developer wants to continue using traditional username/password authentication for local development or non-cloud deployments.

```rust
use drasi_lib::identity::PasswordIdentityProvider;

// Option A: Use the new identity provider pattern (recommended)
let password_identity = PasswordIdentityProvider::new("myuser", "mypassword");

let reaction = PostgresStoredProcReaction::builder("local-reaction")
    .with_hostname("localhost")
    .with_database("mydb")
    .with_identity_provider(password_identity)
    .build()
    .await?;

// Option B: Use legacy API (backward compatible - still works!)
let reaction = PostgresStoredProcReaction::builder("local-reaction")
    .with_hostname("localhost")
    .with_database("mydb")
    .with_user("myuser")
    .with_password("mypassword")
    .build()
    .await?;
```

#### Scenario 5: Mixed Authentication in One Application

A developer has a hybrid deployment with some databases in Azure, some in AWS, and some on-premises.

```rust
use drasi_lib::identity::{
    AzureIdentityProvider, 
    AwsIdentityProvider, 
    PasswordIdentityProvider
};

// Azure for production analytics
let azure_identity = AzureIdentityProvider::new("user@tenant.onmicrosoft.com");

// AWS for data lake
let aws_identity = AwsIdentityProvider::new(
    "datalake_user", "us-east-1", "datalake.rds.amazonaws.com", 5432
).await?;

// Password for on-prem legacy system
let onprem_identity = PasswordIdentityProvider::new("legacy_user", "legacy_pass");

// All use the same builder pattern - seamless!
let azure_reaction = PostgresStoredProcReaction::builder("azure-analytics")
    .with_hostname("analytics.postgres.database.azure.com")
    .with_database("analytics")
    .with_identity_provider(azure_identity)
    .build().await?;

let aws_reaction = PostgresStoredProcReaction::builder("aws-datalake")
    .with_hostname("datalake.rds.amazonaws.com")
    .with_database("datalake")
    .with_identity_provider(aws_identity)
    .build().await?;

let onprem_reaction = PostgresStoredProcReaction::builder("onprem-legacy")
    .with_hostname("legacy-db.internal.company.com")
    .with_database("legacy")
    .with_identity_provider(onprem_identity)
    .build().await?;
```

### Goals

1. **Support Multiple Authentication Methods:**
   - Password-based authentication (current behavior)
   - Azure Managed Identity / Azure AD tokens
   - AWS IAM authentication
   - Extensible architecture for future auth methods

2. **Clean User API:**
   - Define identity provider once, reuse across reactions
   - Hide complexity of token fetching from users
   - Type-safe and compile-time checked

3. **Minimal Developer Burden:**
   - Reaction/source developers shouldn't duplicate auth logic
   - Centralized implementation in shared library
   - Easy to integrate into existing components

4. **Backward Compatibility:**
   - Existing code using `.with_password()` continues to work
   - Optional migration path to new identity providers

### Non-Goals

1. **Token Refresh for Long-Running Connections** - Initial implementation fetches tokens at connection time only; background refresh will be addressed in future work
2. **Credential Caching** - Performance optimization through caching is out of scope for v1
3. **Secret Management Integration** - Direct integration with Azure Key Vault, AWS Secrets Manager, or HashiCorp Vault is not included in this design
4. **OAuth2/OIDC Flows** - Interactive authentication flows requiring user interaction are not supported

## Design requirements

### Requirements

1. **Trait-Based Abstraction** - Define an `IdentityProvider` trait that all authentication providers implement
2. **Feature Flags** - Cloud SDK dependencies (Azure, AWS) must be behind optional Cargo feature flags
3. **Thread Safety** - Identity providers must be `Send + Sync + Clone` for use across async contexts
4. **Error Handling** - Clear error messages for authentication failures
5. **Testability** - Design must support mocking for unit tests without real cloud credentials

### Dependencies

| Dependency | Purpose | Version |
|------------|---------|---------|
| `async-trait` | Async trait support | 0.1+ |
| `azure_identity` | Azure AD authentication (optional) | 0.20+ |
| `azure_core` | Azure SDK core types (optional) | 0.20+ |
| `aws-config` | AWS configuration loading (optional) | 1.0+ |
| `aws-sdk-rds` | RDS IAM token generation (optional) | 1.0+ |

**Component Dependencies:**
- This feature will be implemented in `drasi-core` library
- Integration required for: PostgreSQL Reaction, PostgreSQL Source, MySQL Reaction, HTTP Reaction

### Out of scope

1. **Certificate-based Authentication** - X.509 certificate authentication is not included but the design supports adding it later
2. **Service Principal Authentication** - Azure service principal with client secret/certificate is deferred to future work
3. **Kubernetes Service Account Tokens** - Integration with Kubernetes workload identity is not included in v1

## Design

### High-level design

The design introduces a trait-based identity provider pattern with three key components:

1. **IdentityProvider Trait** - Defines the contract for all authentication providers
2. **Credentials Enum** - Represents the output of authentication (username/password or token)
3. **Concrete Providers** - Implementations for Password, Azure, and AWS authentication

```
┌─────────────────────────────────────────────────────────────────────┐
│                          User Application                            │
├─────────────────────────────────────────────────────────────────────┤
│  let azure_identity = AzureIdentityProvider::default();             │
│  let reaction = PostgresReaction::builder()                         │
│      .with_identity_provider(azure_identity.clone())                │
│      .build().await?;                                               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      drasi-lib/src/identity/                         │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              trait IdentityProvider                          │   │
│  │  async fn get_credentials(&self) -> Result<Credentials>      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           ▲                    ▲                    ▲               │
│           │                    │                    │               │
│  ┌────────┴────────┐ ┌────────┴────────┐ ┌────────┴────────┐      │
│  │PasswordProvider │ │ AzureProvider   │ │  AwsProvider    │      │
│  │ (always avail)  │ │ (feature flag)  │ │ (feature flag)  │      │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Reactions / Sources                                │
├─────────────────────────────────────────────────────────────────────┤
│  PostgreSQL Reaction   │  MySQL Reaction   │  PostgreSQL Source     │
│  ─────────────────     │  ─────────────    │  ──────────────────    │
│  credentials =         │  credentials =    │  credentials =         │
│    provider            │    provider       │    provider            │
│    .get_credentials()  │    .get_credentials()   │    .get_credentials()  │
│    .await?;            │    .await?;       │    .await?;            │
└─────────────────────────────────────────────────────────────────────┘
```

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              User Application                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │ azure_identity  │    │ password_ident  │    │  aws_identity   │          │
│  │ = AzureProvider │    │ = PasswordProv  │    │ = AwsProvider   │          │
│  │   ::default()   │    │ ::new(u,p)      │    │ ::new(...)      │          │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘          │
│           │                      │                      │                    │
│           └──────────────────────┼──────────────────────┘                    │
│                                  │                                           │
│                                  ▼                                           │
│           ┌──────────────────────────────────────────────┐                  │
│           │          Arc<dyn IdentityProvider>           │                  │
│           └──────────────────────┬───────────────────────┘                  │
│                                  │                                           │
└──────────────────────────────────┼───────────────────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ PostgreSQL        │    │ PostgreSQL        │    │ MySQL             │
│ Reaction          │    │ Source            │    │ Reaction          │
│ ─────────────     │    │ ─────────────     │    │ ─────────────     │
│ config:           │    │ config:           │    │ config:           │
│   identity_prov   │    │   identity_prov   │    │   identity_prov   │
│                   │    │                   │    │                   │
│ executor.rs:      │    │ executor.rs:      │    │ executor.rs:      │
│   get_credentials │    │   get_credentials │    │   get_credentials │
│   → connect()     │    │   → connect()     │    │   → connect()     │
└─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
          │                        │                        │
          ▼                        ▼                        ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ Azure Database    │    │ Azure Database    │    │ Amazon RDS        │
│ for PostgreSQL    │    │ for PostgreSQL    │    │ MySQL             │
└───────────────────┘    └───────────────────┘    └───────────────────┘
```

### Detail design

#### File Structure

```
lib/src/
├── identity/
│   ├── mod.rs          # Trait definition + Credentials enum + re-exports
│   ├── password.rs     # PasswordIdentityProvider
│   ├── azure.rs        # AzureIdentityProvider (behind feature flag)
│   └── aws.rs          # AwsIdentityProvider (behind feature flag)
├── lib.rs              # Add: pub mod identity;
└── ...

components/reactions/storedproc-postgres/
├── src/
│   ├── config.rs       # Add identity_provider: Option<Arc<dyn IdentityProvider>>
│   ├── executor.rs     # Use identity provider for credentials
│   └── reaction.rs     # Add with_identity_provider() to builder
└── Cargo.toml          # Add optional dep on lib identity features
```

#### Core Trait Definition

**lib/src/identity/mod.rs:**

```rust
use anyhow::Result;
use async_trait::async_trait;
use std::sync::Arc;

/// Trait for identity providers that supply authentication credentials.
///
/// Implementations must be thread-safe (`Send + Sync`). To use them as
/// cloneable trait objects (e.g. `Box<dyn IdentityProvider>` or
/// `Arc<dyn IdentityProvider>`), concrete types should also implement
/// `Clone`; cloning of the trait object is then provided via a helper trait
/// and the `Clone` impl for `Box<dyn IdentityProvider + Send + Sync>`.
#[async_trait]
pub trait IdentityProvider: Send + Sync {
    /// Fetch credentials for authentication.
    ///
    /// This method may perform network calls to retrieve tokens
    /// and should be called close to the point of use.
    async fn get_credentials(&self) -> Result<Credentials>;
}

/// Helper trait to enable cloning of `Box<dyn IdentityProvider>` trait objects.
///
/// This is implemented automatically for any `'static` type that implements
/// both `IdentityProvider` and `Clone`.
pub trait IdentityProviderClone {
    fn clone_box(&self) -> Box<dyn IdentityProvider + Send + Sync>;
}

impl<T> IdentityProviderClone for T
where
    T: 'static + IdentityProvider + Clone,
{
    fn clone_box(&self) -> Box<dyn IdentityProvider + Send + Sync> {
        Box::new(self.clone())
    }
}

impl Clone for Box<dyn IdentityProvider + Send + Sync> {
    fn clone(&self) -> Self {
        // Delegate to the `IdentityProviderClone` implementation for the
        // underlying concrete type.
        self.as_ref().clone_box()
    }
}

/// Shared, cloneable handle to an identity provider implementation.
///
/// In most places, Drasi components should depend on this alias rather than
/// holding a concrete implementation type directly.
pub type SharedIdentityProvider = Arc<dyn IdentityProvider + Send + Sync>;
/// Credentials returned by an identity provider.
#[derive(Debug, Clone)]
pub enum Credentials {
    /// Traditional username and password authentication.
    UsernamePassword {
        username: String,
        password: String,
    },
    /// Token-based authentication (Azure AD, AWS IAM, etc.).
    Token {
        username: String,
        token: String,
    },
}

impl Credentials {
    /// Extract username and password/token for connection string building.
    pub fn into_auth_pair(self) -> (String, String) {
        match self {
            Credentials::UsernamePassword { username, password } => (username, password),
            Credentials::Token { username, token } => (username, token),
        }
    }
}

// Re-export providers
mod password;
pub use password::PasswordIdentityProvider;

#[cfg(feature = "azure-identity")]
mod azure;
#[cfg(feature = "azure-identity")]
pub use azure::AzureIdentityProvider;

#[cfg(feature = "aws-identity")]
mod aws;
#[cfg(feature = "aws-identity")]
pub use aws::AwsIdentityProvider;
```

#### Password Identity Provider

**lib/src/identity/password.rs:**

```rust
use super::{Credentials, IdentityProvider};
use anyhow::Result;
use async_trait::async_trait;

/// Identity provider for traditional username/password authentication.
#[derive(Clone)]
pub struct PasswordIdentityProvider {
    username: String,
    password: String,
}

impl PasswordIdentityProvider {
    /// Create a new password identity provider.
    pub fn new(username: impl Into<String>, password: impl Into<String>) -> Self {
        Self {
            username: username.into(),
            password: password.into(),
        }
    }
}

#[async_trait]
impl IdentityProvider for PasswordIdentityProvider {
    async fn get_credentials(&self) -> Result<Credentials> {
        Ok(Credentials::UsernamePassword {
            username: self.username.clone(),
            password: self.password.clone(),
        })
    }
    
    fn clone_box(&self) -> Box<dyn IdentityProvider> {
        Box::new(self.clone())
    }
}
```

#### Azure Identity Provider

**lib/src/identity/azure.rs:**

```rust
use super::{Credentials, IdentityProvider};
use anyhow::{anyhow, Result};
use async_trait::async_trait;
use azure_identity::{DefaultAzureCredential, TokenCredential};
use std::sync::Arc;

/// Default scope for Azure Database for PostgreSQL/MySQL.
const AZURE_DATABASE_SCOPE: &str = "https://ossrdbms-aad.database.windows.net/.default";

/// Identity provider for Azure AD authentication.
/// 
/// Uses Azure's DefaultAzureCredential which tries multiple auth methods:
/// - Environment variables
/// - Managed Identity
/// - Azure CLI
/// - Visual Studio Code
#[derive(Clone)]
pub struct AzureIdentityProvider {
    credential: Arc<dyn TokenCredential>,
    username: String,
    scope: String,
}

impl AzureIdentityProvider {
    /// Create provider using DefaultAzureCredential.
    /// 
    /// The username should be in the format `user@tenant.onmicrosoft.com`
    /// for Azure AD authentication.
    pub fn new(username: impl Into<String>) -> Self {
        Self {
            credential: Arc::new(DefaultAzureCredential::default()),
            username: username.into(),
            scope: AZURE_DATABASE_SCOPE.to_string(),
        }
    }
    
    /// Create provider with a specific managed identity client ID.
    pub fn with_managed_identity(
        username: impl Into<String>,
        client_id: impl Into<String>,
    ) -> Self {
        use azure_identity::ManagedIdentityCredential;
        
        let credential = ManagedIdentityCredential::default()
            .with_client_id(client_id.into());
        
        Self {
            credential: Arc::new(credential),
            username: username.into(),
            scope: AZURE_DATABASE_SCOPE.to_string(),
        }
    }
    
    /// Set a custom scope for token acquisition.
    pub fn with_scope(mut self, scope: impl Into<String>) -> Self {
        self.scope = scope.into();
        self
    }
}

#[async_trait]
impl IdentityProvider for AzureIdentityProvider {
    async fn get_credentials(&self) -> Result<Credentials> {
        let token_response = self.credential
            .get_token(&[&self.scope])
            .await
            .map_err(|e| anyhow!("Failed to get Azure AD token: {}", e))?;
        
        Ok(Credentials::Token {
            username: self.username.clone(),
            token: token_response.token.secret().to_string(),
        })
    }
    
    fn clone_box(&self) -> Box<dyn IdentityProvider> {
        Box::new(self.clone())
    }
}
```

#### AWS Identity Provider

**lib/src/identity/aws.rs:**

```rust
use super::{Credentials, IdentityProvider};
use anyhow::{anyhow, Result};
use async_trait::async_trait;
use aws_config::SdkConfig;
use std::sync::Arc;

/// Identity provider for AWS IAM database authentication.
#[derive(Clone)]
pub struct AwsIdentityProvider {
    config: Arc<SdkConfig>,
    username: String,
    hostname: String,
    port: u16,
    region: String,
}

impl AwsIdentityProvider {
    /// Create a new AWS IAM identity provider.
    /// 
    /// Loads AWS configuration from the environment (credentials, region, etc.).
    pub async fn new(
        username: impl Into<String>,
        region: impl Into<String>,
        hostname: impl Into<String>,
        port: u16,
    ) -> Result<Self> {
        let config = aws_config::load_from_env().await;
        
        Ok(Self {
            config: Arc::new(config),
            username: username.into(),
            hostname: hostname.into(),
            port,
            region: region.into(),
        })
    }
}

#[async_trait]
impl IdentityProvider for AwsIdentityProvider {
    async fn get_credentials(&self) -> Result<Credentials> {
        let rds_client = aws_sdk_rds::Client::new(&self.config);
        
        let token = rds_client
            .generate_db_auth_token()
            .hostname(&self.hostname)
            .port(self.port as i32)
            .username(&self.username)
            .region(&self.region)
            .send()
            .await
            .map_err(|e| anyhow!("Failed to generate AWS IAM token: {}", e))?;
        
        Ok(Credentials::Token {
            username: self.username.clone(),
            token,
        })
    }
    
    fn clone_box(&self) -> Box<dyn IdentityProvider> {
        Box::new(self.clone())
    }
}
```

#### Reaction Integration

**Executor changes (minimal code for reaction developers of new reactions/sources):**

```rust
// In components/reactions/storedproc-postgres/src/executor.rs
pub async fn new(config: &PostgresStoredProcReactionConfig) -> Result<Self> {
    // Fetch credentials using identity provider
    let credentials = config.identity_provider.get_credentials().await?;
    let (username, password_or_token) = credentials.into_auth_pair();
    
    let ssl_mode = if config.ssl { "require" } else { "disable" };
    let connection_string = format!(
        "host={} port={} user={} password={} dbname={} sslmode={}",
        config.hostname, 
        config.port.unwrap_or(5432), 
        username, 
        password_or_token, 
        config.database, 
        ssl_mode
    );
    
    let (client, connection) = tokio_postgres::connect(&connection_string, NoTls).await?;
    // ... rest unchanged
}
```

#### Backward Compatibility

To maintain backward compatibility, the builder pattern supports both approaches:

```rust
impl PostgresStoredProcReactionBuilder {
    /// New method: use an identity provider
    pub fn with_identity_provider(
        mut self, 
        provider: impl IdentityProvider + 'static
    ) -> Self {
        self.config.identity_provider = Box::new(provider);
        self
    }
    
    /// Legacy method: auto-creates PasswordIdentityProvider
    pub fn with_user(mut self, user: impl Into<String>) -> Self {
        self.user = Some(user.into());
        self.maybe_create_password_provider();
        self
    }
    
    /// Legacy method: auto-creates PasswordIdentityProvider
    pub fn with_password(mut self, password: impl Into<String>) -> Self {
        self.password = Some(password.into());
        self.maybe_create_password_provider();
        self
    }
    
    fn maybe_create_password_provider(&mut self) {
        if let (Some(user), Some(password)) = (&self.user, &self.password) {
            self.config.identity_provider = Box::new(
                PasswordIdentityProvider::new(user.clone(), password.clone())
            );
        }
    }
}
```

#### Advantages of this design

1. **Centralized Auth Logic** - Token fetching code written once in `lib/src/identity/`, reused by all reactions/sources
2. **Minimal Reaction Developer Effort** - Only 2-3 lines of code to integrate authentication
3. **Type-Safe API** - Compile-time checking ensures valid provider usage
4. **Extensible** - Add new auth types without modifying existing reactions
5. **Reusable** - Same identity provider instance can be shared across multiple components
6. **Feature Flags** - Cloud SDK dependencies are optional, reducing binary size for simple deployments

#### Disadvantages

1. **Trait Object Overhead** - Requires `Box<dyn IdentityProvider>` which has minor runtime cost
2. **Upfront Complexity** - More initial setup compared to simple username/password
3. **No Token Refresh** - Tokens fetched once at connection time; long-running connections may need reconnection

### API Design

#### User-Facing API

**Creating Identity Providers:**

```rust
// Password authentication
let password_identity = PasswordIdentityProvider::new("myuser", "mypassword");

// Azure AD with DefaultAzureCredential
let azure_identity = AzureIdentityProvider::new("user@tenant.onmicrosoft.com");

// Azure AD with specific managed identity
let azure_identity = AzureIdentityProvider::with_managed_identity(
    "user@tenant.onmicrosoft.com",
    "03bbedd2-cce5-45ab-9414-1c1cb82361f0"  // client_id
);

// AWS IAM authentication
let aws_identity = AwsIdentityProvider::new(
    "myuser",
    "us-west-2",
    "mydb.rds.amazonaws.com",
    5432
).await?;
```

**Using with Reactions:**

```rust
let reaction = PostgresStoredProcReaction::builder("reaction1")
    .with_hostname("db1.postgres.database.azure.com")
    .with_database("mydb")
    .with_identity_provider(azure_identity.clone())
    .build()
    .await?;
```

**Backward Compatible (Legacy) API:**

```rust
// Still works - creates PasswordIdentityProvider internally
let reaction = PostgresStoredProcReaction::builder("reaction1")
    .with_hostname("localhost")
    .with_database("mydb")
    .with_user("myuser")
    .with_password("mypassword")
    .build()
    .await?;
```

#### Cargo Features

```toml
# lib/Cargo.toml
[features]
default = []
azure-identity = ["dep:azure_identity", "dep:azure_core"]
aws-identity = ["dep:aws-config", "dep:aws-sdk-rds"]
all-identity = ["azure-identity", "aws-identity"]
```

### Alternatives Considered

#### Alternative A: No Abstraction (Status Quo + Manual Tokens)

Users manually fetch Azure/AWS tokens and pass them as the password field.

```rust
// User must manually fetch token
let credential = DefaultAzureCredential::default();
let token = credential.get_token("...").await?.token.secret().to_string();

let reaction = PostgresReaction::builder()
    .with_password(token)  // Confusing - it's not a password!
    .build().await?;
```

**Rejected because:**
- User burden too high
- Confusing API semantics
- No token reuse across reactions
- Defeats the purpose of simplifying cloud authentication

#### Alternative B: Enum-Based Factory (Pure Data Holder)

User provides configuration data; each reaction implements token fetching.

```rust
pub enum Identity {
    Password(PasswordIdentity),
    Azure(AzureIdentity),
    Aws(AwsIdentity),
}
```

**Rejected because:**
- Every reaction developer must duplicate Azure/AWS token-fetching code
- Adding new auth type requires updating ALL reactions
- Code duplication nightmare (N reactions × M auth types)

#### Alternative C: Hybrid Factory Pattern

Separate identity provider (factory) from credential fetcher (worker).

**Rejected because:**
- Over-engineered for the use case
- Two traits instead of one with no clear benefit
- Extra indirection without added value

## Security

### Authentication Flow

1. **Credential Isolation** - Credentials are never stored in configuration; only provider references are passed
2. **Token Scope** - Azure tokens are scoped to `https://ossrdbms-aad.database.windows.net/.default`
3. **No Credential Logging** - The `Credentials` enum intentionally doesn't implement `Display` to prevent accidental logging

### Threat Mitigations

| Threat | Mitigation |
|--------|------------|
| Credential exposure in logs | `Credentials` doesn't implement `Display`; passwords/tokens are not logged |
| Token leakage | Tokens are short-lived; fetched close to point of use |
| Hardcoded secrets | Identity providers encourage configuration injection over hardcoding |

### Feature Flag Security

Cloud SDKs are behind feature flags to:
- Reduce attack surface for deployments not using cloud auth
- Minimize dependencies that could have vulnerabilities

## Compatibility impact

### Breaking Changes

**None.** The design is fully backward compatible:
- Existing `.with_user()` and `.with_password()` methods continue to work
- No changes required for existing applications

### Migration Path

Existing code can optionally migrate to the new API:

```rust
// Before (still works)
.with_user("myuser")
.with_password("mypassword")

// After (recommended for cloud deployments)
.with_identity_provider(AzureIdentityProvider::new("user@tenant.onmicrosoft.com"))
```

## Supportability

### Telemetry

| Metric/Log | Description |
|------------|-------------|
| `drasi.identity.get_credentials.duration` | Time to fetch credentials |
| `drasi.identity.get_credentials.error` | Authentication failures with error type |
| `drasi.identity.provider.type` | Type of identity provider used (password/azure/aws) |

### Verification

#### Unit Tests

```rust
#[tokio::test]
async fn test_password_provider() {
    let provider = PasswordIdentityProvider::new("testuser", "testpass");
    let creds = provider.get_credentials().await.unwrap();
    
    match creds {
        Credentials::UsernamePassword { username, password } => {
            assert_eq!(username, "testuser");
            assert_eq!(password, "testpass");
        }
        _ => panic!("Expected UsernamePassword credentials"),
    }
}

#[tokio::test]
async fn test_provider_clone() {
    let provider1 = PasswordIdentityProvider::new("user", "pass");
    let provider2 = provider1.clone();
    
    let creds = provider2.get_credentials().await.unwrap();
    assert!(matches!(creds, Credentials::UsernamePassword { .. }));
}
```

#### Integration Tests

- Test PostgreSQL reaction with `PasswordIdentityProvider` against local database
- Test PostgreSQL reaction with `AzureIdentityProvider` against Azure Database for PostgreSQL (requires Azure credentials)
- Test reusing same provider across multiple reactions
- Test backward compatibility with legacy `.with_password()` API


## References

- [Azure Identity SDK for Rust](https://github.com/Azure/azure-sdk-for-rust/tree/main/sdk/identity)
- [AWS SDK for Rust](https://github.com/awslabs/aws-sdk-rust)
- [PostgreSQL Azure AD Authentication](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-configure-sign-in-azure-ad-authentication)
- [PostgreSQL AWS IAM Authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html)
- [drasi-project/drasi-core Repository](https://github.com/drasi-project/drasi-core)
