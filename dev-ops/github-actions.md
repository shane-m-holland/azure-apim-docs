# GitHub Actions for APIM

## Overview

GitHub Actions provides CI/CD capabilities for automating Azure API Management deployments. This guide covers workflow patterns, authentication, and best practices for implementing GitOps with APIM.

## Authentication with Azure

### OIDC Federation (Recommended)

**Benefits:**
- No secrets to manage
- Automatic credential rotation
- More secure than long-lived credentials
- Granular permissions per environment

**Setup:**

#### 1. Create Service Principal

```bash
az ad sp create-for-rbac \
  --name "apim-github-actions" \
  --role "API Management Service Contributor" \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group}
```

#### 2. Configure Federated Credentials

```bash
az ad app federated-credential create \
  --id {app-id} \
  --parameters '{
    "name": "github-federation",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:{org}/{repo}:environment:{environment}",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

#### 3. GitHub Secrets

Add these secrets to GitHub repository or environment:

```
AZURE_CLIENT_ID: {app-id}
AZURE_TENANT_ID: {tenant-id}
AZURE_SUBSCRIPTION_ID: {subscription-id}
```

#### 4. Workflow Authentication

```yaml
permissions:
  id-token: write  # Required for OIDC
  contents: read   # Required for checkout

steps:
  - name: Azure Login
    uses: azure/login@v1
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Workflow Patterns

### Infrastructure Deployment Workflow

**File:** `.github/workflows/deploy-infrastructure.yml`

```yaml
name: Deploy APIM Infrastructure

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - test
          - prod

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Validate Bicep
        run: |
          az bicep build --file ./bicep/main.bicep
      
      - name: What-If Analysis
        run: |
          az deployment group what-if \
            --resource-group ${{ vars.RESOURCE_GROUP }} \
            --template-file ./bicep/main.bicep \
            --parameters ./environments/${{ inputs.environment }}.bicepparam
      
      - name: Deploy Infrastructure
        run: |
          az deployment group create \
            --resource-group ${{ vars.RESOURCE_GROUP }} \
            --template-file ./bicep/main.bicep \
            --parameters ./environments/${{ inputs.environment }}.bicepparam \
            --name "apim-${{ inputs.environment }}-$(date +%Y%m%d-%H%M%S)"
```

### API Deployment Workflow

**File:** `.github/workflows/deploy-apis.yml`

```yaml
name: Deploy APIs

on:
  push:
    branches:
      - develop  # Auto-deploy to dev
      - main     # Auto-deploy to prod
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - test
          - prod

permissions:
  id-token: write
  contents: read

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - name: Set environment
        id: set-env
        run: |
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            echo "environment=${{ inputs.environment }}" >> $GITHUB_OUTPUT
          elif [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi

  deploy-apis:
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Checkout tooling repo
        uses: actions/checkout@v4
        with:
          repository: shane-m-holland/azure-apim-bicep
          path: tooling
          ref: main
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Validate API Configuration
        run: |
          ./tooling/scripts/validate-config.sh \
            ${{ needs.determine-environment.outputs.environment }} \
            ./api-config.yaml
      
      - name: Deploy APIs
        run: |
          ./tooling/scripts/deploy-apis.sh \
            ${{ needs.determine-environment.outputs.environment }} \
            ./api-config.yaml \
            --parallel
```

### Validation Workflow

**File:** `.github/workflows/validate.yml`

```yaml
name: Validate

on:
  pull_request:
    branches:
      - develop
      - main

jobs:
  validate-bicep:
    runs-on: ubuntu-latest
    if: contains(github.event.pull_request.changed_files, 'bicep/')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Bicep
        run: az bicep install
      
      - name: Lint Bicep
        run: |
          az bicep build --file ./bicep/main.bicep
      
      - name: Validate Templates
        run: |
          az deployment group validate \
            --resource-group ${{ vars.RESOURCE_GROUP_DEV }} \
            --template-file ./bicep/main.bicep \
            --parameters ./environments/dev.bicepparam

  validate-apis:
    runs-on: ubuntu-latest
    if: contains(github.event.pull_request.changed_files, 'api-config.yaml')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Checkout tooling
        uses: actions/checkout@v4
        with:
          repository: shane-m-holland/azure-apim-bicep
          path: tooling
      
      - name: Validate API Configuration
        run: |
          ./tooling/scripts/validate-config.sh dev ./api-config.yaml --verbose
      
      - name: Validate OpenAPI Specs
        run: |
          npm install -g @stoplight/spectral-cli
          spectral lint ./specs/*.yaml
```

## Reusable Workflows

Create reusable workflows in the tooling repository.

### Reusable Deployment Workflow

**File:** `.github/workflows/deploy-apim.yml` (in tooling repo)

```yaml
name: Deploy APIM (Reusable)

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      resource-group:
        required: true
        type: string
      template-file:
        required: true
        type: string
      parameters-file:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true
      AZURE_TENANT_ID:
        required: true
      AZURE_SUBSCRIPTION_ID:
        required: true

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Deploy
        run: |
          az deployment group create \
            --resource-group ${{ inputs.resource-group }} \
            --template-file ${{ inputs.template-file }} \
            --parameters ${{ inputs.parameters-file }}
```

### Using Reusable Workflow

**In service repository:**

```yaml
name: Deploy Infrastructure

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [dev, test, prod]

jobs:
  deploy:
    uses: shane-m-holland/azure-apim-bicep/.github/workflows/deploy-apim.yml@main
    with:
      environment: ${{ inputs.environment }}
      resource-group: ${{ vars.RESOURCE_GROUP }}
      template-file: ./bicep/main.bicep
      parameters-file: ./environments/${{ inputs.environment }}.bicepparam
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Environment Configuration

### GitHub Environments

Create environments in GitHub:
- `dev` - Auto-deploy from develop branch
- `test` - Manual approval required
- `prod` - Manual approval + reviewers required

**Per-environment configuration:**
- Secrets (Azure credentials)
- Variables (resource names, SKUs)
- Protection rules
- Deployment branches

### Environment Secrets

**Required for each environment:**

```
# Authentication
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID

# APIM Configuration (if different approach)
APIM_RESOURCE_GROUP
APIM_SERVICE_NAME
```

### Environment Variables

**Non-sensitive configuration:**

```
# Infrastructure
RESOURCE_GROUP=rg-apim-dev
LOCATION=eastus2
APIM_SKU=Developer

# Networking
VNET_NAME=vnet-apim-dev
SUBNET_NAME=snet-apim
```

## Advanced Patterns

### Matrix Deployments

Deploy to multiple regions:

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        region: [eastus2, westus2, northeurope]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ matrix.region }}
        run: |
          az deployment group create \
            --resource-group rg-apim-${{ matrix.region }} \
            --template-file ./bicep/main.bicep \
            --parameters location=${{ matrix.region }}
```

### Conditional Steps

Run steps based on conditions:

```yaml
- name: Deploy to Production
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: ./deploy-prod.sh

- name: Deploy to Dev
  if: github.ref == 'refs/heads/develop'
  run: ./deploy-dev.sh
```

### Artifact Handling

Save and restore build artifacts:

```yaml
- name: Build artifacts
  run: |
    ./build.sh
    tar -czf artifacts.tar.gz dist/

- name: Upload artifacts
  uses: actions/upload-artifact@v3
  with:
    name: build-artifacts
    path: artifacts.tar.gz

# In another job
- name: Download artifacts
  uses: actions/download-artifact@v3
  with:
    name: build-artifacts
```

## Best Practices

### Workflow Organization

1. **Separate workflows by purpose**
   - Infrastructure deployment
   - API deployment
   - Validation/testing
   - Scheduled tasks

2. **Use reusable workflows** - DRY principle
3. **Pin action versions** - `uses: actions/checkout@v4` (not `@main`)
4. **Use environments** - Enforce approvals and protections

### Security

1. **Use OIDC** - No long-lived secrets
2. **Minimize permissions** - Least privilege
3. **Protect branches** - Require PR reviews
4. **Enable audit logs** - Track workflow runs
5. **Scan dependencies** - Dependabot alerts

### Performance

1. **Cache dependencies** - Faster workflow runs
2. **Parallel jobs** - When possible
3. **Fail fast** - Don't wait for all jobs
4. **Selective triggers** - Only run when needed

### Monitoring

1. **Set up notifications** - Slack, email, Teams
2. **Track metrics** - Success rate, duration
3. **Review failed runs** - Weekly reviews
4. **Document runbooks** - Incident response

## Troubleshooting

### Authentication Failures

**Symptoms:** Login fails with 401/403

**Solutions:**
1. Verify federated credential configuration
2. Check subject matches repo/environment
3. Ensure OIDC permissions in workflow
4. Validate service principal permissions

### Deployment Timeouts

**Symptoms:** Workflow times out during deployment

**Solutions:**
1. Increase timeout: `timeout-minutes: 60`
2. Check APIM deployment status in Azure
3. Use async deployments for long operations
4. Break into smaller deployment steps

### Failed Validations

**Symptoms:** Bicep/API validation fails

**Solutions:**
1. Run validation locally first
2. Check error messages carefully
3. Validate parameter files
4. Ensure all dependencies exist

## Next Steps

1. **Set up OIDC authentication** - More secure than secrets
2. **Create base workflows** - Infrastructure and API deployment
3. **Configure environments** - Dev, test, prod with protections
4. **Add validation** - PR checks for quality
5. **Enable notifications** - Stay informed of deployment status
6. **Document procedures** - Runbooks for common tasks

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Login Action](https://github.com/Azure/login)
- [Bicep Deploy Action](https://github.com/Azure/arm-deploy)
- [OIDC with Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
