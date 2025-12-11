# Environment Management

## Overview

Environment management involves maintaining separate APIM instances and configurations for development, testing, and production. This guide covers strategies, patterns, and best practices for managing multiple environments.

## Environment Types

### Development (Dev)

**Purpose:** Rapid iteration and testing of changes

**Characteristics:**
- Lowest cost SKU (Developer or Consumption)
- Verbose logging enabled
- Short retention periods
- Automatic deployments on commit
- No approval gates
- Test data and backend services

**Configuration:**
```yaml
environment: dev
sku: Developer
capacity: 1
logging:
  sampling: 100%
  verbosity: verbose
retention: 7 days
approvals: none
```

### Testing/Staging (Test)

**Purpose:** Pre-production validation and QA

**Characteristics:**
- Standard SKU
- Production-like configuration
- Moderate logging
- Manual deployment trigger
- Optional approval
- Staging backend services

**Configuration:**
```yaml
environment: test
sku: Standard
capacity: 1
logging:
  sampling: 50%
  verbosity: information
retention: 30 days
approvals: optional
```

### Production (Prod)

**Purpose:** Live customer-facing environment

**Characteristics:**
- Premium or Standard SKU
- Optimized for performance
- Sampling-based logging
- Gated deployments
- Required approvals
- Production backend services
- Multi-region (Premium)

**Configuration:**
```yaml
environment: prod
sku: Premium
capacity: 2
logging:
  sampling: 20%
  verbosity: warning
retention: 90 days
approvals: required
reviewers: 2
```

## Configuration Management

### Parameter Files

Separate `.bicepparam` or `.json` files per environment:

**Structure:**
```
environments/
├── dev.bicepparam
├── test.bicepparam
└── prod.bicepparam
```

**Example:** `dev.bicepparam`

```bicep
using '../bicep/main.bicep'

param apimName = 'apim-dev-eastus2'
param location = 'eastus2'
param skuName = 'Developer'
param skuCapacity = 1
param publisherEmail = 'dev-team@company.com'
param publisherName = 'Company Dev'
param backendUrl = 'https://backend-dev.company.com'
```

**Example:** `prod.bicepparam`

```bicep
using '../bicep/main.bicep'

param apimName = 'apim-prod-eastus2'
param location = 'eastus2'
param skuName = 'Premium'
param skuCapacity = 2
param publisherEmail = 'api-ops@company.com'
param publisherName = 'Company'
param backendUrl = 'https://backend-prod.company.com'
```

### Environment Variables

**File:** `environments/dev.env`

```bash
# Azure Resources
RESOURCE_GROUP=rg-apim-dev
APIM_SERVICE_NAME=apim-dev-eastus2
LOCATION=eastus2

# Backend Services
WEATHER_SERVICE_URL=https://weather-dev.company.com
USERS_SERVICE_URL=https://users-dev.company.com
ORDERS_SERVICE_URL=https://orders-dev.company.com

# Feature Flags
ENABLE_CACHING=false
ENABLE_RATE_LIMITING=true
RATE_LIMIT=1000

# Monitoring
LOG_ANALYTICS_WORKSPACE_ID=<workspace-id>
APP_INSIGHTS_KEY=<instrumentation-key>
```

## Environment Isolation

### Network Isolation

**Development:** Public endpoints, open access

**Production:** Private endpoints, VNET integration

```bicep
// Production network configuration
param enableVnetIntegration = true
param vnetName = 'vnet-apim-prod'
param subnetName = 'snet-apim'
param nsgName = 'nsg-apim-prod'

// Development - no VNET
param enableVnetIntegration = false
```

### Access Control

**Development:**
- Broad access for developers
- Shared subscriptions
- Minimal authentication

**Production:**
- Restricted access
- Per-consumer subscriptions
- Strong authentication (OAuth, mutual TLS)
- Role-based access control

### Data Isolation

**Development:**
- Test data only
- Synthetic users
- Mock backends allowed

**Production:**
- Real customer data
- Production backends
- Strict data protection

## Promotion Strategy

### Environment Promotion Flow

```
Developer → commit → CI validates
         ↓
   Auto-deploy to Dev
         ↓
      Test in Dev
         ↓
   Create PR to Main
         ↓
   Approval required
         ↓
   Merge to Main
         ↓
   Auto-deploy to Prod
         ↓
   Smoke tests
         ↓
   Monitor metrics
```

### Configuration Promotion

**Do:**
- Promote validated configs
- Use git to track changes
- Test in lower environments first
- Automate promotion

**Don't:**
- Manually copy configs
- Skip testing in lower environments
- Promote untested changes
- Make prod-only changes

## Environment-Specific Features

### Development Environment

**Enable:**
- Verbose logging
- Test console in portal
- Mock responses
- Policy debugging
- CORS for local development

**Disable:**
- Rate limiting (or set high)
- Caching (for testing changes)
- Approval workflows

### Production Environment

**Enable:**
- Caching policies
- Rate limiting
- Throttling
- Multi-region deployment
- Auto-scaling
- Comprehensive monitoring

**Disable:**
- Test console (optional)
- Verbose logging
- Debug endpoints

## Secrets Management

### Development

**Approach:** Can use less secure methods

- Shared Key Vault
- GitHub secrets
- Environment variables in CI/CD

### Production

**Approach:** Must use secure methods

- Dedicated Key Vault per environment
- Managed identities
- RBAC on Key Vault
- Secrets rotation policies
- Audit logging

**Example:**

```bicep
// Reference Key Vault secret in APIM
resource namedValue 'Microsoft.ApiManagement/service/namedValues@2023-03-01-preview' = {
  parent: apimService
  name: 'backend-api-key'
  properties: {
    displayName: 'Backend API Key'
    secret: true
    keyVault: {
      secretIdentifier: 'https://${keyVaultName}.vault.azure.net/secrets/backend-api-key'
      identityClientId: apimManagedIdentity.clientId
    }
  }
}
```

## Cost Optimization by Environment

### Development

**Strategies:**
- Use Developer or Consumption SKU
- Single unit
- Auto-shutdown if supported
- Delete when not in use (ephemeral)
- Shared resources

**Estimated Cost:** $50-75/month

### Testing

**Strategies:**
- Standard SKU
- Minimal capacity
- Shared with multiple teams
- Delete test data regularly

**Estimated Cost:** $500-700/month

### Production

**Strategies:**
- Right-size capacity
- Enable caching to reduce backend load
- Monitor and scale based on usage
- Use reserved capacity if available

**Estimated Cost:** $1,500-3,000+/month

## Deployment Approval Gates

### Development

**Approvals:** None

**Automatic deployment** on merge to develop branch

### Testing

**Approvals:** Optional

**Manual trigger** or automatic with optional approval

### Production

**Approvals:** Required

**Require:**
- 2+ reviewers
- Successful test environment validation
- Change ticket/approval
- Off-hours deployment window

## Environment-Specific Testing

### Development

**Tests:**
- Unit tests
- Integration tests
- Policy tests
- Spec validation

### Testing

**Tests:**
- Full integration tests
- Load testing
- Security scanning
- UAT (User Acceptance Testing)

### Production

**Tests:**
- Smoke tests (immediate post-deploy)
- Synthetic monitoring
- Canary deployments
- Rollback testing

## Monitoring Per Environment

### Development

**Metrics:**
- Basic platform metrics
- Error logs
- Developer activity

**Retention:** 7-14 days

### Testing

**Metrics:**
- All platform and application metrics
- Performance baselines
- Test execution results

**Retention:** 30 days

### Production

**Metrics:**
- Comprehensive telemetry
- Business metrics
- SLA compliance
- Cost tracking

**Retention:** 90 days (operational), archive for compliance

## Environment Cleanup

### Ephemeral Environments

For feature branches or testing:

**Create:**
```bash
./create-ephemeral-env.sh feature-123
```

**Destroy after testing:**
```bash
./destroy-ephemeral-env.sh feature-123
```

### Scheduled Cleanup

**Development:** 
- Purge logs weekly
- Clean test data daily
- Remove old revisions monthly

**Testing:**
- Purge logs monthly
- Clean test data weekly
- Archive old configs quarterly

## Best Practices

1. **Use infrastructure as code** - All environments defined in code
2. **Consistent naming** - Clear environment indicators (e.g., `-dev`, `-prod`)
3. **Separate subscriptions** - Consider separate Azure subscriptions per environment
4. **Promote, don't replicate** - Deploy same code across environments
5. **Environment parity** - Make prod-like for accurate testing
6. **Automate provisioning** - Script environment creation
7. **Document differences** - Clearly document why environments differ
8. **Use feature flags** - Control features without redeployment
9. **Monitor costs** - Track spending per environment
10. **Regular cleanup** - Remove obsolete resources

## Common Pitfalls

### Pitfall 1: Configuration Drift

**Problem:** Manual changes in portal not reflected in code

**Solution:** 
- Enforce code-only changes
- Regular configuration audits
- Automated drift detection

### Pitfall 2: Environment-Specific Code

**Problem:** Different code in different environments

**Solution:**
- Use feature flags
- Parameterize configuration
- Same codebase across environments

### Pitfall 3: Production-Only Testing

**Problem:** Testing changes only in production

**Solution:**
- Production-like staging environment
- Mandatory testing before production
- Automated validation

### Pitfall 4: Shared Secrets

**Problem:** Same secrets across all environments

**Solution:**
- Unique secrets per environment
- Separate Key Vaults
- Regular rotation

## Next Steps

1. **Define environments** - Determine what environments you need
2. **Create parameter files** - One per environment
3. **Set up GitHub environments** - With appropriate protections
4. **Implement promotion workflow** - Dev → Test → Prod
5. **Configure monitoring** - Per environment
6. **Document differences** - Environment-specific configurations
7. **Test promotion** - Validate workflow end-to-end

## Additional Resources

- [Azure Resource Manager Parameters](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameters)
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/)
