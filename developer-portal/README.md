# Developer Portal

This section provides comprehensive guidance on the Azure API Management Developer Portal, including customization, security, authentication delegation, and content management strategies.

## Table of Contents

- [Developer Portal Overview](dev-portal-overview.md) - Introduction to the Developer Portal and its capabilities
- [Customization Guide](dev-portal-customization.md) - How to customize branding, content, and functionality
- [OIDC Authentication and Delegation](oidc-delegation.md) - Implementing external authentication with OIDC providers
- [User Management](user-management.md) - Managing users, groups, and permissions
- [Content Strategy](content-strategy.md) - Best practices for documentation and developer experience
- [Environment Promotion](environment-promotion.md) - Promoting portal changes across QA, Stage, and Production

## What is the Developer Portal?

The Developer Portal is an automatically generated, customizable website that serves as the primary interface for API consumers. It provides:

- **API Discovery** - Browse available APIs and products
- **Interactive Documentation** - Auto-generated from OpenAPI specifications
- **API Testing** - Built-in console for trying APIs in the browser
- **Self-Service Subscriptions** - Request access and manage subscription keys
- **Authentication** - Sign in with various identity providers
- **Customization** - Brand and tailor the experience to your organization

## Key Features

### For API Consumers

**Discover APIs**
- Browse catalog of published APIs
- Search by name, category, or functionality
- View detailed API documentation

**Learn and Test**
- Interactive API console with "Try It" functionality
- Code samples in multiple languages
- Example requests and responses

**Self-Service Access**
- Sign up for developer accounts
- Subscribe to API products
- Manage subscription keys
- View usage analytics

### For API Publishers

**Automated Documentation**
- Documentation generated from OpenAPI specs
- Always up-to-date with latest API definitions
- Consistent format across all APIs

**Customizable Experience**
- Visual editor for no-code customization
- Programmatic customization for advanced scenarios
- Custom branding and content

**Access Control**
- Role-based access to APIs and content
- Product-based subscription models
- Integration with identity providers

## Common Use Cases

| Scenario | How Developer Portal Helps |
|----------|---------------------------|
| **Internal API Discovery** | Developers browse and discover internal APIs across the organization |
| **Partner Integration** | External partners self-service subscribe to specific API products |
| **API Documentation** | Centralized, always up-to-date documentation for all APIs |
| **API Testing** | Interactive console allows testing without writing code |
| **Developer Onboarding** | Custom guides and tutorials help new developers get started |
| **Subscription Management** | Developers manage their own keys without admin involvement |

## Authentication Strategies

### Native Authentication

Built-in authentication using APIM's user database:

**Pros:**
- Simple setup, no external dependencies
- Immediate availability
- Good for testing and internal use

**Cons:**
- Separate user database from enterprise identity
- Limited integration with existing SSO
- Manual user management

### Azure AD / Azure AD B2C Integration

Leverage Azure Active Directory for authentication:

**Pros:**
- Enterprise SSO integration
- Centralized identity management
- Support for MFA and conditional access

**Cons:**
- Azure-specific
- Requires Azure AD configuration
- Licensing considerations for B2C

### OIDC Delegation Pattern

Delegate authentication to external OIDC providers (Okta, Auth0, etc.):

**Pros:**
- Use existing identity providers
- Support for any OIDC-compliant provider
- Centralized identity across applications

**Cons:**
- Requires delegation function app implementation
- Additional infrastructure to maintain
- More complex setup

See [OIDC Authentication and Delegation](oidc-delegation.md) for detailed implementation guidance.

## Customization Approaches

### Visual Editor (No-Code)

Use the built-in editor for common customization:

- **Branding** - Logo, colors, favicon, fonts
- **Content** - Add pages, widgets, and custom HTML
- **Navigation** - Organize menus and page structure
- **Layouts** - Arrange widgets with drag-and-drop

**Best For:**
- Standard customization needs
- Non-technical teams
- Quick changes and updates

### Programmatic Customization

Fork the open-source portal for advanced customization:

- **Full control** - Modify any aspect of the portal
- **Custom functionality** - Add JavaScript, CSS, new features
- **Version control** - Track changes in git
- **CI/CD integration** - Automated deployment

**Best For:**
- Unique branding requirements
- Custom functionality
- Integration with other systems
- Organizations with frontend development capacity

## Getting Started

If you're new to the Developer Portal, follow this path:

1. **Understand Capabilities** - Read [Developer Portal Overview](dev-portal-overview.md)
2. **Access Your Portal** - Navigate to the portal from Azure Portal
3. **Customize Branding** - Follow [Customization Guide](dev-portal-customization.md)
4. **Configure Authentication** - Choose and implement authentication strategy
5. **Publish APIs** - Add APIs to products for developer access
6. **Create Content** - Add guides, FAQs, and onboarding materials

## Architecture Considerations

### Hosting

The Developer Portal is hosted by Azure APIM with options for:

- **Default URL** - `{apim-name}.developer.azure-api.net`
- **Custom Domain** - `developer.yourcompany.com`
- **Internal Mode** - Portal accessible only within VNET

### Deployment

Portal content and customization can be managed through:

- **Azure Portal** - Visual editor for live changes
- **ARM/Bicep** - Infrastructure as Code deployment
- **REST API** - Programmatic content management
- **Git Repository** - Version control for custom portal code

### Performance

The portal is a static site with CDN delivery:

- Content cached at edge locations globally
- API calls proxied through APIM gateway
- Responsive design for mobile devices
- SEO-friendly for public APIs

## Security Considerations

### Authentication

- **Enable HTTPS only** - Enforce secure connections
- **Choose appropriate identity provider** - Match enterprise requirements
- **Implement MFA** - For sensitive API access
- **Use product subscriptions** - Control API access granularly

### Content Security

- **Review custom content** - Sanitize user-provided HTML/JavaScript
- **Limit user permissions** - Principle of least privilege
- **Monitor access patterns** - Detect anomalous behavior

### API Protection

- **Require subscriptions** - Don't expose APIs without keys
- **Rate limit aggressively** - Protect against abuse
- **Hide sensitive APIs** - Use unpublished products for internal-only APIs

## Best Practices

### Design

1. **Keep navigation simple** - Developers should find APIs quickly
2. **Provide search** - Enable easy discovery of specific APIs
3. **Use consistent layouts** - Standard structure across pages
4. **Mobile-friendly** - Ensure responsive design

### Content

1. **Auto-generate from specs** - Keep documentation in sync with APIs
2. **Add examples** - Show request/response samples
3. **Provide quick-starts** - Help developers get started fast
4. **Include troubleshooting** - Common errors and solutions

### Operations

1. **Monitor portal usage** - Track which APIs are popular
2. **Update regularly** - Keep content fresh and accurate
3. **Gather feedback** - Provide channels for developer input
4. **Test changes** - Use staging environments before production (see [Environment Promotion](environment-promotion.md))

## Monitoring and Analytics

### Built-in Analytics

APIM provides analytics for:

- API call volumes and trends
- Response times and error rates
- Top consumers and APIs
- Geographic distribution

### Custom Analytics

Integration with Application Insights enables:

- Developer portal page views
- API console usage
- Subscription sign-up funnels
- Custom events and metrics

See [Monitoring and Observability](../monitoring-and-observability/) for comprehensive guidance.

## Migration and Upgrades

### Portal Versions

The Developer Portal has evolved through versions:

- **Legacy Portal** - Original portal (deprecated)
- **Current Portal** - Modern, open-source portal
- **Self-Hosted Option** - Custom deployments

### Migration Path

If migrating from legacy portal:

1. Review customizations in old portal
2. Identify equivalent features in new portal
3. Recreate customizations using visual editor or code
4. Test thoroughly before switching
5. Communicate change to developers

## Common Issues and Solutions

### Portal Not Loading

**Symptoms:** Blank page or loading errors

**Solutions:**
- Check if portal is published
- Verify custom domain DNS settings
- Review browser console for errors
- Clear browser cache

### Authentication Failures

**Symptoms:** Unable to sign in or redirect loops

**Solutions:**
- Verify identity provider configuration
- Check delegation URL and signature validation
- Ensure callback URLs are whitelisted
- Review Application Insights logs

### API Console Not Working

**Symptoms:** "Try It" button doesn't work

**Solutions:**
- Check CORS policy configuration
- Verify subscription keys are valid
- Ensure API is added to a product
- Check browser console for errors

## Related Topics

- **Azure APIM** - See [Azure APIM](../azure-apim/) for core concepts
- **Monitoring** - See [Monitoring and Observability](../monitoring-and-observability/) for portal analytics
- **DevOps** - See [DevOps](../dev-ops/) for portal deployment automation
