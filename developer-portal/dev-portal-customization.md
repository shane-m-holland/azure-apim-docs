# Azure API Management (APIM)

## Developer Portal Customization Options

You can **customize the portal** via the built-in visual editor or programmatically using GitHub and the REST API.

### Click-Ops Customization (No-Code)

- **Change branding**: Logo, colors, favicon
- **Edit page content**: Use drag-and-drop widgets
- **Add custom pages**: FAQs, onboarding guides, changelogs
- **Adjust layouts**: Modify page structure using drag-and-drop components.
- **Add content**: Create custom pages for guides, announcements, or FAQs.
- **Organize navigation**: Control menus and page hierarchy to simplify discovery.
- **Modify navigation**: Add menus, rename links

This approach provides flexibility without requiring a full fork of the portal.

### Programmatic Customization (Advanced)

- **Clone and edit source code** using GitHub
- Modify templates, scripts, and stylesheets
- Use REST API to deploy or sync changes
- Add JavaScript, HTML, CSS for enhanced interactivity

### Content Strategy

Good documentation is essential for adoption of APIs, particularly for non-technical consumers. Consider:

- **Clarity**: Use plain language and examples.
- **Consistency**: Standardize how APIs are described (naming conventions, parameters, error codes).
- **Guides**: Provide quick-start tutorials for common use cases.
- **Support information**: Clearly state how developers can request help or escalate issues.

### Interactive Features

The developer portal offers interactive capabilities that improve the developer experience:

- **API Console**: Users can authenticate and test APIs directly in the browser.
- **Try It Functionality**: Enables hands-on exploration without needing a custom client.
- **Sample Code**: Provide snippets in popular languages (C#, Python, JavaScript).
- **Subscription Management**: Developers can request, view, and regenerate subscription keys.

### Branding and Customization Basics

While deep customization requires forking the portal, most organizations achieve their goals with the built-in editor:

- Replace the default logo with the company brand.
- Configure themes to match corporate style guides.
- Add documentation for onboarding and API usage.
- Create landing pages for internal vs. external developers.

The result is a portal that aligns with organizational branding and provides a professional experience for API consumers.

## Useful Features to Enable

- **Try-it-out console**: Built-in API testbed with live requests
- **Role-based access**: Public, authenticated, and admin experiences
- **Multi-language support**: Built-in localization options
- **Custom domains**: Host the portal under your branded domain (e.g., `dev.yourcompany.com`)

## Official Microsoft Documentation

| Topic | Link |
|-------|------|
| Overview of Developer Portal | [https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-developer-portal](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-developer-portal) |
| Customize with Visual Editor | [https://learn.microsoft.com/en-us/azure/api-management/developer-portal-overview#visual-editor](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-overview#visual-editor) |
| Programmatic Customization (GitHub) | [https://learn.microsoft.com/en-us/azure/api-management/developer-portal-overview#code](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-overview#code) |
