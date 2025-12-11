# Security Best Practices

## Overview

Security is critical in DevOps pipelines. This guide covers best practices for securing CI/CD pipelines, protecting secrets, and maintaining secure APIM deployments.

## Authentication and Authorization

### OIDC Federation (Recommended)

**Benefits:**
- No long-lived secrets in GitHub
- Automatic credential rotation
- Reduced attack surface
- Fine-grained permissions

**Setup:**

```bash
# Create service principal
az ad sp create-for-rbac --name "github-apim-deployer" \
  --role "API Management Service Contributor" \
  --scopes /subscriptions/{sub-id}/resourceGroups/{rg}

# Configure OIDC federation
az ad app federated-credential create \
  --id {app-id} \
  --parameters '{
    "name": "github-prod",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:org/repo:environment:prod",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

### Least Privilege Principle

**Grant minimum required permissions:**

```bash
# API Management Service Contributor for deployments
az role assignment create \
  --assignee {sp-id} \
  --role "API Management Service Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}

# Reader for validation/what-if
az role assignment create \
  --assignee {sp-id} \
  --role "Reader" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}
```

## Secrets Management

### Never Commit Secrets

**Use `.gitignore`:**

```gitignore
# Environment files with secrets
.env
.env.*
*.env.local

# Azure credentials
.azure/
credentials.json

# Bicep parameters with secrets
*.parameters.prod.json

# API keys and tokens
**/secrets/
*.key
*.pem
*.pfx
```

### Azure Key Vault

Store secrets in Key Vault, reference in APIM:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: apimManagedIdentity.principalId
        permissions: {
          secrets: ['get']
        }
      }
    ]
  }
}

// Store backend API key in Key Vault
resource backendApiKeySecret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'backend-api-key'
  properties: {
    value: backendApiKey  // Passed as secure parameter
  }
}

// Reference in APIM Named Value
resource namedValue 'Microsoft.ApiManagement/service/namedValues@2023-03-01-preview' = {
  parent: apimService
  name: 'backend-api-key'
  properties: {
    displayName: 'Backend API Key'
    secret: true
    keyVault: {
      secretIdentifier: backendApiKeySecret.properties.secretUri
      identityClientId: apimManagedIdentity.clientId
    }
  }
}
```

### GitHub Secrets

Store deployment credentials securely:

**Repository Secrets:** Accessible to all workflows
**Environment Secrets:** Scoped to specific environments

```yaml
# Use secrets in workflow
- name: Azure Login
  uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Secure Parameters

Use `@secure()` decorator in Bicep:

```bicep
@secure()
param adminPassword string

@secure()
param apiKey string

// These parameters won't be logged or visible in deployment history
```

## Code Security

### Branch Protection

**Enable on main/develop branches:**

```yaml
Required:
  - Require pull request reviews (2+ approvers for prod)
  - Require status checks to pass
  - Require conversation resolution
  - Require signed commits
  - Restrict force pushes
  - Restrict deletions
```

### Code Scanning

**GitHub Advanced Security:**

```yaml
name: CodeQL Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, typescript
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

### Dependency Scanning

**Dependabot configuration:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Secret Scanning

Enable GitHub secret scanning:

1. Repository Settings → Security → Secret scanning
2. Push protection (prevent committing secrets)
3. Custom patterns for organization-specific secrets

## Pipeline Security

### Workflow Permissions

**Minimize permissions:**

```yaml
permissions:
  id-token: write  # Only what's needed for OIDC
  contents: read   # Only read access to repository
  # Don't grant write permissions unless absolutely necessary
```

### Pin Action Versions

**Use specific versions, not latest:**

```yaml
# Bad - uses latest, could break or introduce vulnerabilities
- uses: actions/checkout@main

# Good - pinned to specific version
- uses: actions/checkout@v4

# Better - pinned to commit SHA
- uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab
```

### Environment Isolation

**Separate environments with distinct permissions:**

```yaml
environments:
  dev:
    protection_rules: []
    secrets:
      AZURE_CLIENT_ID: {dev-sp-id}
  
  prod:
    protection_rules:
      - required_reviewers: 2
      - wait_timer: 30  # 30 minute wait
    secrets:
      AZURE_CLIENT_ID: {prod-sp-id}
```

### Input Validation

**Validate all inputs:**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, test, prod]  # Restrict to valid values
      
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Validate input
        run: |
          if [[ ! "${{ inputs.environment }}" =~ ^(dev|test|prod)$ ]]; then
            echo "Invalid environment"
            exit 1
          fi
```

## Infrastructure Security

### Network Security

**Restrict APIM access:**

```bicep
// Network Security Group rules
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-04-01' = {
  name: 'nsg-apim-prod'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-HTTPS-Inbound'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '443'
        }
      }
      {
        name: 'Deny-All-Inbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
    ]
  }
}
```

### Managed Identities

**Use managed identities instead of credentials:**

```bicep
// Enable managed identity on APIM
resource apimService 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: apimName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    // ...
  }
}

// Grant APIM access to Key Vault
resource keyVaultAccessPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2023-02-01' = {
  parent: keyVault
  name: 'add'
  properties: {
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: apimService.identity.principalId
        permissions: {
          secrets: ['get', 'list']
        }
      }
    ]
  }
}
```

### TLS/SSL Configuration

**Enforce HTTPS only:**

```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.OriginalUrl.Scheme != "https")">
        <return-response>
          <set-status code="403" reason="HTTPS Required" />
        </return-response>
      </when>
    </choose>
  </inbound>
</policies>
```

## Audit and Compliance

### Audit Logging

**Enable Azure Activity Logs:**

```bicep
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  scope: apimService
  name: 'audit-logs'
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      {
        category: 'GatewayLogs'
        enabled: true
      }
      {
        categoryGroup: 'audit'
        enabled: true
      }
    ]
  }
}
```

### Deployment History

**Track all deployments:**

```yaml
- name: Record deployment
  run: |
    cat <<EOF > deployment-record.json
    {
      "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "environment": "${{ inputs.environment }}",
      "commit": "${{ github.sha }}",
      "actor": "${{ github.actor }}",
      "run_id": "${{ github.run_id }}"
    }
    EOF
    
    # Upload to audit storage
    az storage blob upload \
      --account-name $AUDIT_STORAGE \
      --container deployments \
      --file deployment-record.json \
      --name "$(date +%Y/%m/%d)/${{ github.run_id }}.json"
```

### Compliance Scanning

**Scan for compliance violations:**

```yaml
- name: Policy Compliance Scan
  uses: azure/policy-compliance-scan@v0
  with:
    scopes: |
      /subscriptions/{subscription-id}/resourceGroups/{rg}
    wait: true
```

## Incident Response

### Security Incident Playbook

**If secrets are compromised:**

1. **Immediate Actions:**
   - Revoke compromised credentials
   - Remove from GitHub secrets
   - Delete from Key Vault
   - Rotate all affected keys

2. **Investigation:**
   - Review audit logs
   - Identify scope of exposure
   - Check for unauthorized access
   - Document timeline

3. **Remediation:**
   - Deploy new credentials
   - Update CI/CD pipelines
   - Validate no residual access
   - Monitor for suspicious activity

4. **Prevention:**
   - Review security practices
   - Update documentation
   - Train team
   - Implement additional controls

### Rollback Security

**Secure rollback process:**

```yaml
- name: Emergency Rollback
  if: failure()
  run: |
    # Revert to last known good revision
    az apim api revision list \
      --resource-group $RG \
      --service-name $APIM \
      --api-id $API_ID \
      --query "[?isCurrent].{revision:apiRevision}" \
      -o tsv
    
    # Make previous revision current
    PREVIOUS_REVISION=$((CURRENT_REVISION - 1))
    az apim api release create \
      --resource-group $RG \
      --service-name $APIM \
      --api-id $API_ID \
      --api-revision $PREVIOUS_REVISION \
      --notes "Emergency rollback"
```

## Security Checklist

### Pre-Deployment

- [ ] No secrets in code or configs
- [ ] All sensitive parameters marked `@secure()`
- [ ] OIDC federation configured
- [ ] Least privilege permissions granted
- [ ] Branch protection enabled
- [ ] Code scanning enabled
- [ ] Dependency scanning enabled
- [ ] Secret scanning enabled

### During Deployment

- [ ] Workflow uses pinned action versions
- [ ] Minimal workflow permissions
- [ ] Input validation implemented
- [ ] Environment-specific credentials
- [ ] Audit logging enabled
- [ ] Deployment recorded

### Post-Deployment

- [ ] Verify no secrets exposed
- [ ] Review audit logs
- [ ] Monitor for anomalies
- [ ] Test security controls
- [ ] Update documentation
- [ ] Rotate credentials on schedule

## Best Practices Summary

1. **Use OIDC** - No long-lived secrets
2. **Secrets in Key Vault** - Never in code
3. **Least privilege** - Minimum required permissions
4. **Branch protection** - Require reviews and checks
5. **Pin versions** - Specific action versions
6. **Managed identities** - Avoid credentials where possible
7. **Audit everything** - Comprehensive logging
8. **Regular rotation** - Scheduled credential updates
9. **Incident response** - Documented procedures
10. **Security training** - Educate team regularly

## Additional Resources

- [GitHub Actions Security](https://docs.github.com/en/actions/security-guides)
- [Azure Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/best-practices-and-patterns)
- [OWASP DevSecOps](https://owasp.org/www-project-devsecops-guideline/)
- [CIS Azure Foundations Benchmark](https://www.cisecurity.org/benchmark/azure)
