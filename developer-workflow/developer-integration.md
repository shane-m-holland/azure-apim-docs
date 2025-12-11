# Developer Integration Guide

This guide outlines how developers integrate with Azure API Management (APIM), covering deployment strategies, workflow patterns, and the evolution from manual to automated API publishing.

## Overview

Azure API Management is a managed service that acts as a gateway between backend services and API consumers. This guide helps development teams understand how to work with APIM effectively, from initial learning phases through mature automated workflows.

## Core APIM Capabilities for Developers

APIM provides centralized management for:

- **API Gateway** - Routing and load balancing for backend services
- **Authentication & Authorization** - Centralized security enforcement
- **Rate Limiting & Quotas** - Protection against overuse
- **Request/Response Transformation** - Protocol and format adaptation
- **Analytics & Monitoring** - Built-in observability
- **Developer Portal** - Auto-generated documentation and testing tools
- **API Versioning** - Manage breaking and non-breaking changes
- **Product Management** - Group APIs with tiered access permissions

## Common Use Cases

### 1. API Consolidation
Provide a unified API gateway where all services are published with consistent authentication mechanisms.

**Example Scenario:**
- Multiple microservices with different auth patterns
- APIM provides single OAuth endpoint
- All services inherit consistent security policies

### 2. Legacy Modernization
Expose SOAP services as REST APIs with modern authentication.

**Example Scenario:**
- Legacy SOAP service requires complex XML
- APIM transforms REST JSON to SOAP XML
- Consumers use modern REST interfaces

### 3. Partner Integration
Subscription-based access with tiered permissions for external partners.

**Example Scenario:**
- Partners need access to specific APIs
- APIM Products control which APIs partners can access
- Subscription keys track usage and enforce quotas

### 4. Microservices Gateway
Centralized service mesh capabilities for internal microservices.

**Example Scenario:**
- Dozens of microservices need consistent policies
- APIM applies rate limiting, logging, and auth globally
- Services focus on business logic, not cross-cutting concerns

### 5. Mobile/SPA Backends
Optimized caching and response transformation for frontend applications.

**Example Scenario:**
- Mobile apps need lightweight JSON responses
- APIM caches frequently accessed data
- Backend changes don't require mobile app updates

## Deployment Strategy Evolution

Organizations typically evolve through three distinct phases when adopting APIM. Each phase builds on the previous one, adding automation and developer empowerment.

### Phase 1: Short-Term - Centralized Manual Deployments

#### Overview
Single infrastructure repository for all APIs with manual deployment triggers.

#### Characteristics

**Repository Structure:**
```
apim-infrastructure/
├── apis/
│   ├── products-api/
│   │   ├── openapi.yaml
│   │   └── apim-config.yaml
│   ├── orders-api/
│   │   ├── openapi.yaml
│   │   └── apim-config.yaml
├── environments/
│   ├── dev.yaml
│   ├── test.yaml
│   └── prod.yaml
└── .github/workflows/
    └── deploy-apim.yaml
```

**Workflow:**
1. Developers create OpenAPI specs and APIM configs
2. Submit pull request to infrastructure repository
3. Infrastructure team reviews submission
4. Manual trigger of GitHub Actions workflow
5. Deployment to APIM instance

**Authentication:**
- Remains at backend services initially
- APIM validates subscription keys only
- Focus is on learning APIM capabilities

**Pros:**
- Simple to understand and implement
- Central team maintains control and quality
- Good for learning APIM features
- Low risk during initial adoption

**Cons:**
- Bottleneck on infrastructure team
- Slower deployment cycles
- Developers don't own their API definitions
- Duplicates authentication logic

**Best For:**
- Initial APIM implementation (first 3-6 months)
- Organizations with small API catalogs (< 10 APIs)
- Teams new to APIM concepts
- Proof-of-concept and learning phases

#### Implementation Details

**Developer Responsibilities:**
- Create OpenAPI specification for service
- Define APIM configuration (policies, products, versioning)
- Test service independently before submission
- Submit both files via pull request

**Infrastructure Team Responsibilities:**
- Review API specifications for quality and consistency
- Validate APIM configurations
- Manually trigger deployments to each environment
- Monitor deployment success and troubleshoot issues

**Example APIM Configuration:**
```yaml
api:
  name: products-api
  displayName: Products API
  description: Manage product catalog
  path: products
  serviceUrl: https://products-backend.example.com
  protocols:
    - https

policies:
  inbound:
    - rateLimit:
        calls: 100
        renewalPeriod: 60
    - quota:
        calls: 10000
        renewalPeriod: 604800

products:
    - starter
    - premium

versioning:
  scheme: path
  currentVersion: v1
```

### Phase 2: Medium-Term - Service-Managed Specifications

#### Overview
API specifications move to individual service repositories while maintaining manual deployment approvals.

#### Characteristics

**Repository Structure:**
```
products-service/
├── src/
├── openapi.yaml          # Owned by service team
└── apim-config.yaml      # Owned by service team

apim-infrastructure/
├── apis/
│   ├── products-api/
│   │   └── api-reference.yaml    # Points to service repo
│   ├── orders-api/
│   │   └── api-reference.yaml
└── .github/workflows/
    └── sync-apis.yaml
```

**API Reference File:**
```yaml
apiReference:
  repository: products-service
  specPath: openapi.yaml
  configPath: apim-config.yaml
  branch: main
```

**Workflow:**
1. Developers maintain specs in their service repository
2. Infrastructure repo references service spec locations
3. Automated sync detects changes in service repos
4. Manual approval required before deployment
5. Approved changes deployed to APIM

**Authentication:**
- Pilot OAuth at gateway layer for selected APIs
- Begin transitioning from backend auth to APIM auth
- Gradually migrate APIs to centralized authentication

**Pros:**
- Service teams own their API contracts
- Specs live close to implementation code
- Enables automated sync processes
- Reduces infrastructure team bottleneck

**Cons:**
- Still requires manual approval gates
- Multiple repositories to manage
- Authentication migration adds complexity
- Coordination needed across teams

**Best For:**
- Growing API catalogs (10-30 APIs)
- Organizations building DevOps maturity
- Transition period (6-12 months)
- Teams with clear service ownership

#### Implementation Details

**Service Repository Changes:**
Developers add two files to their service repository:

1. **openapi.yaml** - API specification
2. **apim-config.yaml** - APIM-specific configuration

**Synchronization Process:**
- Infrastructure repository workflows check service repos for changes
- SHA comparison determines if specs have been updated
- Changed specs queued for approval
- Approved changes automatically deployed

**Example Sync Workflow:**
```yaml
name: Sync APIM APIs
on:
  schedule:
    - cron: '0 * * * *'  # Hourly check
  workflow_dispatch:

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    steps:
      - name: Check service repos for spec changes
        # Compare commit SHAs of spec files
        # Queue changes for review

  deploy-approved:
    needs: detect-changes
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy approved API changes
        # Deploy to APIM using Bicep/scripts
```

### Phase 3: Long-Term - Automated CI/CD

#### Overview
Fully automated API lifecycle from code commit to production deployment.

#### Characteristics

**Repository Structure:**
```
products-service/
├── src/
│   └── controllers/
│       └── products.controller.ts  # Annotated code
├── .github/workflows/
│   └── deploy-api.yaml              # Auto-deploys on merge
└── apim-config.yaml                 # Minimal config
```

**Workflow:**
1. Developers annotate code with OpenAPI decorators
2. Build process auto-generates OpenAPI spec
3. Merge to main triggers automated deployment to dev/QA
4. Manual approval gate for production
5. Full end-to-end automation with rollback capability

**Authentication:**
- Full OAuth 2.0 / OIDC at gateway layer
- Backend services trust gateway authentication
- No redundant auth logic in services
- Centralized identity provider integration

**Pros:**
- Specs always match implementation
- Fastest time from code to deployment
- Developers follow normal git workflows
- Consistent authentication across all APIs
- Reduced operational overhead

**Cons:**
- Requires mature DevOps practices
- Initial setup complexity
- Teams must understand CI/CD pipelines
- Requires discipline in code annotations

**Best For:**
- Mature DevOps organizations
- Large API catalogs (30+ APIs)
- Teams with established CI/CD practices
- Long-term operational efficiency

#### Implementation Details

**Auto-Generated Specifications:**

Many frameworks support OpenAPI generation from code annotations:

**NestJS (Node.js):**
```typescript
@Controller('products')
@ApiTags('products')
export class ProductsController {
  @Get()
  @ApiOperation({ summary: 'Get all products' })
  @ApiResponse({ status: 200, type: [Product] })
  async findAll(): Promise<Product[]> {
    return this.productsService.findAll();
  }
}
```

**ASP.NET Core:**
```csharp
[ApiController]
[Route("products")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// Gets all products
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<Product>), 200)]
    public async Task<IActionResult> GetProducts()
    {
        return Ok(await _service.GetAllAsync());
    }
}
```

**Automated Deployment Pipeline:**
```yaml
name: Deploy API to APIM
on:
  push:
    branches: [main]

jobs:
  deploy-to-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Generate OpenAPI Spec
        run: npm run generate:openapi

      - name: Deploy to APIM Dev
        run: |
          az apim api import \
            --path products \
            --specification-format OpenApi \
            --specification-path ./openapi.yaml

  deploy-to-prod:
    needs: deploy-to-dev
    environment: production  # Requires manual approval
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to APIM Production
        # Same deployment steps for prod
```

## Developer Changes Required

Regardless of which phase you're in, developers need to make these minimal additions:

### 1. Create OpenAPI Specification

Either manually or auto-generated from code:

```yaml
openapi: 3.0.1
info:
  title: Products API
  version: 1.0.0
  description: API for managing product catalog
servers:
  - url: https://api.example.com/products
paths:
  /products:
    get:
      summary: List all products
      operationId: listProducts
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Product'
components:
  schemas:
    Product:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        price:
          type: number
```

### 2. Add APIM Configuration File

Define policies and settings:

```yaml
api:
  name: products-api
  displayName: Products API
  path: products
  serviceUrl: https://products-backend.example.com
  protocols:
    - https

policies:
  inbound:
    - authentication:
        type: oauth2
        provider: azure-ad
    - rateLimit:
        calls: 1000
        renewalPeriod: 60
    - cors:
        allowedOrigins:
          - https://app.example.com
        allowedMethods:
          - GET
          - POST
          - PUT
          - DELETE

  outbound:
    - setHeader:
        name: X-Powered-By
        value: Azure APIM

products:
  - starter
  - premium

versioning:
  scheme: path
```

### 3. Continue Normal Development

Developers continue standard practices:
- Local development and testing
- Unit and integration tests
- Code reviews and pull requests
- Service deployment to backend infrastructure

APIM integration happens separately and doesn't block service development.

## Benefits Gained Through APIM

| Activity | Before APIM | With APIM |
|----------|-------------|-----------|
| **Authentication** | Per-service implementation | Centralized gateway management |
| **Documentation** | Manual maintenance | Auto-generated from specs |
| **Monitoring** | Custom logging | Built-in analytics |
| **Rate Limiting** | Custom code | Policy-based configuration |
| **API Versioning** | URL routing logic | Declarative versioning |
| **Partner Access** | Manual key distribution | Self-service subscriptions |
| **Testing** | Postman/curl scripts | Built-in test console |
| **Security** | Per-service SSL/auth | Gateway-enforced policies |

## Developer Workflow Comparison

### Traditional Development (Without APIM)

```
1. Write service code
2. Implement authentication in service
3. Add rate limiting logic
4. Create documentation manually
5. Deploy service
6. Manually distribute API keys to consumers
7. Monitor with custom logging
```

### APIM-Enabled Development (Phase 3)

```
1. Write service code with OpenAPI annotations
2. Commit code (spec auto-generated)
3. CI/CD deploys service and APIM config
4. Documentation auto-published to Developer Portal
5. Consumers self-service subscribe
6. Monitor via Azure Portal/Application Insights
```

## Migration Path

### Moving from Phase 1 to Phase 2

**Prerequisites:**
- Service teams comfortable with OpenAPI
- Clear service ownership established
- Git workflows mature

**Steps:**
1. Create OpenAPI specs in service repositories
2. Update infrastructure repo to reference service specs
3. Implement sync workflow in infrastructure repo
4. Migrate one API as pilot
5. Roll out to remaining APIs incrementally

**Timeline:** 2-3 months

### Moving from Phase 2 to Phase 3

**Prerequisites:**
- Mature CI/CD pipelines per service
- Auto-generation tooling selected
- Gateway authentication strategy defined

**Steps:**
1. Add OpenAPI generation to service build processes
2. Create deployment workflows in service repos
3. Pilot with non-critical API
4. Refine automation based on learnings
5. Roll out to remaining services

**Timeline:** 3-6 months

## Best Practices

### API Specification Management

- **Version control all specs** - Track changes over time
- **Validate specs in CI** - Catch errors before deployment
- **Use consistent naming** - Standard conventions across APIs
- **Document examples** - Include sample requests/responses
- **Define error schemas** - Standard error response formats

### Configuration Management

- **Environment-specific configs** - Separate dev/test/prod settings
- **Secure secrets** - Use Key Vault for sensitive values
- **Consistent policies** - Standard rate limits and quotas
- **Review policy changes** - Test policies before production

### Workflow Integration

- **Automate where possible** - Reduce manual steps
- **Clear approval gates** - Define who approves what
- **Fast feedback loops** - Quick validation and deployment
- **Rollback procedures** - Use revisions for safe rollback

## Troubleshooting Common Issues

### API Not Accessible After Deployment

**Symptoms:** 404 errors when calling API through APIM

**Causes:**
- API not added to a Product
- Subscription key not provided
- Incorrect path configuration

**Solutions:**
```bash
# Verify API is added to product
az apim api list --resource-group <rg> --service-name <apim>

# Check product associations
az apim product api list --resource-group <rg> \
  --service-name <apim> --product-id <product>

# Test with subscription key
curl -H "Ocp-Apim-Subscription-Key: <key>" \
  https://<apim>.azure-api.net/api/endpoint
```

### Backend Service Unreachable

**Symptoms:** 500 errors or timeout errors

**Causes:**
- Incorrect backend URL in config
- Network connectivity issues
- Backend service authentication required

**Solutions:**
- Verify backend URL is correct and accessible
- Check VNET integration if using private endpoints
- Add authentication policies if backend requires it

### Policy Validation Failures

**Symptoms:** Deployment fails with policy validation error

**Causes:**
- Invalid XML in policy definition
- Unsupported policy element
- Missing required attributes

**Solutions:**
- Validate policy XML syntax
- Reference [APIM policy documentation](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- Test policies in non-production first

## Additional Resources

### Related Documentation
- [API Specification Management](api-specification-management.md) - OpenAPI best practices
- [Configuration Management](configuration-management.md) - APIM config patterns
- [Azure APIM Overview](../azure-apim/apim-overview.md) - Core concepts

### External References
- [Azure APIM Documentation](https://learn.microsoft.com/en-us/azure/api-management/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [APIM Policy Reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)

### Related Repositories
- [azure-apim-bicep](https://github.com/shane-m-holland/azure-apim-bicep) - Infrastructure automation
- [example-apim-infrastructure](https://github.com/shane-m-holland/example-apim-infrastructure) - Reference implementation
