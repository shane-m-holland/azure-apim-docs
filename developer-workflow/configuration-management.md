# Configuration Management

## Overview

APIM configuration files define policies, settings, and behaviors for APIs deployed to Azure API Management. This guide covers configuration structure, best practices, and environment management.

## Configuration File Structure

### Basic APIM Configuration

```yaml
api:
  name: products-api
  displayName: Products API
  description: Product catalog management API
  path: products
  serviceUrl: https://products-backend.example.com
  protocols:
    - https
  subscriptionRequired: true

policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad
        validateClaims: true
    - rateLimit:
        calls: 1000
        renewalPeriod: 60
    - cors:
        allowedOrigins:
          - https://app.example.com
          - https://app-dev.example.com
        allowedMethods:
          - GET
          - POST
          - PUT
          - DELETE
        allowedHeaders:
          - Content-Type
          - Authorization

  outbound:
    - setHeader:
        name: X-API-Version
        value: "@(context.Api.Version)"
    - cache:
        duration: 300

  backend:
    - timeout:
        seconds: 30
    - retry:
        count: 3
        interval: 1

products:
  - starter
  - premium

versioning:
  scheme: path
  currentVersion: v1

tags:
  - products
  - catalog
```

## Environment-Specific Configuration

### Multi-Environment Strategy

```yaml
# environments/dev.yaml
environment: dev
apim:
  serviceName: apim-dev
  resourceGroup: rg-apim-dev

apis:
  products-api:
    serviceUrl: https://products-dev.example.com
    policies:
      rateLimit:
        calls: 10000  # Higher limits in dev

# environments/prod.yaml
environment: prod
apim:
  serviceName: apim-prod
  resourceGroup: rg-apim-prod

apis:
  products-api:
    serviceUrl: https://products.example.com
    policies:
      rateLimit:
        calls: 1000  # Production limits
```

### Configuration Merging

Base configuration with environment overrides:

```yaml
# base-config.yaml
api:
  name: products-api
  displayName: Products API
  path: products
  protocols:
    - https

policies:
  inbound:
    - authentication:
        type: oauth2
    - rateLimit:
        calls: ${RATE_LIMIT_CALLS}
        renewalPeriod: 60

# Environment variables or overrides
# dev: RATE_LIMIT_CALLS=10000
# prod: RATE_LIMIT_CALLS=1000
```

## Policy Configuration

### Common Policy Patterns

#### Authentication

```yaml
policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad
        audiences:
          - api://products-api
        requiredClaims:
          - name: roles
            values:
              - API.Read
              - API.Write
```

#### Rate Limiting and Quotas

```yaml
policies:
  inbound:
    # Per-minute rate limit
    - rateLimit:
        calls: 1000
        renewalPeriod: 60

    # Per-week quota
    - quota:
        calls: 100000
        renewalPeriod: 604800

    # Rate limit per subscription
    - rateLimitByKey:
        calls: 100
        renewalPeriod: 60
        counterKey: "@(context.Subscription.Id)"
```

#### Request Transformation

```yaml
policies:
  inbound:
    - setHeader:
        name: X-Forwarded-For
        value: "@(context.Request.IpAddress)"

    - setQueryParameter:
        name: api-version
        value: "2024-01-01"

    - rewriteUri:
        template: "/v2{path}"
```

#### Response Caching

```yaml
policies:
  inbound:
    - cacheVaryByHeaders:
        - Accept
        - Accept-Language

  outbound:
    - cacheStore:
        duration: 3600
        cacheResponse: true
        varyByQueryParameter:
          - page
          - pageSize
```

#### CORS Configuration

```yaml
policies:
  inbound:
    - cors:
        allowedOrigins:
          - https://app.example.com
        allowedMethods:
          - GET
          - POST
          - PUT
          - DELETE
          - OPTIONS
        allowedHeaders:
          - Content-Type
          - Authorization
          - X-Requested-With
        exposeHeaders:
          - X-Total-Count
          - X-Page-Number
        allowCredentials: true
        maxAge: 3600
```

## Advanced Configuration

### Backend Service Configuration

```yaml
backend:
  url: https://products.example.com
  protocol: http

  # Load balancing
  loadBalancing:
    enabled: true
    type: roundRobin
    backends:
      - url: https://products-1.example.com
        weight: 1
      - url: https://products-2.example.com
        weight: 2

  # TLS configuration
  tls:
    validateCertificate: true
    validateHostname: true
    clientCertificate: cert-name

  # Timeouts and retries
  timeout:
    connect: 5
    request: 30
  retry:
    count: 3
    interval: 1
    maxInterval: 10
```

### Named Values (Variables)

```yaml
namedValues:
  - name: backend-url
    value: https://products.example.com
    secret: false

  - name: api-key
    value: ${API_KEY}  # From environment or Key Vault
    secret: true
    keyVault:
      secretIdentifier: https://kv.vault.azure.net/secrets/api-key

# Usage in policies
policies:
  inbound:
    - setHeader:
        name: X-API-Key
        value: "@(context.Variables.GetValueOrDefault('api-key', ''))"
```

### Product Configuration

```yaml
products:
  starter:
    displayName: Starter Plan
    description: Basic access for development
    subscriptionRequired: true
    approvalRequired: false
    subscriptionsLimit: 1
    state: published
    policies:
      quota:
        calls: 10000
        renewalPeriod: 2592000  # 30 days

  premium:
    displayName: Premium Plan
    description: Full production access
    subscriptionRequired: true
    approvalRequired: true
    subscriptionsLimit: 5
    state: published
    policies:
      quota:
        calls: 1000000
        renewalPeriod: 2592000
```

## Secrets Management

### Azure Key Vault Integration

```yaml
# Reference Key Vault secrets
secrets:
  backend-api-key:
    keyVault:
      vaultName: apim-keyvault
      secretName: backend-api-key

  oauth-client-secret:
    keyVault:
      vaultName: apim-keyvault
      secretName: oauth-client-secret

# Use in policies
policies:
  inbound:
    - setHeader:
        name: X-API-Key
        value: "@(context.Variables.GetValueOrDefault('backend-api-key', ''))"
```

### Environment Variables

```yaml
# Use environment variables for non-sensitive config
api:
  serviceUrl: ${BACKEND_URL}

policies:
  inbound:
    - rateLimit:
        calls: ${RATE_LIMIT_CALLS:1000}  # Default: 1000
```

## Configuration Validation

### Pre-Deployment Validation

```bash
# Validate YAML syntax
yamllint apim-config.yaml

# Validate against schema
ajv validate -s apim-config.schema.json -d apim-config.yaml

# Custom validation script
node validate-config.js apim-config.yaml
```

### Schema Definition

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["api", "policies"],
  "properties": {
    "api": {
      "type": "object",
      "required": ["name", "path", "serviceUrl"],
      "properties": {
        "name": { "type": "string", "pattern": "^[a-z0-9-]+$" },
        "displayName": { "type": "string" },
        "path": { "type": "string" },
        "serviceUrl": { "type": "string", "format": "uri" }
      }
    }
  }
}
```

## Best Practices

### Organization

1. **Separate concerns**
   - Base configuration for common settings
   - Environment-specific overrides
   - Sensitive data in Key Vault

2. **Use consistent naming**
   - API names: kebab-case (e.g., `products-api`)
   - Policy keys: camelCase (e.g., `rateLimit`)
   - Environment files: `{env}.yaml` pattern

3. **Version control**
   - Track all configuration in git
   - Use branches for environment-specific config
   - Review configuration changes in PRs

### Security

1. **Never commit secrets**
   - Use Key Vault for sensitive values
   - Use environment variables for configuration
   - Add secrets to .gitignore

2. **Principle of least privilege**
   - Apply restrictive policies by default
   - Open up as needed
   - Document exceptions

3. **Validate inputs**
   - Use JSON schema validation policies
   - Check request payloads
   - Sanitize query parameters

### Performance

1. **Enable caching appropriately**
   - Cache GET responses for frequently accessed data
   - Vary cache by relevant parameters
   - Set appropriate TTL values

2. **Optimize policies**
   - Minimize policy processing overhead
   - Use conditional policies when possible
   - Avoid unnecessary transformations

3. **Configure timeouts**
   - Set realistic timeout values
   - Implement retry logic
   - Handle timeouts gracefully

## Deployment

### Using Azure CLI

```bash
# Deploy configuration
az apim api create \
  --resource-group <rg> \
  --service-name <apim> \
  --api-id products-api \
  --path products \
  --display-name "Products API" \
  --service-url "$(cat config.yaml | yq '.api.serviceUrl')"

# Apply policies
az apim api policy create \
  --resource-group <rg> \
  --service-name <apim> \
  --api-id products-api \
  --policy-format rawxml \
  --value "@policy.xml"
```

### Using Bicep/ARM

```bicep
resource api 'Microsoft.ApiManagement/service/apis@2021-08-01' = {
  name: 'products-api'
  parent: apimService
  properties: {
    displayName: 'Products API'
    path: 'products'
    serviceUrl: backendUrl
    protocols: ['https']
    subscriptionRequired: true
  }
}

resource apiPolicy 'Microsoft.ApiManagement/service/apis/policies@2021-08-01' = {
  name: 'policy'
  parent: api
  properties: {
    format: 'rawxml'
    value: loadTextContent('policy.xml')
  }
}
```

## Related Documentation

- [Developer Integration Guide](developer-integration.md) - Overall workflow
- [API Specification Management](api-specification-management.md) - OpenAPI specs
- [DevOps Automation](../dev-ops/devops-automation.md) - CI/CD integration
