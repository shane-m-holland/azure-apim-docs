# API Specification Management

## Overview

OpenAPI (formerly Swagger) specifications are the foundation of API definitions in Azure APIM. This guide covers best practices for creating, maintaining, and managing API specifications.

## What is OpenAPI?

OpenAPI is a standardized, language-agnostic specification for describing REST APIs. It provides:

- **Machine-readable format** - Tools can parse and validate specifications
- **Human-readable documentation** - Clear description of API capabilities
- **Code generation** - Auto-generate client libraries and server stubs
- **Testing support** - Import into testing tools like Postman
- **APIM integration** - Direct import into Azure API Management

## OpenAPI Structure

### Basic Structure

```yaml
openapi: 3.0.1
info:
  title: API Name
  description: API description
  version: 1.0.0
  contact:
    name: API Team
    email: api-team@example.com

servers:
  - url: https://api.example.com
    description: Production server

paths:
  /resource:
    get:
      summary: Get resource
      operationId: getResource
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Resource'

components:
  schemas:
    Resource:
      type: object
      properties:
        id:
          type: string
        name:
          type: string

  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://login.example.com/oauth/authorize
          tokenUrl: https://login.example.com/oauth/token
          scopes:
            read: Read access

security:
  - oauth2: [read]
```

## Best Practices

### Design Guidelines

1. **Use Consistent Naming**
   - Resource names should be plural nouns (e.g., `/products`, `/orders`)
   - Use kebab-case for multi-word resources (e.g., `/order-items`)
   - Operation IDs should be camelCase (e.g., `getProducts`, `createOrder`)

2. **Version from the Start**
   - Include version in info section
   - Plan versioning strategy (path, header, or query)
   - Document breaking vs. non-breaking changes

3. **Document Thoroughly**
   - Add descriptions for all endpoints
   - Include examples for requests and responses
   - Document error responses with status codes
   - Explain query parameters and their effects

4. **Define Schemas Clearly**
   - Use component schemas for reusability
   - Include data types, formats, and validations
   - Mark required fields explicitly
   - Add examples for complex schemas

### Validation

Always validate OpenAPI specifications before deployment:

```bash
# Using openapi-validator
npm install -g @apidevtools/swagger-cli
swagger-cli validate openapi.yaml

# Using Spectral
npm install -g @stoplight/spectral-cli
spectral lint openapi.yaml
```

### Maintenance

- **Keep specs in sync with code** - Update specs when API changes
- **Use automated generation** - Generate specs from code annotations when possible
- **Version control** - Track spec changes in git
- **Review changes** - Include spec updates in code reviews

## Auto-Generation from Code

### NestJS (Node.js)

```typescript
// Install dependencies
// npm install @nestjs/swagger swagger-ui-express

// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Products API')
    .setDescription('Product management API')
    .setVersion('1.0')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document);

  // Export OpenAPI spec
  fs.writeFileSync('./openapi.yaml', yaml.stringify(document));

  await app.listen(3000);
}

// Controller annotations
@Controller('products')
@ApiTags('products')
export class ProductsController {
  @Get()
  @ApiOperation({ summary: 'Get all products' })
  @ApiResponse({ status: 200, description: 'Success', type: [Product] })
  async findAll(): Promise<Product[]> {
    return this.productsService.findAll();
  }
}
```

### ASP.NET Core

```csharp
// Install NuGet package: Swashbuckle.AspNetCore

// Program.cs
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Products API",
        Version = "v1"
    });
});

app.UseSwagger();
app.UseSwaggerUI();

// Controller
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

### Python (FastAPI)

```python
# FastAPI auto-generates OpenAPI
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="Products API", version="1.0")

class Product(BaseModel):
    id: str
    name: str
    price: float

@app.get("/products", response_model=List[Product])
async def get_products():
    """Get all products"""
    return await products_service.get_all()

# Export OpenAPI spec
import json
with open("openapi.json", "w") as f:
    json.dump(app.openapi(), f)
```

## Common Patterns

### Pagination

```yaml
paths:
  /products:
    get:
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: pageSize
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Paginated results
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Product'
                  page:
                    type: integer
                  pageSize:
                    type: integer
                  totalCount:
                    type: integer
```

### Error Responses

```yaml
components:
  schemas:
    Error:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: string

  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

paths:
  /products/{id}:
    get:
      responses:
        '200':
          description: Success
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
```

## Integration with APIM

### Importing Specifications

```bash
# Azure CLI
az apim api import \
  --resource-group <resource-group> \
  --service-name <apim-service> \
  --path products \
  --display-name "Products API" \
  --specification-format OpenApi \
  --specification-path ./openapi.yaml \
  --subscription-required true

# PowerShell
Import-AzApiManagementApi `
  -Context $context `
  -SpecificationFormat OpenApi `
  -SpecificationPath ./openapi.yaml `
  -Path products
```

### Specification Updates

When updating an existing API:

```bash
# Update using revision (non-breaking)
az apim api revision create \
  --resource-group <rg> \
  --service-name <apim> \
  --api-id products-api \
  --api-revision 2 \
  --api-revision-description "Added filter parameters"

# Update using version (breaking changes)
az apim api version-set create \
  --resource-group <rg> \
  --service-name <apim> \
  --version-set-id products-versions \
  --display-name "Products API" \
  --versioning-scheme Path
```

## Tools and Resources

### Specification Tools

- **Swagger Editor** - Online editor with validation
- **Stoplight Studio** - Visual API design tool
- **Postman** - Import specs for testing
- **Spectral** - OpenAPI linter with custom rules

### Validation Libraries

```bash
# JavaScript
npm install @apidevtools/swagger-cli

# Python
pip install openapi-spec-validator

# Go
go get github.com/go-openapi/spec
```

## Related Documentation

- [Developer Integration Guide](developer-integration.md) - Overall workflow
- [Configuration Management](configuration-management.md) - APIM-specific configs
- [API Overview](../azure-apim/api-overview.md) - APIM API concepts
