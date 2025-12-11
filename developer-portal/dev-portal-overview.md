# Azure API Management (APIM)

## Developer Portal

The **Developer Portal** in Azure API Management (APIM) is an automatically generated, consumer-facing web application that provides a centralized place for developers, partners, and customers to discover, learn about, and consume APIs published through APIM.

It serves as the primary interface for onboarding API consumers, offering documentation, testing tools, and subscription management in a customizable, branded experience.

It provides a **self-service platform** where developers can:

- Provide interactive documentation of APIs.
- Allow developers to discover APIs and learn how to use them.
- Enable testing of APIs directly from the browser.
- Support onboarding with subscription and access management.
- Allow organizations to brand and customize the experience.

*Built on a **headless CMS**, the portal is **open-source** and can be customized via a visual editor or programmatically.*

Click here for details on customizing the [Developer Portal](./dev-portal-customization.md)

## Key Features of the Developer Portal

- **API Documentation**
  - Auto-generated from imported OpenAPI specifications.
  - Includes details on endpoints, parameters, responses, and error codes.

- **Interactive API Console**
  - “Try It” functionality that allows developers to send test requests directly from the portal.
  - Supports authentication and subscription keys for secure testing.

- **Self-Service Access**
  - Developers can browse available APIs.
  - Consumers can request access to products and manage their subscription keys.

- **Custom Branding**
  - Customizable UI: logos, colors, and layouts to align with organizational branding.
  - Ability to add custom pages (FAQs, onboarding guides, or announcements).

- **User Management**
  - Authentication integration with Azure AD, Azure AD B2C, or social providers.
  - Role-based access (e.g., developers, administrators, guests).

---

### Typical Use Cases

| Use Case | Example |
|----------|---------|
| **API Discovery** | A partner developer browses public APIs your company offers. |
| **Documentation Hosting** | Internal developers view OpenAPI (Swagger) definitions and endpoint guides. |
| **Testing APIs** | Try-it-out console lets devs call APIs in real time. |
| **Subscription Management** | External users sign up and request access to specific API products. |
| **Developer Onboarding** | Teams use the portal as a landing page with tutorials, FAQs, and support. |

---

## Best Practices & Tips

### Design & Branding

- Align the portal with organizational branding guidelines (colors, fonts, logos).
- Keep navigation simple and intuitive for API discovery.
- Provide onboarding documentation for new developers.

### Documentation

- Ensure all APIs have up-to-date OpenAPI specifications.
- Include examples for requests and responses.
- Document error codes and usage limits clearly.

### Developer Experience

- Offer quick-start guides and sample code snippets (e.g., cURL, C#, Python, JavaScript).
- Provide interactive “Try It” functionality for hands-on learning.
- Make subscription management (requesting/regenerating keys) simple and self-service.

### Security & Access

- Integrate with your identity provider (Azure AD, Azure AD B2C, etc.) for user authentication.
- Control access by grouping APIs into **Products** with subscription requirements.
- Enforce usage policies (quotas, rate limits) to protect backends.

### Governance & Maintenance

- Keep the portal updated as APIs evolve (revisions, versions).
- Regularly audit which APIs are published and ensure they are still relevant.
- Monitor developer sign-ups and API consumption patterns to improve adoption.
