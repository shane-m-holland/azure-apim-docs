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

### Method 1: Automated Migration Script (Recommended)

The proper way to migrate developer portal content between API Management services is using the `scripts.v3/migrate.js` script from the [developer portal GitHub repository](https://github.com/Azure/api-management-developer-portal/blob/master/scripts.v3/migrate.js).

**Official Microsoft Documentation:** [Automate Developer Portal Deployments](https://learn.microsoft.com/en-us/azure/api-management/automate-portal-deployments)

#### Prerequisites

1. **Node.js and npm** installed locally
2. **Azure CLI** authenticated with appropriate permissions
3. **Access** to both source and destination APIM instances
4. **Backup** of destination portal (script removes existing content)

#### Important Warnings

⚠️ **The script removes all content from the destination portal before migration**  
⚠️ **Migration between classic tiers (Standard) and v2 tiers (Standard v2) is NOT supported**  
⚠️ **Migration between v2 tier instances is NOT supported**  
⚠️ **Always backup the destination environment before running the script**

#### Migration Steps

**Step 1: Clone the Developer Portal Repository**
```bash
git clone https://github.com/Azure/api-management-developer-portal.git
cd api-management-developer-portal
```

**Step 2: Install Dependencies**
```bash
npm install
```

**Step 3: Authenticate with Azure**
```bash
az login
az account set --subscription "your-subscription-id"
```

**Step 4: Run Migration Script**

For **managed portals** (with autopublish):
```bash
node ./scripts.v3/migrate.js \
  --sourceSubscriptionId "source-subscription-id" \
  --sourceResourceGroupName "source-rg" \
  --sourceServiceName "source-apim-name" \
  --destSubscriptionId "dest-subscription-id" \
  --destResourceGroupName "dest-rg" \
  --destServiceName "dest-apim-name" \
  --publishDest true
```

For **self-hosted portals** (manual publish required):
```bash
node ./scripts.v3/migrate.js \
  --sourceSubscriptionId "source-subscription-id" \
  --sourceResourceGroupName "source-rg" \
  --sourceServiceName "source-apim-name" \
  --destSubscriptionId "dest-subscription-id" \
  --destResourceGroupName "dest-rg" \
  --destServiceName "dest-apim-name"
```

**Step 5: Verify Migration**
- Navigate to destination APIM portal as administrator
- Verify all customizations, pages, and content migrated successfully
- For managed portals: check published site is live
- For self-hosted portals: manually publish following [self-hosting instructions](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-self-host)

#### What the Script Does

1. **Captures** portal content and media from source APIM service
2. **Removes** all portal content and media from destination APIM service
3. **Uploads** captured content and media to destination APIM service
4. **Publishes** portal automatically (managed portals only, if `--publishDest true`)

#### Special Case: Custom Storage Account

If using a self-hosted portal with an explicitly defined custom storage account (i.e., `blobStorageUrl` setting in `config.design.json`), use the [original scripts.v3/migrate.js script](https://github.com/Azure/api-management-developer-portal/blob/master/scripts.v3/migrate.js). Note that the original script does not work for managed portals or self-hosted portals with storage managed by API Management.

### Workaround: Standard (Classic) to V2 SKU Migration

⚠️ **Important:** The automated migration script does NOT support migrating portal content from Standard (classic tier) to Standard v2 or other V2 SKU instances. This is a known limitation.

If you need to promote portal content from a Standard APIM instance to a V2 APIM instance (common scenario: QA/Stage on Standard, Production on V2), use one of the following workarounds:

#### Option 1: Manual Recreation (Recommended for Simple Customizations)

**Best for:** Portals with basic branding, limited custom pages, and minimal customization.

**Steps:**

1. **Document source customizations:**
   - Take screenshots of all custom pages
   - Export branding assets (logos, favicon, color schemes)
   - Document custom navigation structure
   - Save any custom HTML/CSS/JavaScript

2. **Recreate in V2 environment:**
   - Open V2 APIM portal administrative interface
   - Use visual editor to apply branding (logo, colors)
   - Recreate custom pages using drag-and-drop widgets
   - Rebuild navigation menus
   - Add custom code if needed

3. **Test and validate:**
   - Compare with source portal using screenshots
   - Test all functionality
   - Verify API documentation displays correctly

**Pros:** Simple, no scripting required  
**Cons:** Time-consuming for complex portals, risk of missing customizations

#### Option 2: Self-Hosted Portal Migration Bridge

**Best for:** Complex customizations that are difficult to recreate manually.

**Strategy:** Use a temporary Standard tier instance as a migration bridge.

**Steps:**

1. **Create temporary Standard tier APIM instance:**
   ```bash
   az apim create \
     --name temp-migration-apim \
     --resource-group temp-rg \
     --location eastus \
     --publisher-name "Your Org" \
     --publisher-email "admin@yourorg.com" \
     --sku-name Standard
   ```

2. **Migrate from source Standard to temporary Standard:**
   ```bash
   # This works because both are Standard tier
   cd api-management-developer-portal
   
   node ./scripts.v3/migrate.js \
     --sourceSubscriptionId "source-subscription-id" \
     --sourceResourceGroupName "source-rg" \
     --sourceServiceName "source-standard-apim" \
     --destSubscriptionId "temp-subscription-id" \
     --destResourceGroupName "temp-rg" \
     --destServiceName "temp-migration-apim" \
     --publishDest true
   ```

3. **Use self-hosted portal approach:**
   - Clone developer portal repository
   - Configure to connect to temporary Standard instance
   - Generate static files
   - Deploy static files to Azure Storage or CDN
   - Point V2 APIM to self-hosted portal
   - See [self-hosting documentation](https://learn.microsoft.com/en-us/azure/api-management/developer-portal-self-host)

4. **Clean up temporary resources:**
   ```bash
   az apim delete --name temp-migration-apim --resource-group temp-rg --yes --no-wait
   az group delete --name temp-rg --yes --no-wait
   ```

**Pros:** Preserves all customizations  
**Cons:** Requires self-hosted portal setup, additional infrastructure, temporary costs

#### Option 3: REST API Content Export/Recreation

**Best for:** Teams with development resources who need automated, repeatable migrations.

**Strategy:** Export content via REST API, transform if needed, recreate in V2 environment.

**Steps:**

1. **Export content from Standard tier:**
   ```bash
   # Get access token
   ACCESS_TOKEN=$(az account get-access-token --resource https://management.azure.com --query accessToken -o tsv)
   
   # Set variables
   SOURCE_SUB_ID="source-subscription-id"
   SOURCE_RG="source-rg"
   SOURCE_APIM="source-apim-name"
   API_VERSION="2021-08-01"
   
   # Export pages
   curl -X GET \
     "https://management.azure.com/subscriptions/${SOURCE_SUB_ID}/resourceGroups/${SOURCE_RG}/providers/Microsoft.ApiManagement/service/${SOURCE_APIM}/contentTypes/page/contentItems?api-version=${API_VERSION}" \
     -H "Authorization: Bearer ${ACCESS_TOKEN}" \
     > pages-export.json
   
   # Export other content types (document, layout, media, etc.)
   for contentType in document layout media; do
     curl -X GET \
       "https://management.azure.com/subscriptions/${SOURCE_SUB_ID}/resourceGroups/${SOURCE_RG}/providers/Microsoft.ApiManagement/service/${SOURCE_APIM}/contentTypes/${contentType}/contentItems?api-version=${API_VERSION}" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       > ${contentType}-export.json
   done
   ```

2. **Create import script for V2:**
   ```bash
   # Set V2 variables
   V2_SUB_ID="v2-subscription-id"
   V2_RG="v2-rg"
   V2_APIM="v2-apim-name"
   
   # Import pages to V2 tier
   # Note: May require transformation for compatibility
   for item in $(jq -r '.value[].name' pages-export.json); do
     ITEM_DATA=$(jq ".value[] | select(.name == \"${item}\")" pages-export.json)
     
     echo "${ITEM_DATA}" | curl -X PUT \
       "https://management.azure.com/subscriptions/${V2_SUB_ID}/resourceGroups/${V2_RG}/providers/Microsoft.ApiManagement/service/${V2_APIM}/contentTypes/page/contentItems/${item}?api-version=${API_VERSION}" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       -H "Content-Type: application/json" \
       -d @-
   done
   ```

3. **Test and adjust:**
   - Verify content imported correctly
   - Fix any compatibility issues
   - Republish portal in V2 environment

**Pros:** Can be automated, repeatable, scriptable  
**Cons:** Complex, may require content transformation, not officially supported, requires testing

#### Option 4: Version Control + Infrastructure as Code Approach

**Best for:** Organizations using Infrastructure as Code for portal management.

**Strategy:** Store portal configuration in source control, apply to any environment.

**Steps:**

1. **Store customizations in source control:**
   - Branding configuration files
   - Custom page definitions (JSON)
   - Custom CSS/JavaScript files
   - ARM/Bicep templates for portal configuration

2. **Use ARM/Bicep to deploy base configuration:**
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
       delegation: {
         delegateRegistration: false
         delegateSubscription: false
       }
     }
   }
   ```

3. **Apply customizations via CI/CD pipeline:**
   - Deploy to both Standard and V2 environments from same source
   - Maintain environment parity through code
   - Version control all changes

**Pros:** Best long-term approach, environment-agnostic, version controlled, repeatable  
**Cons:** Requires upfront investment, ongoing maintenance, learning curve

#### Recommendation for Standard (QA/Stage) → V2 (Production) Scenario

For environments where **QA/Stage runs on Standard** and **Production runs on V2**:

**Initial Production Setup:**
- Use **Option 1 (Manual Recreation)** for first deployment to V2 Production
- Document the process thoroughly with screenshots and checklists

**Ongoing Changes:**
1. Make and test changes in Standard (QA/Stage) environment
2. Document all changes with screenshots and notes
3. Apply same changes manually to V2 (Production) using visual editor
4. Maintain a change log to track what's been promoted

**Long-term Strategy (Choose one):**

**Path A: Migrate Lower Environments to V2**
- Upgrade QA/Stage to V2 SKU when budget allows
- Enables use of automated migration script across all environments
- Simplifies operations and reduces manual work

**Path B: Adopt Infrastructure as Code**
- Move to **Option 4 (IaC approach)** for long-term maintainability
- Store all portal configurations in Git
- Deploy to both Standard and V2 from same source
- Enables consistent, repeatable deployments

**Interim Process:**
```bash
# Example workflow for each change

# 1. Test in QA/Stage (Standard)
# - Make changes via visual editor
# - Test thoroughly
# - Document with screenshots

# 2. Prepare for Production (V2)
# - Create change checklist from documentation
# - Schedule maintenance window if needed
# - Have rollback plan (re-create previous state)

# 3. Apply to Production (V2)
# - Open V2 APIM portal admin interface
# - Apply changes following checklist
# - Test each change before moving to next
# - Publish portal
# - Validate with smoke tests
```

### Method 2: REST API for Advanced Custom Automation

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
# Set environment variables
export QA_SUBSCRIPTION_ID="qa-subscription-id"
export QA_RG="qa-resource-group"
export QA_APIM="qa-apim-name"
export STAGE_SUBSCRIPTION_ID="stage-subscription-id"
export STAGE_RG="stage-resource-group"
export STAGE_APIM="stage-apim-name"

# Navigate to the developer portal repository
cd api-management-developer-portal

# Run migration script from QA to Stage
node ./scripts.v3/migrate.js \
  --sourceSubscriptionId ${QA_SUBSCRIPTION_ID} \
  --sourceResourceGroupName ${QA_RG} \
  --sourceServiceName ${QA_APIM} \
  --destSubscriptionId ${STAGE_SUBSCRIPTION_ID} \
  --destResourceGroupName ${STAGE_RG} \
  --destServiceName ${STAGE_APIM} \
  --publishDest true
```

**Note:** The script automatically backs up by capturing source content, removes destination content, and uploads. For additional safety, document the Stage portal state before promotion.

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
# Set environment variables
export STAGE_SUBSCRIPTION_ID="stage-subscription-id"
export STAGE_RG="stage-resource-group"
export STAGE_APIM="stage-apim-name"
export PROD_SUBSCRIPTION_ID="prod-subscription-id"
export PROD_RG="prod-resource-group"
export PROD_APIM="prod-apim-name"

# Navigate to the developer portal repository
cd api-management-developer-portal

# Run migration script from Stage to Production
node ./scripts.v3/migrate.js \
  --sourceSubscriptionId ${STAGE_SUBSCRIPTION_ID} \
  --sourceResourceGroupName ${STAGE_RG} \
  --sourceServiceName ${STAGE_APIM} \
  --destSubscriptionId ${PROD_SUBSCRIPTION_ID} \
  --destResourceGroupName ${PROD_RG} \
  --destServiceName ${PROD_APIM} \
  --publishDest true
```

**Critical:** The script removes all destination content. Ensure you have documented the production portal state and have a tested rollback procedure before executing.

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

### Rollback Strategy

Since the migration script removes all destination content, rollback requires re-running the migration from the previous working environment.

**Option 1: Roll Forward (Preferred)**
Fix the issue in the source environment and re-run the migration script.

**Option 2: Rollback to Previous Environment**
Re-run the migration script from the last known good environment.

#### Rollback Example: Revert Production to Stage

If issues are detected after Stage → Production promotion:

```bash
# Set environment variables (reverse direction)
export STAGE_SUBSCRIPTION_ID="stage-subscription-id"
export STAGE_RG="stage-resource-group"
export STAGE_APIM="stage-apim-name"
export PROD_SUBSCRIPTION_ID="prod-subscription-id"
export PROD_RG="prod-resource-group"
export PROD_APIM="prod-apim-name"

# Navigate to the developer portal repository
cd api-management-developer-portal

# Re-run migration from Stage to Production
# This restores Production to match Stage (the last known good state)
node ./scripts.v3/migrate.js \
  --sourceSubscriptionId ${STAGE_SUBSCRIPTION_ID} \
  --sourceResourceGroupName ${STAGE_RG} \
  --sourceServiceName ${STAGE_APIM} \
  --destSubscriptionId ${PROD_SUBSCRIPTION_ID} \
  --destResourceGroupName ${PROD_RG} \
  --destServiceName ${PROD_APIM} \
  --publishDest true
```

**Important:** This assumes Stage still has the previous working portal content. If Stage was also affected, you may need to promote from QA.

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
