# Azure API Management (APIM)

## Overview on APIs

In Azure API Management (APIM), an **API** represents a set of operations that expose backend services to consumers. APIs are the core building blocks in APIM, enabling developers, partners, and customers to interact with backend systems in a secure, consistent, and managed way.

This guide provides an overview of how APIs are defined, managed, and consumed in APIM, along with best practices for designing and maintaining them.

---

## Defining APIs in APIM

APIs in APIM can be created in several ways:

- **Import from specification**:
  - OpenAPI (Swagger)
  - WSDL (SOAP services)
  - GraphQL schema
- **Link from Azure services**:
  - App Services
  - Azure Functions
  - Logic Apps
- **Manual creation**:
  - Define operations directly within APIM

Key properties of an API:

- **Display name**: Human-readable name for the API.
- **Path**: URL path that routes requests to the API (e.g., `/weather`).
- **Service URL**: Backend endpoint where requests are forwarded.
- **Protocols**: Supported protocols (`https` is required).
- **Operations**: Individual endpoints (e.g., `GET /forecast`, `POST /orders`).

---

## API Lifecycle in APIM

1. **Design**: Define endpoints, operations, and contracts using OpenAPI or GraphQL.
2. **Import**: Bring the API into APIM via specification or Azure service integration.
3. **Configure**: Apply security, policies, and transformations.
4. **Test**: Validate endpoints using the developer portal or APIM test console.
5. **Publish**: Bundle the API into a **Product** for consumer access.
6. **Maintain**: Use revisions for non-breaking changes and versions for breaking changes. 

    - Links to additional documentation on [Versioning](./api-versioning.md) & [Revisions](./api-revisions.md)

---

## Policies for APIs

Policies are XML-based rules applied to API requests and responses at runtime. Common policies for APIs include:

- **Rate limiting**: Control the number of requests per minute/hour.
- **Quota enforcement**: Set maximum allowed calls per subscription.
- **Transformation**: Modify request/response formats (e.g., XML ↔ JSON).
- **Caching**: Store responses to reduce backend load.
- **Validation**: Ensure request payloads meet schema requirements.
- **Logging**: Send request/response data to Application Insights or Event Hub.

---

## Best Practices for APIs in APIM

### API Design

- **Use OpenAPI (Swagger)** for standardized definitions.
- **Adopt consistent naming conventions** for paths, operations, and parameters.
- **Keep APIs versioned** to manage breaking changes.
- **Document error responses** and use standard HTTP status codes.

### Security

- **Enforce authentication** (OAuth 2.0, OpenID Connect, subscription keys).
- **Use HTTPS-only** for all APIs.
- **Mask sensitive data** (e.g., headers, payloads) in logs and responses.

### Performance

- **Enable caching** for frequently accessed endpoints.
- **Throttle requests** to protect backend systems from overload.
- **Use async APIs** where possible for long-running operations.

### Governance

- **Separate environments** (dev, test, prod) for safe testing and rollouts.
- **Use Products** to group APIs logically and control access.
- **Apply consistent policies** across APIs for uniform governance.
- **Audit changes** using source control (ARM/Bicep + CI/CD).

### Developer Experience

- **Provide a developer portal** with documentation and interactive testing.
- **Include examples and sample code** in API docs.
- **Offer clear onboarding steps** for requesting and using subscriptions.
- **Monitor usage analytics** to improve API quality and adoption.

---

## Example API Definition (OpenAPI snippet)

```yaml
openapi: 3.0.1
info:
  title: Weather API
  version: 1.0.0
paths:
  /forecast:
    get:
      summary: Get weather forecast
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  temperature:
                    type: integer
                  condition:
                    type: string
```
