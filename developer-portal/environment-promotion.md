# Developer Portal Environment Promotion

## Overview

This guide provides best practices and step-by-step procedures for promoting Developer Portal customizations across environments (QA → Stage → Production). Proper environment promotion ensures changes are thoroughly tested before reaching production users.

## Environment Strategy

### Recommended Environment Structure

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│     QA      │ --> │    Stage    │ --> │  Production │
│ Development │     │   Testing   │     │    Live     │
│  & Testing  │     │ & Validation│     │   Users     │
└─────────────┘     └─────────────┘     └─────────────┘
```

**QA Environment:**
- Development and initial testing
- Frequent changes and iterations
- Used by developers and QA teams
- Low risk for breaking changes

**Stage Environment:**
- Pre-production validation
- Mirrors production configuration
- Used for UAT and final testing
- Integration testing with other systems

**Production Environment:**
- Live user traffic
- Changes only after thorough testing
- Strict change control
- Rollback capability required

## Portal Content and Customizations

### What Needs to be Promoted

**Portal Customizations:**
- Branding (logos, colors, favicon, fonts)
- Page layouts and widgets
- Custom pages and content
- Navigation menus and structure
- Custom CSS and JavaScript

**Configuration:**
- Authentication settings (identity providers)
- Product visibility and access rules
- Custom domain configurations
- CORS policies

**Content:**
- API documentation pages
- Getting started guides
- Tutorials and how-tos
- FAQs and support pages

### What is Environment-Specific

**Do NOT promote these items** (configure separately per environment):
- APIM instance URLs
- API backend endpoints
- Subscription keys
- OAuth client IDs and secrets
- Custom domain DNS settings
- Environment-specific feature flags

## Promotion Methods

### Method 1: Manual Content Export/Import

Use for simple customizations made via the visual editor.

#### Step 1: Export from Source Environment

**Option A: Via Azure Portal**
1. Navigate to APIM instance
2. Go to Developer Portal → Portal Overview
3. Click "Export website" button
4. Save the exported JSON file

**Option B: Via Azure CLI**
```bash
# Export portal content
az rest --method get \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.ApiManagement/service/{apimServiceName}/contentTypes?api-version=2021-08-01" \
  --output-file portal-content.json
```

#### Step 2: Review and Test Export

1. Review the exported content for environment-specific values
2. Update any hardcoded URLs or references
3. Document any manual configuration needed

#### Step 3: Import to Target Environment

**Via Azure Portal:**
1. Navigate to target APIM instance
2. Go to Developer Portal → Portal Overview
3. Click "Import website"
4. Upload the exported JSON file
5. Confirm the import operation

**Via Azure CLI:**
```bash
# Import portal content
az rest --method put \
  --url "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.ApiManagement/service/{apimServiceName}/contentTypes?api-version=2021-08-01" \
  --body @portal-content.json
```

#### Step 4: Republish Portal

After import, republish the portal:

```bash
# Publish the developer portal
az apim api-portal publish \
  --resource-group {resourceGroup} \
  --service-name {apimServiceName}
```

### Method 2: REST API Automation

Use for repeatable, automated deployments.

#### Prerequisites

```bash
# Get access token
az account get-access-token --resource https://management.azure.com

# Set environment variables
export ACCESS_TOKEN="your-token"
export SOURCE_APIM="source-apim-name"
export TARGET_APIM="target-apim-name"
export RESOURCE_GROUP="your-resource-group"
export SUBSCRIPTION_ID="your-subscription-id"
```

#### Export Content Items

```bash
# List all content types
curl -X GET \
  "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${SOURCE_APIM}/contentTypes?api-version=2021-08-01" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# Export specific content type (e.g., pages)
curl -X GET \
  "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${SOURCE_APIM}/contentTypes/page/contentItems?api-version=2021-08-01" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  > pages-export.json
```

#### Import Content Items

```bash
# Import content to target environment
curl -X PUT \
  "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${TARGET_APIM}/contentTypes/page/contentItems/{itemId}?api-version=2021-08-01" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @page-item.json
```

### Method 3: Infrastructure as Code (IaC)

Use for version-controlled, repeatable deployments.

#### ARM Template Example

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "apimServiceName": {
      "type": "string"
    }
  },
  "resources": [
    {
      "type": "Microsoft.ApiManagement/service/portalConfigs",
      "apiVersion": "2021-08-01",
      "name": "[concat(parameters('apimServiceName'), '/default')]",
      "properties": {
        "enableBasicAuth": true,
        "signin": {
          "require": false
        },
        "signup": {
          "termsOfService": {
            "requireConsent": true
          }
        },
        "delegation": {
          "delegateRegistration": false,
          "delegateSubscription": false
        }
      }
    }
  ]
}
```

#### Bicep Example

```bicep
resource apimPortalConfig 'Microsoft.ApiManagement/service/portalConfigs@2021-08-01' = {
  name: '${apimServiceName}/default'
  properties: {
    enableBasicAuth: true
    signin: {
      require: false
    }
    signup: {
      termsOfService: {
        requireConsent: true
      }
    }
  }
}
```

### Method 4: Programmatic Portal (Self-Hosted)

For advanced customization using the open-source portal code.

#### Setup CI/CD Pipeline

```yaml
# Azure DevOps Pipeline Example
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - developer-portal/*

stages:
- stage: Build
  jobs:
  - job: BuildPortal
    steps:
    - script: |
        cd developer-portal
        npm install
        npm run build
      displayName: 'Build Developer Portal'
    
    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: 'developer-portal/dist'
        artifactName: 'portal'

- stage: DeployToQA
  dependsOn: Build
  jobs:
  - deployment: DeployQA
    environment: 'QA'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              az apim api-portal create-or-update \
                --resource-group $(QA_RESOURCE_GROUP) \
                --service-name $(QA_APIM_NAME) \
                --portal-content $(Pipeline.Workspace)/portal
            displayName: 'Deploy to QA'

- stage: DeployToStage
  dependsOn: DeployToQA
  jobs:
  - deployment: DeployStage
    environment: 'Stage'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              az apim api-portal create-or-update \
                --resource-group $(STAGE_RESOURCE_GROUP) \
                --service-name $(STAGE_APIM_NAME) \
                --portal-content $(Pipeline.Workspace)/portal
            displayName: 'Deploy to Stage'

- stage: DeployToProd
  dependsOn: DeployToStage
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployProd
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              az apim api-portal create-or-update \
                --resource-group $(PROD_RESOURCE_GROUP) \
                --service-name $(PROD_APIM_NAME) \
                --portal-content $(Pipeline.Workspace)/portal
            displayName: 'Deploy to Production'
```

## Promotion Workflow

### QA → Stage Promotion

#### 1. Pre-Promotion Checklist

- [ ] All QA testing completed and passed
- [ ] Visual regression testing completed
- [ ] Custom JavaScript/CSS tested
- [ ] Authentication flows validated
- [ ] API console functionality verified
- [ ] Mobile responsive design checked
- [ ] Browser compatibility tested
- [ ] Accessibility requirements met
- [ ] Change documented in release notes
- [ ] Rollback plan prepared

#### 2. Execute Promotion

```bash
# Export from QA
az rest --method get \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${QA_RG}/providers/Microsoft.ApiManagement/service/${QA_APIM}/contentTypes?api-version=2021-08-01" \
  --output-file qa-portal-export.json

# Backup Stage (before import)
az rest --method get \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${STAGE_RG}/providers/Microsoft.ApiManagement/service/${STAGE_APIM}/contentTypes?api-version=2021-08-01" \
  --output-file stage-portal-backup.json

# Import to Stage
az rest --method put \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${STAGE_RG}/providers/Microsoft.ApiManagement/service/${STAGE_APIM}/contentTypes?api-version=2021-08-01" \
  --body @qa-portal-export.json

# Publish Stage portal
az apim api-portal publish \
  --resource-group ${STAGE_RG} \
  --service-name ${STAGE_APIM}
```

#### 3. Post-Promotion Testing

- [ ] Portal loads without errors
- [ ] All pages render correctly
- [ ] Navigation menus function properly
- [ ] API console works with Stage APIs
- [ ] Authentication/sign-in functional
- [ ] Custom content displays correctly
- [ ] Search functionality operational
- [ ] Forms and interactive elements work
- [ ] Performance acceptable (page load times)

### Stage → Production Promotion

#### 1. Pre-Production Checklist

- [ ] Stage testing completed successfully
- [ ] UAT sign-off received
- [ ] Security review completed
- [ ] Performance testing passed
- [ ] Load testing completed (if applicable)
- [ ] Stakeholder approval obtained
- [ ] Maintenance window scheduled (if needed)
- [ ] Communication plan executed
- [ ] Rollback procedure tested
- [ ] Support team notified

#### 2. Execute Production Promotion

```bash
# Backup Production (critical!)
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
az rest --method get \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${PROD_RG}/providers/Microsoft.ApiManagement/service/${PROD_APIM}/contentTypes?api-version=2021-08-01" \
  --output-file "prod-portal-backup-${BACKUP_DATE}.json"

# Export from Stage
az rest --method get \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${STAGE_RG}/providers/Microsoft.ApiManagement/service/${STAGE_APIM}/contentTypes?api-version=2021-08-01" \
  --output-file stage-portal-export.json

# Import to Production
az rest --method put \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${PROD_RG}/providers/Microsoft.ApiManagement/service/${PROD_APIM}/contentTypes?api-version=2021-08-01" \
  --body @stage-portal-export.json

# Publish Production portal
az apim api-portal publish \
  --resource-group ${PROD_RG} \
  --service-name ${PROD_APIM}
```

#### 3. Post-Production Validation

- [ ] Portal accessible at production URL
- [ ] All critical paths tested
- [ ] Monitoring alerts reviewed
- [ ] Error rates within acceptable range
- [ ] Performance metrics acceptable
- [ ] User feedback channels monitored
- [ ] Support tickets reviewed

## Testing Strategy

### Visual Regression Testing

Use automated tools to catch unintended visual changes:

```javascript
// Example using Playwright
const { test, expect } = require('@playwright/test');

test('Developer Portal Homepage', async ({ page }) => {
  await page.goto('https://qa-apim.developer.azure-api.net');
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100
  });
});

test('API Documentation Page', async ({ page }) => {
  await page.goto('https://qa-apim.developer.azure-api.net/apis/products-api');
  await expect(page).toHaveScreenshot('api-docs.png', {
    maxDiffPixels: 100
  });
});
```

### Functional Testing Checklist

**Authentication & Authorization:**
- [ ] Sign-up flow works
- [ ] Sign-in with username/password
- [ ] SSO/OIDC authentication
- [ ] Sign-out functionality
- [ ] Password reset flow

**API Discovery & Documentation:**
- [ ] API catalog loads
- [ ] Search functionality works
- [ ] Filtering by category/tag
- [ ] API details page displays
- [ ] OpenAPI specification renders

**API Console (Try It):**
- [ ] Console loads for each API
- [ ] Parameters can be entered
- [ ] Authentication works
- [ ] Requests execute successfully
- [ ] Responses display correctly
- [ ] Error messages are clear

**Subscription Management:**
- [ ] Product listing displays
- [ ] Subscribe to product works
- [ ] Subscription keys visible
- [ ] Regenerate key functionality
- [ ] Subscription approval (if enabled)

**Custom Content:**
- [ ] Custom pages load
- [ ] Navigation menus correct
- [ ] Links functional
- [ ] Forms submit correctly
- [ ] Downloads work

### Performance Testing

```bash
# Load test portal homepage
ab -n 1000 -c 10 https://stage-apim.developer.azure-api.net/

# Lighthouse audit
lighthouse https://stage-apim.developer.azure-api.net/ \
  --output html \
  --output-path ./lighthouse-report.html

# Check page load time
curl -w "@curl-format.txt" -o /dev/null -s \
  https://stage-apim.developer.azure-api.net/
```

## Rollback Procedures

### Quick Rollback (Using Backup)

If issues are detected after promotion:

```bash
# Restore from backup
az rest --method put \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${PROD_RG}/providers/Microsoft.ApiManagement/service/${PROD_APIM}/contentTypes?api-version=2021-08-01" \
  --body @prod-portal-backup-${BACKUP_DATE}.json

# Republish
az apim api-portal publish \
  --resource-group ${PROD_RG} \
  --service-name ${PROD_APIM}
```

### Rollback Decision Criteria

Initiate rollback if:
- Portal is inaccessible (5xx errors)
- Critical functionality broken (can't sign in, can't subscribe)
- Performance degradation >50%
- Security vulnerability introduced
- Data loss or corruption

Do NOT rollback for:
- Minor visual inconsistencies
- Non-critical content issues
- Performance degradation <20%
- Issues affecting <5% of users

## Best Practices

### Change Management

1. **Version Control Everything**
   - Store portal customizations in Git
   - Tag releases with semantic versioning
   - Document changes in CHANGELOG.md

2. **Maintain Environment Parity**
   - Keep Stage as close to Production as possible
   - Use same APIM tier across environments
   - Match custom domain configurations

3. **Test Thoroughly**
   - Never skip Stage testing
   - Include regression testing
   - Test edge cases and error scenarios

4. **Automate Where Possible**
   - Use CI/CD pipelines for promotion
   - Automate testing and validation
   - Script backup and rollback procedures

5. **Document Everything**
   - Maintain runbooks for promotion
   - Document environment differences
   - Keep release notes up to date

### Communication

**Before Changes:**
- Notify stakeholders of upcoming changes
- Schedule maintenance windows if needed
- Communicate expected downtime (if any)

**During Deployment:**
- Provide status updates
- Notify of any issues immediately
- Keep communication channels open

**After Deployment:**
- Confirm successful deployment
- Share validation results
- Document any issues encountered

### Monitoring

Set up monitoring for:

```kusto
// Portal availability
requests
| where name contains "developer.azure-api.net"
| summarize
    TotalRequests = count(),
    FailedRequests = countif(resultCode >= 400),
    AvgDuration = avg(duration)
  by bin(timestamp, 5m)

// Portal errors
exceptions
| where cloud_RoleName contains "developer-portal"
| summarize ErrorCount = count() by problemId, outerMessage
| order by ErrorCount desc

// Page load performance
pageViews
| where name contains "developer-portal"
| summarize
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 95)
  by name
```

## Troubleshooting

### Portal Not Updating After Import

**Symptoms:** Changes imported but not visible in portal

**Solutions:**
1. Ensure portal was republished after import
2. Clear browser cache and hard refresh
3. Check CDN cache expiration
4. Verify content was actually imported (check via API)

### Content Conflicts

**Symptoms:** Import fails with conflict errors

**Solutions:**
1. Export target environment first
2. Compare content IDs between environments
3. Delete conflicting items before import
4. Use REST API to import individual items

### Authentication Breaks After Promotion

**Symptoms:** Can't sign in after portal update

**Solutions:**
1. Verify identity provider configuration
2. Check callback URLs match environment
3. Ensure delegation URLs are correct
4. Review CORS settings

### Custom JavaScript Errors

**Symptoms:** Console errors after promotion

**Solutions:**
1. Check for environment-specific URLs in code
2. Verify external dependencies are accessible
3. Test in multiple browsers
4. Review Content Security Policy settings

## Related Documentation

- [Developer Portal Overview](dev-portal-overview.md) - Portal capabilities
- [Customization Guide](dev-portal-customization.md) - Modifying portal appearance
- [DevOps Best Practices](../dev-ops/) - CI/CD and automation guidance
- [Monitoring and Observability](../monitoring-and-observability/) - Portal analytics
