# Developer Workflow

This section provides guidance for development teams on how to integrate with and work effectively with Azure API Management.

## Table of Contents

- [Developer Integration Guide](developer-integration.md) - How developers work with APIM, deployment strategies, and workflow patterns
- [API Specification Management](api-specification-management.md) - Creating and maintaining OpenAPI specifications
- [Configuration Management](configuration-management.md) - Managing APIM configuration files and environment settings
- [Service Integration Patterns](service-integration-patterns.md) - Connecting backend services to APIM

## Overview

Azure API Management changes how developers publish and manage APIs, but the goal is to minimize disruption to existing workflows while adding powerful capabilities for security, monitoring, and governance.

## Developer Experience Goals

The APIM implementation is designed with these developer experience principles:

- **Minimal Code Changes** - Developers continue to build services as before
- **Self-Service Publishing** - Teams can publish APIs without manual intervention (in mature workflows)
- **Automated Documentation** - API docs are generated from specifications
- **Integrated Testing** - Built-in test console and Developer Portal try-it functionality
- **Clear Workflows** - Defined paths from development to production

## Three Deployment Strategies

Organizations typically evolve through three phases of APIM adoption:

### 1. Short-Term: Centralized Manual Deployments

**Characteristics:**
- Single infrastructure repository manages all APIs
- Developers submit API specs and configurations
- Central team reviews and manually triggers deployments
- Authentication remains at backend services
- Focus on learning and proof-of-concept

**Best For:**
- Initial APIM implementation
- Small number of APIs
- Teams learning APIM capabilities

### 2. Medium-Term: Service-Managed Specifications

**Characteristics:**
- API specifications move to individual service repositories
- Infrastructure repository references service spec locations
- Manual deployment approvals continue
- Pilot OAuth authentication at gateway layer
- Teams gain ownership of their API definitions

**Best For:**
- Growing API catalogs
- Organizations building DevOps maturity
- Transition from centralized to distributed ownership

### 3. Long-Term: Automated CI/CD

**Characteristics:**
- API specifications auto-generated from service code
- Automated deployments to dev/QA environments
- Manual approval gates for production
- Full gateway-level authentication
- Streamlined end-to-end workflow

**Best For:**
- Mature DevOps organizations
- Large API catalogs
- Teams with established CI/CD practices

## What Developers Need to Provide

Regardless of deployment strategy, developers need to create:

### 1. OpenAPI Specification

A standardized API definition file describing:
- Endpoints and operations
- Request/response schemas
- Authentication requirements
- Parameter definitions

**Example:**
```yaml
openapi: 3.0.1
info:
  title: Products API
  version: 1.0.0
paths:
  /products:
    get:
      summary: List all products
      responses:
        '200':
          description: Success
```

### 2. APIM Configuration File

YAML or JSON file defining:
- Backend service URL
- Policies (rate limits, authentication, transformation)
- Product associations
- Versioning scheme

**Example:**
```yaml
api:
  name: products-api
  displayName: Products API
  path: products
  serviceUrl: https://backend.example.com
  protocols:
    - https
policies:
  rateLimit: 100
  quota: 10000
products:
  - starter
  - premium
```

## Developer Benefits

### Before APIM
- Custom authentication in each service
- Manual documentation maintenance
- Service-specific logging and monitoring
- No centralized rate limiting
- Direct backend exposure

### With APIM
- Centralized authentication and authorization
- Auto-generated documentation from specs
- Unified monitoring and analytics
- Policy-based rate limiting and quotas
- Gateway protection for backends

## Integration Workflow

### Typical Development Flow

1. **Develop Service** - Build and test your backend service locally
2. **Create OpenAPI Spec** - Define your API contract (can be auto-generated)
3. **Create APIM Config** - Specify policies and settings
4. **Submit for Deployment** - PR to infrastructure repo or automated pipeline
5. **Review & Deploy** - Approval and deployment to APIM
6. **Test in Portal** - Validate using Developer Portal test console
7. **Monitor Usage** - Track API performance and consumption

### Local Development

Developers can:
- Continue using existing local development environments
- Test services directly without APIM during development
- Use mock API servers for frontend development
- Validate OpenAPI specs with CLI tools before submission

## Migration Considerations

### Adding APIM to Existing Services

When migrating existing APIs to APIM:

1. **Assess Current State** - Document existing authentication, rate limits, and contracts
2. **Create Specifications** - Generate OpenAPI specs from existing APIs
3. **Define Policies** - Map current behaviors to APIM policies
4. **Plan Cutover** - Coordinate DNS/routing changes for traffic migration
5. **Monitor Transition** - Compare metrics before and after migration

### Backward Compatibility

- Maintain existing endpoint paths when possible
- Use APIM rewrite policies to adapt requests if needed
- Consider running APIM in parallel initially
- Communicate changes clearly to API consumers

## Common Developer Tasks

| Task | How to Accomplish |
|------|-------------------|
| **Add new API** | Create OpenAPI spec + APIM config, submit to infrastructure repo |
| **Update existing API** | Use revisions for non-breaking changes, versions for breaking changes |
| **Change backend URL** | Update APIM configuration, redeploy |
| **Add authentication** | Configure OAuth policy in APIM config |
| **Apply rate limiting** | Add rate-limit policy to APIM config |
| **View API analytics** | Access Azure Portal or Application Insights dashboards |

## Getting Started

For detailed guidance on specific topics, see:

- [Developer Integration Guide](developer-integration.md) - Comprehensive workflow documentation
- [API Specification Management](api-specification-management.md) - OpenAPI best practices
- [Configuration Management](configuration-management.md) - APIM configuration patterns
- [Service Integration Patterns](service-integration-patterns.md) - Backend connection strategies

## Related Topics

- **Azure APIM** - See [Azure APIM](../azure-apim/) for core concepts and capabilities
- **DevOps** - See [DevOps](../dev-ops/) for CI/CD automation patterns
- **Developer Portal** - See [Developer Portal](../developer-portal/) for consumer experience
