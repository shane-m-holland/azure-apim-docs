# Azure API Management (APIM)

## Automation with Bicep

Bicep is a domain-specific language (DSL) that simplifies authoring Azure Resource Manager (ARM) templates. When applied to Azure API Management (APIM), Bicep enables repeatable, auditable, and automated deployments of APIM services, APIs, products, and policies.

Automation via Bicep reduces manual configuration, eliminates configuration drift, and integrates seamlessly with CI/CD pipelines such as GitHub Actions or Azure DevOps.

---

## Why Use Bicep for APIM?

- **Readable**: Concise syntax compared to raw JSON ARM templates.
- **Modular**: Encourages splitting APIM configuration into reusable modules.
- **Parametrized**: Supports `.bicepparam` files for environment-specific values.
- **Integrated**: Works natively with Azure CLI, GitHub Actions, and Azure DevOps.
- **Idempotent**: Deployments can be safely re-run without introducing drift.

---

## Common APIM Resources Managed by Bicep

- **APIM Service Instance**
  - SKU, capacity, location, networking, diagnostics
- **APIs**
  - Definitions imported via OpenAPI, GraphQL, or WSDL
- **Products**
  - Logical groupings of APIs for consumer access
- **Policies**
  - XML-based inbound/outbound rules for requests and responses
- **Named Values**
  - Centralized configuration items (often secret-backed by Key Vault)
- **Loggers & Diagnostics**
  - Integrations with Application Insights or Event Hub

---

## Example: APIM Deployment with Bicep

```bicep
@description('APIM service name')
param apimServiceName string

@description('Location of resources')
param location string = 'eastus'

@description('Publisher email')
param publisherEmail string

@description('Publisher name')
param publisherName string

@description('Backend service base URL')
param backendServiceUrl string

@description('URI to an OpenAPI definition')
param openApiDocumentUri string

resource apim 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: apimServiceName
  location: location
  sku: {
    name: 'Developer'
    capacity: 1
  }
  properties: {
    publisherEmail: publisherEmail
    publisherName: publisherName
  }
}

resource weatherApi 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  name: '${apim.name}/weather-v1'
  properties: {
    displayName: 'Weather API'
    path: 'weather'
    serviceUrl: backendServiceUrl
    protocols: [ 'https' ]
    format: 'openapi-link'
    value: openApiDocumentUri
  }
}
```

Deploying with Azure CLI

```bash
az deployment group create \
  --resource-group <rg-name> \
  --template-file main.bicep \
  --parameters apimServiceName=<name> \
               publisherEmail=<email> \
               publisherName=<org> \
               backendServiceUrl=https://backend.example.com \
               openApiDocumentUri=https://example.com/openapi.json
```

### Best Practices for Bicep with APIM

#### Structure & Organization

- Use modules for logical components( `service.bicep`, `api.bicep`, `product.bicep`)
- Separate infrastructure and configuration (deploy the APIM instance separately from APIs and policies).
- Adopt `.bicepparam` files for environment-specific differences (dev/test/prod).

#### Security

- Reference Key Vault secrets for subscription keys or backend credentials:
- Never hardcode secrets in templates or repositories.
- Apply policies to mask sensitive headers or payloads.

#### CI/CD Integration

- Automate deployments with GitHub Actions (`azure/arm-deploy`) or Azure DevOps pipelines.
- Use environment approvals before promoting to test or prod.
- Validate templates inCI before running deployments (`az deployment group validate`).

#### Governance & Lifecycle

- Version APIs for breaking changes and use revisions for incremental edits.
- Apply consistent policies across APIs for rate limiting, securtiy, and transformations.
- Monitor drift: Periodically export APIM configuration (via DevOps Resource Kit) and compare with repo state.

#### Performance

- Bundle APIs logically into products for access management.
- Leverage caching policies in APIs where appropriate.
- Test deployments in lower environments before promoting to production.

---

### Key Takeaways

- Bicep is the prefferred IaC approach for managing APIM due to its simplicity and integration with Azure tooling.
- Modularize and parameterize your templates to enable scalable, environment-aware deployments.
- Pair Bicep with **CI/CD pipelines** for a fully automated APIM lifecycle.
- Follow best practices for security, governance, and performance to maximize the value of automation.
