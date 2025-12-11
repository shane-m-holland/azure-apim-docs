# Automated API Deployments

## Overview

Automated API deployment enables development teams to deploy API changes quickly and safely through CI/CD pipelines. This guide covers patterns, workflows, and best practices for automating APIM API deployments.

## Deployment Approaches

### 1. Bicep-Based Deployment

Deploy APIs using Bicep templates:

**Pros:**
- Declarative and idempotent
- Version controlled
- Validated before deployment
- Rollback via git revert

**Cons:**
- Requires Bicep knowledge
- More setup initially
- Template maintenance

### 2. Script-Based Deployment

Use shell scripts with Azure CLI:

**Pros:**
- Flexible and scriptable
- Easy to customize
- Quick to implement
- Good for one-off operations

**Cons:**
- Imperative (not idempotent)
- Error handling complexity
- Harder to audit changes

### 3. Hybrid Approach (Recommended)

Combination of Bicep and scripts:
- **Bicep**: Infrastructure and API definitions
- **Scripts**: Orchestration and validation
- **CI/CD**: GitHub Actions for automation

## API Configuration Format

### YAML Configuration (Recommended)

**File:** `api-config.yaml`

```yaml
apis:
  - name: weather-api
    displayName: Weather API
    description: Provides weather forecast data
    path: weather
    serviceUrl: ${WEATHER_SERVICE_URL}
    protocols:
      - https
    subscriptionRequired: true
    specFormat: openapi
    specFile: ./specs/weather-api.yaml
    products:
      - starter
      - unlimited
    policies:
      inbound: ./policies/weather-inbound.xml
      backend: ./policies/common-backend.xml
    revision:
      number: 2
      description: Added caching policy
    
  - name: users-api
    displayName: Users API
    path: users
    serviceUrl: ${USERS_SERVICE_URL}
    protocols:
      - https
    specFormat: openapi
    specFile: ./specs/users-api.yaml
    products:
      - internal
```

### Environment Variable Substitution

Variables in config are replaced at deployment:

```bash
export WEATHER_SERVICE_URL=https://weather.backend.com
export USERS_SERVICE_URL=https://users.backend.com

./deploy-apis.sh dev ./api-config.yaml
```

## Deployment Scripts

### Validation Script

**File:** `validate-apis.sh`

```bash
#!/bin/bash
set -e

CONFIG_FILE=$1
ENVIRONMENT=$2

echo "Validating API configuration..."

# Check YAML syntax
yq eval '.' "$CONFIG_FILE" > /dev/null || {
    echo "Error: Invalid YAML syntax"
    exit 1
}

# Validate OpenAPI specs exist
for spec in $(yq eval '.apis[].specFile' "$CONFIG_FILE"); do
    if [ ! -f "$spec" ]; then
        echo "Error: Spec file not found: $spec"
        exit 1
    fi
    
    # Validate OpenAPI syntax
    npx @stoplight/spectral-cli lint "$spec" || {
        echo "Error: Invalid OpenAPI spec: $spec"
        exit 1
    }
done

# Validate policy files exist
for policy in $(yq eval '.apis[].policies.inbound' "$CONFIG_FILE" | grep -v null); do
    if [ ! -f "$policy" ]; then
        echo "Error: Policy file not found: $policy"
        exit 1
    fi
done

echo "Validation successful!"
```

### Deployment Script

**File:** `deploy-apis.sh`

```bash
#!/bin/bash
set -e

ENVIRONMENT=$1
CONFIG_FILE=$2
PARALLEL=${3:-false}

# Load environment configuration
source "./environments/${ENVIRONMENT}.env"

echo "Deploying APIs to $ENVIRONMENT..."

# Get list of APIs from config
apis=$(yq eval '.apis[].name' "$CONFIG_FILE")

deploy_api() {
    local api_name=$1
    
    echo "Deploying API: $api_name"
    
    # Extract API configuration
    display_name=$(yq eval ".apis[] | select(.name == \"$api_name\") | .displayName" "$CONFIG_FILE")
    path=$(yq eval ".apis[] | select(.name == \"$api_name\") | .path" "$CONFIG_FILE")
    service_url=$(yq eval ".apis[] | select(.name == \"$api_name\") | .serviceUrl" "$CONFIG_FILE")
    spec_file=$(yq eval ".apis[] | select(.name == \"$api_name\") | .specFile" "$CONFIG_FILE")
    
    # Substitute environment variables
    service_url=$(eval echo "$service_url")
    
    # Deploy API using Bicep
    az deployment group create \
        --resource-group "$RESOURCE_GROUP" \
        --template-file ./bicep/api.bicep \
        --parameters \
            apimServiceName="$APIM_SERVICE_NAME" \
            apiName="$api_name" \
            displayName="$display_name" \
            path="$path" \
            serviceUrl="$service_url" \
            specFile="$spec_file" \
        --no-wait
    
    echo "API $api_name deployment initiated"
}

if [ "$PARALLEL" == "--parallel" ]; then
    # Deploy all APIs in parallel
    for api in $apis; do
        deploy_api "$api" &
    done
    wait
else
    # Deploy APIs sequentially
    for api in $apis; do
        deploy_api "$api"
    done
fi

echo "All API deployments completed!"
```

### Sync Script (Smart Deployment)

Only deploy changed APIs:

**File:** `sync-apis.sh`

```bash
#!/bin/bash
set -e

ENVIRONMENT=$1
CONFIG_FILE=$2

# Load environment
source "./environments/${ENVIRONMENT}.env"

# Get deployed APIs
deployed_apis=$(az apim api list \
    --resource-group "$RESOURCE_GROUP" \
    --service-name "$APIM_SERVICE_NAME" \
    --query "[].name" -o tsv)

# Get configured APIs
configured_apis=$(yq eval '.apis[].name' "$CONFIG_FILE")

# Compare and deploy only changed APIs
for api in $configured_apis; do
    if echo "$deployed_apis" | grep -q "^$api$"; then
        # API exists, check if changed
        current_revision=$(az apim api show \
            --resource-group "$RESOURCE_GROUP" \
            --service-name "$APIM_SERVICE_NAME" \
            --api-id "$api" \
            --query "apiRevision" -o tsv)
        
        config_revision=$(yq eval ".apis[] | select(.name == \"$api\") | .revision.number" "$CONFIG_FILE")
        
        if [ "$current_revision" != "$config_revision" ]; then
            echo "API $api changed, deploying..."
            ./deploy-api.sh "$ENVIRONMENT" "$api"
        else
            echo "API $api unchanged, skipping"
        fi
    else
        # New API, deploy
        echo "New API $api, deploying..."
        ./deploy-api.sh "$ENVIRONMENT" "$api"
    fi
done

echo "Sync completed!"
```

## GitHub Actions Integration

### Service Repository Workflow

**File:** `.github/workflows/deploy-api.yml`

```yaml
name: Deploy API

on:
  push:
    branches:
      - develop
      - main
    paths:
      - 'api-config.yaml'
      - 'specs/**'
      - 'policies/**'

permissions:
  id-token: write
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: |
          npm install -g @stoplight/spectral-cli
          pip install yq
      
      - name: Validate configuration
        run: ./scripts/validate-apis.sh api-config.yaml

  deploy-dev:
    needs: validate
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: dev
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Checkout tooling
        uses: actions/checkout@v4
        with:
          repository: shane-m-holland/azure-apim-bicep
          path: tooling
      
      - uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Deploy APIs
        env:
          WEATHER_SERVICE_URL: ${{ vars.WEATHER_SERVICE_URL }}
          USERS_SERVICE_URL: ${{ vars.USERS_SERVICE_URL }}
        run: |
          ./tooling/scripts/deploy-apis.sh dev api-config.yaml --parallel

  deploy-prod:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: prod
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Checkout tooling
        uses: actions/checkout@v4
        with:
          repository: shane-m-holland/azure-apim-bicep
          path: tooling
      
      - uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Deploy APIs (with approval)
        env:
          WEATHER_SERVICE_URL: ${{ vars.WEATHER_SERVICE_URL }}
          USERS_SERVICE_URL: ${{ vars.USERS_SERVICE_URL }}
        run: |
          ./tooling/scripts/deploy-apis.sh prod api-config.yaml
```

## Deployment Strategies

### Blue-Green Deployment with Revisions

Deploy new revision, test, then make current:

```bash
# Create new revision
az apim api revision create \
    --resource-group $RG \
    --service-name $APIM \
    --api-id my-api \
    --api-revision 2 \
    --api-revision-description "New features"

# Deploy changes to revision 2
# Test revision 2

# Make revision 2 current
az apim api release create \
    --resource-group $RG \
    --service-name $APIM \
    --api-id my-api \
    --api-revision 2 \
    --notes "Promoting revision 2 to current"
```

### Canary Deployment

Route percentage of traffic to new revision:

```xml
<!-- In APIM policy -->
<choose>
  <when condition="@(context.Variables.GetValueOrDefault<int>("random") < 10)">
    <!-- 10% of traffic to new revision -->
    <set-backend-service base-url="https://new-backend.com" />
  </when>
  <otherwise>
    <!-- 90% to current revision -->
    <set-backend-service base-url="https://current-backend.com" />
  </otherwise>
</choose>
```

### Rolling Deployment

Deploy to regions sequentially:

```yaml
strategy:
  matrix:
    region: [eastus2, westus2, northeurope]
  max-parallel: 1  # One region at a time
```

## Rollback Procedures

### Revision Rollback

Revert to previous revision:

```bash
# List revisions
az apim api revision list \
    --resource-group $RG \
    --service-name $APIM \
    --api-id my-api

# Make previous revision current
az apim api release create \
    --resource-group $RG \
    --service-name $APIM \
    --api-id my-api \
    --api-revision 1 \
    --notes "Rollback to revision 1"
```

### Git Revert

Revert commit and redeploy:

```bash
# Revert last commit
git revert HEAD

# Push revert
git push origin main

# CI/CD automatically deploys reverted version
```

## Testing in Pipeline

### Smoke Tests

Test deployed APIs:

```yaml
- name: Smoke test APIs
  run: |
    # Test weather API
    response=$(curl -s -o /dev/null -w "%{http_code}" \
      -H "Ocp-Apim-Subscription-Key: ${{ secrets.APIM_SUBSCRIPTION_KEY }}" \
      "https://${{ vars.APIM_GATEWAY }}/weather/forecast")
    
    if [ "$response" != "200" ]; then
      echo "Weather API test failed"
      exit 1
    fi
    
    echo "Smoke tests passed!"
```

### Contract Tests

Validate API contracts:

```yaml
- name: Contract tests
  run: |
    npm install -g @pact-foundation/pact
    npm run test:contract
```

## Monitoring Deployments

### Deployment Metrics

Track in Azure Monitor:
- Deployment duration
- Success/failure rate
- APIs deployed per day
- Rollback frequency

### Alerts

Configure alerts for:
- Deployment failures
- Long deployment times (>30 minutes)
- High rollback rate
- API availability drops after deployment

## Best Practices

1. **Validate before deploy** - Catch errors early
2. **Use revisions** - Safe, testable updates
3. **Parallel deployments** - Faster for multiple APIs
4. **Smart sync** - Only deploy changed APIs
5. **Environment variables** - Never hardcode URLs
6. **Test after deploy** - Smoke tests in pipeline
7. **Monitor deployments** - Track metrics and failures
8. **Document procedures** - Rollback runbooks
9. **Automate everything** - Reduce human error
10. **Use Git tags** - Mark production releases

## Troubleshooting

### Deployment Hangs

**Problem:** Deployment never completes

**Solutions:**
- Check APIM instance status
- Review deployment logs in Azure
- Use `--no-wait` for async deployments
- Increase timeout in workflow

### Invalid OpenAPI Spec

**Problem:** API deployment fails with spec error

**Solutions:**
- Validate spec with Spectral before deployment
- Check for unsupported OpenAPI features in APIM
- Review error message for specific issues

### Policy Errors

**Problem:** Policy deployment fails

**Solutions:**
- Validate XML syntax
- Test policies in APIM test console
- Check for invalid policy elements
- Review policy execution order

## Next Steps

1. **Set up configuration file** - Create api-config.yaml
2. **Implement validation** - Add to CI/CD pipeline
3. **Enable automated deployment** - On merge to develop/main
4. **Add smoke tests** - Verify deployments succeed
5. **Configure monitoring** - Track deployment metrics
6. **Document rollback** - Create runbook for incidents

## Additional Resources

- [APIM DevOps Resource Kit](https://github.com/Azure/azure-api-management-devops-resource-kit)
- [Azure APIM Bicep](https://github.com/shane-m-holland/azure-apim-bicep)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
