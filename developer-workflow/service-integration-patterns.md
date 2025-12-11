# Service Integration Patterns

## Overview

This guide covers common patterns and best practices for integrating backend services with Azure API Management, including authentication, network connectivity, and error handling strategies.

## Basic Integration Pattern

### Direct HTTP Backend

The simplest integration pattern connects APIM directly to an HTTP/HTTPS backend:

```
[Client] → [APIM Gateway] → [Backend Service]
```

**Configuration:**
```yaml
api:
  serviceUrl: https://products.example.com
  protocol: https
```

**Use Cases:**
- Public APIs with HTTPS endpoints
- Services in same Azure region
- Simple request/response patterns

## Authentication Patterns

### Pattern 1: Pass-Through Authentication

Client authenticates directly with APIM; gateway passes token to backend:

```yaml
policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad
    - setHeader:
        name: Authorization
        value: "@(context.Request.Headers.GetValueOrDefault('Authorization', ''))"
```

**Pros:**
- Backend can validate tokens independently
- Works with existing backend auth

**Cons:**
- Backend must implement auth validation
- Duplicated validation logic

### Pattern 2: Gateway Authentication with Backend Trust

Gateway validates authentication; backend trusts gateway:

```yaml
policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad
        validateClaims: true
    - setHeader:
        name: X-User-Id
        value: "@(context.Request.Headers.GetValueOrDefault('sub', ''))"
    - setHeader:
        name: X-Gateway-Key
        value: "@(context.Variables.GetValueOrDefault('gateway-secret', ''))"
```

**Backend Implementation:**
```csharp
// Backend validates gateway secret
if (Request.Headers["X-Gateway-Key"] != expectedSecret)
{
    return Unauthorized();
}

// Trust user identity from gateway
var userId = Request.Headers["X-User-Id"];
```

**Pros:**
- Centralized authentication
- Backend logic simplified
- Consistent auth across APIs

**Cons:**
- Backend depends on gateway
- Shared secret management needed

### Pattern 3: Managed Identity Integration

APIM uses managed identity to authenticate with backend:

```yaml
policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad  # User auth

  backend:
    - authenticationManagedIdentity:
        resource: https://backend.example.com
        outputTokenVariableName: backendToken
    - setHeader:
        name: Authorization
        value: "@('Bearer ' + context.Variables['backendToken'])"
```

**Prerequisites:**
- APIM managed identity enabled
- Backend grants RBAC role to APIM identity

**Pros:**
- No credential management
- Azure-native security
- Automatic token rotation

**Cons:**
- Azure-specific
- Requires RBAC configuration

## Network Integration Patterns

### Pattern 1: Public Internet

Both APIM and backend are publicly accessible:

```
[Internet] → [Public APIM] → [Public Backend]
```

**Configuration:**
- Standard APIM SKU
- Public DNS for backend
- NSG rules for backend protection

**Use Cases:**
- Public APIs
- SaaS backends
- External partners

### Pattern 2: VNET Integration

APIM in VNET with private backend access:

```
[Internet] → [APIM in VNET] → [Private Backend in VNET]
```

**Requirements:**
- Premium or Developer SKU
- Dedicated subnet for APIM
- NSG rules allowing APIM traffic

**Configuration:**
```bicep
resource apim 'Microsoft.ApiManagement/service@2021-08-01' = {
  properties: {
    virtualNetworkType: 'External'  // or 'Internal'
    virtualNetworkConfiguration: {
      subnetResourceId: subnetId
    }
  }
}
```

**Use Cases:**
- Private backends
- Regulated environments
- Enhanced security posture

### Pattern 3: Private Endpoint

Backend accessible via Azure Private Link:

```
[APIM] → [Private Endpoint] → [Backend Service]
```

**Configuration:**
```yaml
api:
  serviceUrl: https://products.privatelink.example.com
```

**Use Cases:**
- Azure PaaS services (App Service, Function Apps)
- Cross-subscription private access
- Enhanced network isolation

## Service Discovery Patterns

### Static Configuration

Backend URL configured in APIM settings:

```yaml
api:
  serviceUrl: https://products.example.com
```

**Pros:** Simple and reliable
**Cons:** Requires update for URL changes

### Dynamic Backend Selection

Select backend based on request properties:

```xml
<policies>
  <inbound>
    <set-backend-service
      base-url="@{
        var region = context.Request.Headers.GetValueOrDefault('X-Region', 'us');
        return region == 'eu'
          ? 'https://products-eu.example.com'
          : 'https://products-us.example.com';
      }" />
  </inbound>
</policies>
```

**Use Cases:**
- Multi-region routing
- A/B testing
- Canary deployments

### Service Registry Integration

Resolve backend from external registry:

```xml
<policies>
  <inbound>
    <send-request mode="new" response-variable-name="serviceRegistry">
      <set-url>https://registry.example.com/services/products</set-url>
    </send-request>
    <set-backend-service
      base-url="@(((IResponse)context.Variables['serviceRegistry']).Body.As<JObject>()['url'].ToString())" />
  </inbound>
</policies>
```

**Use Cases:**
- Microservices architectures
- Dynamic service discovery
- Kubernetes service mesh

## Error Handling Patterns

### Pattern 1: Retry Logic

Automatically retry failed requests:

```xml
<policies>
  <inbound>
    <retry condition="@(context.Response.StatusCode >= 500)"
           count="3"
           interval="1"
           max-interval="10"
           delta="1"
           first-fast-retry="true">
      <forward-request />
    </retry>
  </inbound>
</policies>
```

### Pattern 2: Circuit Breaker

Prevent cascading failures:

```xml
<policies>
  <inbound>
    <cache-lookup-value key="circuit-breaker-state" variable-name="circuitState" />
    <choose>
      <when condition="@(context.Variables.GetValueOrDefault('circuitState', '') == 'open')">
        <return-response>
          <set-status code="503" reason="Service Unavailable" />
          <set-body>Service temporarily unavailable</set-body>
        </return-response>
      </when>
    </choose>
  </inbound>

  <on-error>
    <cache-store-value key="circuit-breaker-state" value="open" duration="60" />
  </on-error>
</policies>
```

### Pattern 3: Graceful Degradation

Return cached or default response on backend failure:

```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
      <vary-by-query-parameter>id</vary-by-query-parameter>
    </cache-lookup>
  </inbound>

  <backend>
    <forward-request />
  </backend>

  <outbound>
    <cache-store duration="3600" />
  </outbound>

  <on-error>
    <choose>
      <when condition="@(context.Response == null)">
        <!-- Return default/cached response -->
        <return-response>
          <set-status code="200" />
          <set-body>@{/* return cached data */}</set-body>
        </return-response>
      </when>
    </choose>
  </on-error>
</policies>
```

## Request Transformation Patterns

### Header Manipulation

Add, modify, or remove headers:

```xml
<policies>
  <inbound>
    <!-- Add headers -->
    <set-header name="X-API-Key" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault("backend-key", ""))</value>
    </set-header>

    <!-- Remove headers -->
    <set-header name="X-Internal-Token" exists-action="delete" />

    <!-- Conditional headers -->
    <choose>
      <when condition="@(context.Request.Headers.GetValueOrDefault('X-Environment', '') == 'dev')">
        <set-header name="X-Debug" exists-action="override">
          <value>true</value>
        </set-header>
      </when>
    </choose>
  </inbound>
</policies>
```

### Body Transformation

Transform request/response payloads:

```xml
<policies>
  <inbound>
    <!-- JSON to JSON transformation -->
    <set-body>@{
      var body = context.Request.Body.As<JObject>();
      return new JObject(
        new JProperty("data", body),
        new JProperty("timestamp", DateTime.UtcNow),
        new JProperty("apiVersion", "v1")
      ).ToString();
    }</set-body>

    <!-- XML to JSON -->
    <xml-to-json kind="direct" apply="always" consider-accept-header="false" />
  </inbound>
</policies>
```

### Query Parameter Manipulation

Modify query strings:

```xml
<policies>
  <inbound>
    <!-- Add query parameter -->
    <set-query-parameter name="api-version" exists-action="override">
      <value>2024-01-01</value>
    </set-query-parameter>

    <!-- Remove sensitive parameters -->
    <set-query-parameter name="internal-key" exists-action="delete" />

    <!-- Rewrite URL -->
    <rewrite-uri template="/v2{path}" />
  </inbound>
</policies>
```

## Load Balancing Patterns

### Round Robin

Distribute requests across multiple backends:

```xml
<policies>
  <inbound>
    <set-backend-service
      base-url="@{
        var backends = new[] {
          'https://backend-1.example.com',
          'https://backend-2.example.com',
          'https://backend-3.example.com'
        };
        var index = new Random().Next(backends.Length);
        return backends[index];
      }" />
  </inbound>
</policies>
```

### Weighted Load Balancing

Prefer certain backends:

```xml
<policies>
  <inbound>
    <set-backend-service
      base-url="@{
        var rand = new Random().Next(100);
        if (rand < 70) return 'https://primary.example.com';  // 70%
        if (rand < 90) return 'https://secondary.example.com';  // 20%
        return 'https://tertiary.example.com';  // 10%
      }" />
  </inbound>
</policies>
```

## Caching Patterns

### Response Caching

Cache backend responses:

```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
      <vary-by-query-parameter>id</vary-by-query-parameter>
      <vary-by-query-parameter>page</vary-by-query-parameter>
    </cache-lookup>
  </inbound>

  <outbound>
    <cache-store duration="3600" />
  </outbound>
</policies>
```

### Cache with Conditional Requests

Use ETags for cache validation:

```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false">
      <vary-by-header>Accept</vary-by-header>
    </cache-lookup>
  </inbound>

  <outbound>
    <cache-store duration="3600" />
    <set-header name="ETag" exists-action="override">
      <value>@(context.Response.Body.As<string>().GetHashCode().ToString())</value>
    </set-header>
  </outbound>
</policies>
```

## Monitoring Integration

### Request Correlation

Propagate correlation IDs:

```xml
<policies>
  <inbound>
    <set-variable name="correlationId"
                  value="@(context.Request.Headers.GetValueOrDefault('X-Correlation-ID', Guid.NewGuid().ToString()))" />
    <set-header name="X-Correlation-ID" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault<string>("correlationId"))</value>
    </set-header>
  </inbound>

  <outbound>
    <set-header name="X-Correlation-ID" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault<string>("correlationId"))</value>
    </set-header>
  </outbound>
</policies>
```

### Application Insights Logging

Log detailed request information:

```xml
<policies>
  <inbound>
    <trace source="apim-trace" severity="information">
      <message>@($"Request: {context.Request.Method} {context.Request.Url}")</message>
      <metadata name="User-Agent" value="@(context.Request.Headers.GetValueOrDefault('User-Agent', 'unknown'))" />
    </trace>
  </inbound>
</policies>
```

## Best Practices

### Security
- Never expose backend URLs to clients
- Use managed identities when possible
- Validate and sanitize all inputs
- Apply rate limiting and quotas

### Performance
- Enable caching for frequently accessed data
- Configure appropriate timeouts
- Use compression for large responses
- Minimize policy processing overhead

### Reliability
- Implement retry logic with exponential backoff
- Use circuit breakers for failing backends
- Provide graceful degradation
- Monitor backend health

### Maintainability
- Keep policies simple and readable
- Document complex transformations
- Use named values for configuration
- Version control all policies

## Related Documentation

- [Configuration Management](configuration-management.md) - Policy configuration
- [Developer Integration Guide](developer-integration.md) - Overall workflow
- [Gateway Overview](../azure-apim/gateway-overview.md) - Gateway architecture
- [Monitoring and Observability](../monitoring-and-observability/) - Operational insights
