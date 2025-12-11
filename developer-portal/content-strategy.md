# Content Strategy for Developer Portal

## Overview

A well-designed content strategy ensures developers can quickly understand, adopt, and successfully use your APIs. This guide provides best practices for creating effective developer portal content.

## Content Goals

### Primary Objectives

1. **Enable Self-Service** - Developers find answers without contacting support
2. **Accelerate Onboarding** - New users productive within minutes
3. **Reduce Support Burden** - Comprehensive documentation reduces tickets
4. **Increase Adoption** - Clear value propositions drive API usage
5. **Build Trust** - Professional presentation and accurate information

## Content Types

### API Documentation

**Auto-Generated from OpenAPI**
- Always up-to-date with API definitions
- Consistent format across all APIs
- Interactive try-it console
- Code samples in multiple languages

**Best Practices:**
- Include descriptions for all operations
- Document all parameters with examples
- Provide realistic request/response samples
- Explain error codes and handling

### Getting Started Guides

**Quick Start Tutorial**
```markdown
# Quick Start: Products API

Get your first API response in under 5 minutes.

## Prerequisites
- Developer Portal account
- Subscription to Starter product

## Steps

1. **Get Your API Key**
   - Sign in to Developer Portal
   - Navigate to Products > Starter
   - Click "Show" to reveal primary key

2. **Make Your First Request**
   ```bash
   curl -H "Ocp-Apim-Subscription-Key: YOUR-KEY" \
     https://api.example.com/products
   ```

3. **Understand the Response**
   ```json
   {
     "products": [...],
     "page": 1,
     "totalCount": 100
   }
   ```

## Next Steps
- Explore other endpoints
- Read authentication guide
- Review rate limits
```

### How-To Guides

Task-oriented documentation for common scenarios:

- **Authentication Setup** - Configuring OAuth tokens
- **Error Handling** - Dealing with common errors
- **Rate Limit Management** - Optimizing requests
- **Pagination** - Handling large result sets
- **Webhooks** - Setting up event notifications

### Troubleshooting Guides

Help developers solve common problems:

```markdown
# Troubleshooting: 401 Unauthorized

## Symptoms
API returns 401 Unauthorized error

## Common Causes

### 1. Missing Subscription Key
**Solution:** Add Ocp-Apim-Subscription-Key header
```bash
curl -H "Ocp-Apim-Subscription-Key: your-key" \
  https://api.example.com/endpoint
```

### 2. Expired OAuth Token
**Solution:** Refresh your access token

### 3. Invalid Subscription
**Solution:** Verify subscription is active in portal
```

### Release Notes and Changelog

Keep developers informed of changes:

```markdown
# API Changelog

## 2024-12-11 - Version 1.2.0

### Added
- New `/products/search` endpoint with full-text search
- Support for filtering by multiple categories

### Changed
- `/products` endpoint now includes inventory count
- Increased default page size from 20 to 50

### Deprecated
- `/products/query` endpoint (use `/products/search` instead)
- Will be removed in v2.0.0 (March 2025)

### Fixed
- Pagination links now correctly encode special characters
```

## Content Organization

### Navigation Structure

Organize content logically:

```
Developer Portal
├── Home
├── APIs
│   ├── Products API
│   ├── Orders API
│   └── Customers API
├── Documentation
│   ├── Getting Started
│   ├── Authentication
│   ├── Rate Limits
│   ├── Error Handling
│   └── Best Practices
├── Guides
│   ├── Quick Start Tutorials
│   ├── Integration Patterns
│   └── Use Case Examples
└── Support
    ├── FAQ
    ├── Troubleshooting
    └── Contact Us
```

### Search and Discovery

Enable effective search:
- Descriptive page titles
- Relevant keywords in content
- Clear headings and subheadings
- Cross-references between related topics

## Writing Guidelines

### Style and Tone

**Be Conversational but Professional**
```markdown
❌ "The utilization of the authentication mechanism necessitates..."
✅ "To authenticate, include your API key in the request header."
```

**Use Active Voice**
```markdown
❌ "The response is returned by the API"
✅ "The API returns the response"
```

**Write for Scanning**
- Short paragraphs (2-4 sentences)
- Bulleted lists for options
- Code examples for clarity
- Headings for section breaks

### Code Examples

**Provide Multiple Languages**

```markdown
# Example: Create Product

**cURL**
```bash
curl -X POST https://api.example.com/products \
  -H "Content-Type: application/json" \
  -H "Ocp-Apim-Subscription-Key: your-key" \
  -d '{"name":"Widget","price":29.99}'
```

**JavaScript (Node.js)**
```javascript
const axios = require('axios');

const response = await axios.post(
  'https://api.example.com/products',
  { name: 'Widget', price: 29.99 },
  { headers: { 'Ocp-Apim-Subscription-Key': 'your-key' } }
);
```

**Python**
```python
import requests

response = requests.post(
    'https://api.example.com/products',
    json={'name': 'Widget', 'price': 29.99},
    headers={'Ocp-Apim-Subscription-Key': 'your-key'}
)
```

**C#**
```csharp
var client = new HttpClient();
client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", "your-key");

var content = new StringContent(
    @"{""name"":""Widget"",""price"":29.99}",
    Encoding.UTF8,
    "application/json"
);

var response = await client.PostAsync(
    "https://api.example.com/products",
    content
);
```

### Visual Aids

Use diagrams for complex concepts:

- **Authentication Flows** - Sequence diagrams
- **Data Models** - Entity relationship diagrams
- **Integration Patterns** - Architecture diagrams
- **Request/Response Flow** - Flowcharts

## Interactive Elements

### API Console

Ensure try-it functionality works:
- Pre-fill example values
- Show full request/response
- Display generated code
- Handle authentication properly

### Embedded Videos

For complex topics:
- Keep videos under 5 minutes
- Provide transcript or summary
- Update as APIs evolve

## Maintenance Strategy

### Regular Reviews

Schedule content reviews:

**Monthly:**
- Check for outdated information
- Verify code examples still work
- Update screenshots if UI changed

**Quarterly:**
- Full content audit
- User feedback incorporation
- Analytics review

**After Major Releases:**
- Update all affected documentation
- Add release notes
- Update examples

### Version Control

Track documentation changes:
- Use git for custom content
- Tag versions alongside API versions
- Maintain changelog for docs

### Feedback Mechanisms

Collect developer feedback:

**Page-Level Feedback**
```html
<div class="feedback">
  <p>Was this helpful?</p>
  <button>Yes</button>
  <button>No</button>
</div>
```

**Support Channels**
- Link to support email/form
- Community forum or Slack channel
- GitHub issues for public APIs

## Measuring Success

### Key Metrics

Track these indicators:

**Engagement Metrics:**
- Page views per documentation page
- Time spent on getting started guide
- API console usage
- Search queries

**Success Metrics:**
- Support ticket reduction
- Time to first API call
- Developer activation rate
- API adoption rate

**Quality Metrics:**
- Search success rate (found relevant content)
- Feedback ratings (helpful/not helpful)
- Documentation-related support tickets

### Analytics Queries

```kusto
// Most viewed documentation pages
pageViews
| where name contains "docs"
| summarize Views = count() by name
| order by Views desc
| take 20

// Search effectiveness
customEvents
| where name == "Search"
| extend query = tostring(customDimensions.query)
| extend resultsFound = toint(customDimensions.resultsFound)
| summarize
    Searches = count(),
    AvgResults = avg(resultsFound),
    ZeroResults = countif(resultsFound == 0)
  by query
```

## Best Practices Summary

### Content Creation

1. **Start with user goals** - What does the developer need to accomplish?
2. **Provide context** - Why would they use this API?
3. **Show, don't just tell** - Code examples over descriptions
4. **Test everything** - All examples must work
5. **Keep it current** - Regular reviews and updates

### Organization

1. **Logical hierarchy** - Group related content
2. **Clear navigation** - Developers find what they need quickly
3. **Progressive disclosure** - Basic first, advanced later
4. **Consistent structure** - Similar pages follow same pattern

### Maintenance

1. **Own the content** - Assign responsibility
2. **Version documentation** - Align with API versions
3. **Accept feedback** - Easy way for developers to report issues
4. **Measure effectiveness** - Track metrics, improve continuously

## Related Documentation

- [Developer Portal Overview](dev-portal-overview.md) - Portal capabilities
- [Customization Guide](dev-portal-customization.md) - Modifying portal appearance
- [API Specification Management](../developer-workflow/api-specification-management.md) - OpenAPI best practices
