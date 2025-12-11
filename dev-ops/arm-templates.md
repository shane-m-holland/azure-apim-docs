# Azure API Mangaement (APIM)

## ARM Templates for Azure API Management (APIM)

Azure Resource Manager (ARM) templates are **declarative JSON** files that describe Azure resources. They enable idempotent deployments, parameterization, and repeatable infrastructure changes.

### Why Use ARM Templates for APIM?

- **Consistency**: Eliminate manual configuration across environments.
- **Version Control**: Store APIM configuration in source control for audit and rollback.
- **Repeatability**: Deploy the same APIM configuration to dev, test, and prod environments.
- **Integration**: Use ARM with CI/CD tools like GitHub Actions or Azure DevOps.
- **Governance**: Enforce organizational standards by codifying configurations.

### How ARM works

- **Schema**: ARM is JSON describing desired state.
- **Parameters/variables**: Drive environment differences; do not hardcode names, SKUs, or URIs.
- **Resource types**: `Microsoft.Api.Management/service`, and children like `.../apis`, `.../products`, `.../namedValues`, `.../loggers`, `.../diagnostics`.

---

### APIM Resources Defined in ARM

ARM templates can define:

- **APIM Service**: SKU, capacity, networking, diagnostics.
- **APIs**: Endpoints, operations, and backends.
- **Products**: Logical groupings of APIs for access control.
- **Policies**: Inbound/outbound rules written in XML.
- **Named Values**: Centralized configuration items, often linked to Key Vault.
- **Loggers & Diagnostics**: Integration with Application Insights, Event Hub, or Storage.

---

### APIM considerations

- APIM has many **child resources** (e.g., `service/apis`, `service/products`, `service/namedValues`, `service/apis/policies`).
- Use `dependsOn` to ensure service exists before child resources.
- You can deploy child resources **without** redefining the parent APIM service if it already exists; supply the correct **name** hierarchy.

## Example: APIM ARM Template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "apimServiceName": { "type": "string" },
    "location": { "type": "string", "defaultValue": "eastus" },
    "publisherEmail": { "type": "string" },
    "publisherName": { "type": "string" },
    "backendServiceUrl": { "type": "string" },
    "openApiDocumentUri": { "type": "string" }
  },
  "resources": [
    {
      "type": "Microsoft.ApiManagement/service",
      "apiVersion": "2023-03-01-preview",
      "name": "[parameters('apimServiceName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Developer",
        "capacity": 1
      },
      "properties": {
        "publisherEmail": "[parameters('publisherEmail')]",
        "publisherName": "[parameters('publisherName')]"
      }
    },
    {
      "type": "Microsoft.ApiManagement/service/apis",
      "apiVersion": "2023-03-01-preview",
      "name": "[format('{0}/{1}', parameters('apimServiceName'), 'weather-v1')]",
      "dependsOn": [
        "[resourceId('Microsoft.ApiManagement/service', parameters('apimServiceName'))]"
      ],
      "properties": {
        "displayName": "Weather API",
        "path": "weather",
        "serviceUrl": "[parameters('backendServiceUrl')]",
        "protocols": [ "https" ],
        "format": "openapi-link",
        "value": "[parameters('openApiDocumentUri')]"
      }
    }
  ],
  "outputs": {
    "apimServiceId": {
      "type": "string",
      "value": "[resourceId('Microsoft.ApiManagement/service', parameters('apimServiceName'))]"
    }
  }
}
```

## Best Practices for ARM Templates with APIM

### Structure & Modularity

- Separate **infrastructure** (APIM instance) from **configuration** (APIs, products, policies).
- Use **linked templates** or modular deployments for maintainability.
- Organize templates into directories: `/infra`, `/apis`, `/policies`.

### Parameterization

- Use parameters for environment-specific values (service name, location, URLs).
- Keep secrets in **Azure Key Vault** and reference them in ARM, not in source control.
- Use parameter files (`.json`) for dev/test/prod environments.

### Governance

- Apply consistent **API naming conventions** in templates.
- Use **policies** for rate limiting, transformations, and caching.
- Track template changes through version control and pull requests.

### CI/CD Integration

- Deploy ARM templates in **pipelines** (GitHub Actions or Azure DevOps).
- Validate templates before deployment:

  ```bash
  az deployment group validate --resource-group <rg> --template-file apim-template.json
    ```

### Maintainablity

- Keep templates under source control for audit and rollback.
- Use outputs to pass APIM identifiers to dependent resources.
- Regularly export APIM configurations using the APIM DevOps Resource Kit ensure templates match deployed state.

## Key Takeaways

- ARM templates provide a declarative and repeatable way to deploy and configure APIM.
- Structure templates for reusability and modularity.
- Parameterize for multiple environments and integrate with CI/CD pipelines.
- Follow best practices for securtiy, governance, and maintainability to maximize autoation benefits.
