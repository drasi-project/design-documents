# Drasi Ingress Support

* Project Drasi - September 11, 2025 - ruokunniu (@ruokunniu)

## Overview

This design document proposes adding ingress support to Drasi to enable external access to Reactions with endpoint configuration set to `external`. The feature includes two main components: a new CLI command to install Contour ingress controller and modifications to the control plane to automatically create Kubernetes ingress resources for externally exposed Reactions.

Currently, Drasi Reactions can only be accessed within the Kubernetes cluster. This limitation prevents external systems from consuming Reaction endpoints directly, requiring additional manual configuration or workarounds. By adding native ingress support, users will be able to expose Reaction endpoints externally through a standardized, automated approach.

## Terms and definitions

| Term | Definition |
|------|------------|
| Ingress Controller | A Kubernetes component that manages external access to services in a cluster, typically HTTP/HTTPS |
| Contour | An open-source Kubernetes ingress controller that provides load balancing and HTTP(S) reverse proxy services |
| External Endpoint | A Reaction configuration where the endpoint is set to `external`, indicating it should be accessible from outside the cluster |
| HTTPProxy | Contour's custom resource for defining ingress routing rules |

## Objectives

### User scenarios

**Scenario 1: Data Engineer exposing analytics endpoint**
As a data engineer, I want to expose my Drasi Reaction that serves analytics data so that external dashboards and applications can consume real-time query results via HTTP endpoints.

**Scenario 2: Integration with external systems**
As a system integrator, I want to configure my Drasi Reaction to be accessible from external microservices so that I can build event-driven architectures that span multiple systems.

**Scenario 3: Development and testing**
As a developer, I want to easily access my Reaction endpoints from my local development environment for testing and debugging purposes without complex port-forwarding or proxy configurations.

### Goals

- Enable automatic ingress creation for Reactions with `endpoint: external` configuration
- Provide a standardized CLI command to install and configure Contour ingress controller
- Ensure seamless integration with existing Drasi control plane and resource management
- Support HTTPS termination and basic routing capabilities
- Maintain compatibility with existing Reaction configurations

### Non-Goals

- Support for multiple ingress controllers (initially focusing on Contour only)
- Advanced ingress features like rate limiting, authentication, or custom middleware
- Automatic SSL certificate management (users will need to provide certificates)
- Load balancing across multiple Reaction instances

## Design requirements

### Requirements

- **R1**: CLI must provide a command to install Contour ingress controller in the cluster
- **R2**: Control plane must detect Reactions with `endpoint: external` and create corresponding ingress resources
- **R3**: Ingress resources must be automatically cleaned up when Reactions are deleted
- **R4**: Solution must integrate with existing Drasi RBAC and security model
- **R5**: Configuration must be extensible for future ingress controller support
- **R6**: Must not impact existing Reactions without `endpoint: external`

### Dependencies

- **Drasi CLI**: Requires updates to add new ingress installation command
- **Control Plane**: Kubernetes Resource Provider needs modification to handle ingress resources
- **Management API**: May need updates to support ingress configuration options
- **Kubernetes RBAC**: Need permissions to create/manage ingress resources and Contour CRDs

### Out of scope

- Support for ingress controllers other than Contour in the initial implementation
- Automatic domain name management or DNS configuration
- Integration with external certificate authorities
- Advanced traffic management features (these can be added in future iterations)

## Design

### High-level design

The ingress support feature consists of two main components:

1. **CLI Enhancement**: A new `drasi ingress install` command that deploys Contour ingress controller to the cluster with appropriate configuration for Drasi.

2. **Control Plane Enhancement**: Modifications to the Kubernetes Resource Provider to automatically create HTTPProxy resources (Contour's ingress CRD) when Reactions are configured with `endpoint: external`.

The workflow will be:
1. User installs Contour using `drasi ingress install contour`
2. User creates or updates a Reaction with `endpoint: external`
3. Control plane detects the external endpoint configuration
4. Control plane creates an HTTPProxy resource pointing to the Reaction service
5. External traffic can now reach the Reaction through the ingress controller

### Architecture Diagram

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   External      │    │    Contour      │    │    Reaction     │
│   Client        │───▶│    Ingress      │───▶│    Service      │
│                 │    │   Controller    │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                │                        │
                       ┌─────────────────┐    ┌─────────────────┐
                       │   HTTPProxy     │    │   Reaction      │
                       │   Resource      │    │     Pod         │
                       │                 │    │                 │
                       └─────────────────┘    └─────────────────┘
                                │                        │
                                │                        │
                       ┌─────────────────────────────────────────┐
                       │         Drasi Control Plane              │
                       │  ┌─────────────────────────────────────┐ │
                       │  │   Kubernetes Resource Provider      │ │
                       │  └─────────────────────────────────────┘ │
                       └─────────────────────────────────────────┘
```

### Detail design

#### 1. CLI Enhancement

**New Command Structure:**
```bash
drasi ingress install contour [--namespace=<namespace>] [--config-file=<path>]
drasi ingress uninstall contour [--namespace=<namespace>]
drasi ingress status
```

**Implementation Details:**
- Add new `ingress` subcommand to Drasi CLI
- `install contour` will deploy Contour using its official Helm chart or YAML manifests
- Default namespace will be `projectcontour` (Contour's standard namespace)
- Command will verify cluster permissions before installation
- Include validation to ensure Contour is properly installed and healthy

**Advantages:**
- Provides standardized installation process
- Ensures compatible Contour configuration for Drasi
- Simplifies user onboarding

**Disadvantages:**
- Adds dependency management complexity to CLI
- May conflict with existing Contour installations

#### 2. Control Plane Enhancement

**Kubernetes Resource Provider Changes:**
- Modify the Reaction resource reconciliation logic to detect `endpoint: external`
- When external endpoint is detected, create HTTPProxy resource alongside existing Reaction resources
- Implement cleanup logic to remove HTTPProxy when Reaction is deleted or endpoint changed

**HTTPProxy Resource Template:**
```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: reaction-<reaction-name>
  namespace: <reaction-namespace>
spec:
  virtualhost:
    fqdn: <reaction-name>.<cluster-domain>
  routes:
  - conditions:
    - prefix: /
    services:
    - name: <reaction-service-name>
      port: <reaction-service-port>
```

**Configuration Options:**
- Support for custom domain configuration via Reaction annotations
- Configurable path prefixes and routing rules
- TLS configuration support for HTTPS endpoints

**Advantages:**
- Automatic lifecycle management of ingress resources
- Consistent with existing Drasi resource management patterns
- No manual configuration required from users

**Disadvantages:**
- Increases complexity of control plane
- Tight coupling with Contour-specific resources

### API Design

#### CLI Commands

**Installation Command:**
```bash
drasi ingress install contour [options]

Options:
  --namespace string     Namespace to install Contour (default "projectcontour")
  --config-file string   Path to custom Contour configuration file
  --dry-run             Show what would be installed without executing
  --wait                Wait for installation to complete
```

**Status Command:**
```bash
drasi ingress status

# Output example:
Ingress Controller: Contour
Status: Ready
Namespace: projectcontour
Version: 1.28.0
Exposed Reactions: 3
```

#### Management API Extensions

**Reaction Resource Schema Update:**
```json
{
  "endpoint": {
    "type": "string",
    "enum": ["internal", "external"],
    "description": "Endpoint accessibility configuration"
  },
  "ingress": {
    "type": "object",
    "properties": {
      "domain": {
        "type": "string",
        "description": "Custom domain for external access"
      },
      "tls": {
        "type": "object",
        "properties": {
          "enabled": {"type": "boolean"},
          "secretName": {"type": "string"}
        }
      }
    }
  }
}
```

#### Kubernetes Resource Provider API

**New Go Interface:**
```go
type IngressManager interface {
    CreateIngress(ctx context.Context, reaction *v1alpha1.Reaction) error
    UpdateIngress(ctx context.Context, reaction *v1alpha1.Reaction) error
    DeleteIngress(ctx context.Context, reactionName, namespace string) error
    GetIngressStatus(ctx context.Context, reactionName, namespace string) (*IngressStatus, error)
}

type IngressStatus struct {
    Ready     bool
    Endpoints []string
    Message   string
}
```

### Alternatives Considered

#### Alternative 1: Support Multiple Ingress Controllers
**Description:** Implement a generic ingress interface supporting multiple controllers (Nginx, Traefik, etc.)

**Pros:** 
- Greater flexibility for users with existing ingress controllers
- Future-proof architecture

**Cons:** 
- Significantly increased complexity
- Different feature sets across controllers
- More testing and maintenance overhead

**Justification for rejection:** The additional complexity is not justified for the initial implementation. We can add support for additional controllers in future iterations based on user demand.

#### Alternative 2: Use Standard Kubernetes Ingress Resources
**Description:** Use built-in Kubernetes Ingress resources instead of Contour's HTTPProxy CRDs

**Pros:**
- Controller-agnostic approach
- Uses standard Kubernetes APIs

**Cons:**
- Limited functionality compared to HTTPProxy
- May require controller-specific annotations
- Less powerful routing capabilities

**Justification for rejection:** HTTPProxy provides better feature set and is specifically designed for Contour, giving us more predictable behavior and advanced routing options.

#### Alternative 3: Manual Configuration Approach
**Description:** Provide documentation for users to manually configure ingress resources

**Pros:**
- No code changes required
- Maximum flexibility for users

**Cons:**
- Poor user experience
- Increased support burden
- Error-prone manual configuration

**Justification for rejection:** This doesn't align with Drasi's goal of providing automated, easy-to-use infrastructure management.

## Security

### Threat Analysis

1. **Exposure of Internal Services**: External ingress creates new attack surface
   - **Mitigation**: Only create ingress for explicitly configured external endpoints
   - **Mitigation**: Support TLS termination and encourage HTTPS usage

2. **Unauthorized Access to Reaction Data**: External endpoints may be accessed without proper authentication
   - **Mitigation**: Document requirement for Reaction-level authentication
   - **Mitigation**: Support for future integration with authentication providers

3. **DNS Spoofing/Domain Hijacking**: Custom domains may be vulnerable to DNS attacks
   - **Mitigation**: Document DNS security best practices
   - **Mitigation**: Support certificate validation

### Security Measures

- HTTPProxy resources will be created in the same namespace as Reactions (namespace isolation)
- TLS configuration support for encrypted traffic
- Integration with Kubernetes RBAC for ingress resource management
- Audit logging for ingress resource creation/deletion

## Compatibility impact

### Breaking Changes
- None expected - this is an additive feature

### Behavioral Changes
- Reactions with `endpoint: external` will automatically get ingress resources created
- New CLI commands added (no impact on existing commands)

### Version Compatibility
- Requires Kubernetes 1.20+ for HTTPProxy CRD support
- Compatible with Contour 1.20+
- Backward compatible with existing Reaction configurations

## Supportability

### Telemetry
- Log ingress resource creation/deletion events
- Metrics for number of external endpoints exposed
- Health checks for ingress controller connectivity
- Error metrics for failed ingress operations

### Verification
- Unit tests for CLI ingress commands
- Unit tests for control plane ingress logic
- Integration tests for end-to-end ingress functionality
- E2E tests for external endpoint accessibility
- Load testing for ingress performance impact

## Development Plan

### Phase 1: CLI Foundation (2 weeks)
- Implement `drasi ingress install contour` command
- Add Contour deployment logic and validation
- Unit tests for CLI commands
- Documentation updates

### Phase 2: Control Plane Integration (3 weeks)
- Implement HTTPProxy creation in Kubernetes Resource Provider
- Add Reaction endpoint detection logic
- Implement ingress lifecycle management (create/update/delete)
- Unit tests for control plane changes

### Phase 3: Integration and Testing (2 weeks)
- End-to-end integration tests
- Performance testing and optimization
- Documentation and examples
- User acceptance testing

### Phase 4: Polish and Documentation (1 week)
- CLI help text and error messaging
- Troubleshooting guides
- Example configurations and tutorials

**Total Estimated Effort:** 8 weeks

## Open issues

1. **Q: How should we handle domain name configuration for external endpoints?**
   A: Initial implementation will use a default pattern (reaction-name.cluster-domain). Future versions can support custom domains through Reaction annotations.

2. **Q: Should we support automatic certificate management?**
   A: Not in initial implementation - users will need to provide their own certificates. This can be added in future iterations with cert-manager integration.

3. **Q: How do we handle conflicts with existing Contour installations?**
   A: CLI should detect existing installations and either skip installation or provide options to configure Drasi to use existing Contour instance.

4. **Q: What happens if Contour is uninstalled while Reactions have external endpoints?**
   A: Need to implement graceful degradation and clear error messages. Consider adding validation webhook.

## References

- [Contour Documentation](https://projectcontour.io/docs/)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [HTTPProxy API Reference](https://projectcontour.io/docs/main/config/api/#projectcontour.io/v1.HTTPProxy)
- [Drasi Platform Architecture](https://github.com/drasi-project/drasi-platform)
