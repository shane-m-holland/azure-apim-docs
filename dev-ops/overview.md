# Azure API Management (APIM)

## Automation Overview

### Why automate APIM?

- **Repeatability & consistency**: Declarative definitions eliminate click‑ops drift and enable identical environments.
- **Versioning & auditability**: IaC lives in git; every change is reviewed, tested, and traceable.
- **Speed with safety**: Pipelines add validation, linting, and gated approvals.
- **Environment promotion**: Reuse the same templates to promote from dev -> test -> prod.
- **Disaster recovery**: Recreate APIM state from code.

### Core patterns

- **IaC first**: Use **Bicep** (or ARM) for APIM resources (service, products, APIs, revisions, policies).
- **GitOps**: Git is the source of truth; merges to `main` trigger deploys.
- **Separation of config vs. code**: Reuse templates; pass environment‑specific parameters (e.g., SKU, capacity).
- **Idempotent deployments**: Declarative templates safely converge to desired state.

### Best practices

- Prefer **modules** per resource type; keep `main.bicep` orchestrating.
- Externalize **secrets** to Key Vault; bind via APIM **named values** with `secret: true`.
- Use **what‑if** in CI to preview changes; require approvals for prod.
- Use **revisions** for safe rollouts; swap `isCurrent` after validation.
- Validate OpenAPI and policies in CI before deployment.

## GitHub Actions

### Prereqs

1. **Azure AD Federated Credential** for your GitHub repo/environment (OIDC).
2. **Resource group(s)** and **APIM instance(s)** created or bootstrapped.
3. Store non‑secret environment config in repo; secrets in **Environments** or **Repository Secrets**.
  
    - For **secrets** (e.g., backend credentials), reference **Key Vault** in APIM named values; never hard‑code.

---

### GitHub Actions Workflow Configuration

⚠️ **Critical**: The GitHub Actions workflow requires specific OIDC permissions to authenticate with Azure. Each job in the workflow **must** include the following permissions block:

```yaml
permissions:
  id-token: write    # Required for OIDC token requests
  contents: read     # Required for repository checkout
```

**Why these permissions are needed:**

- `id-token: write`: Allows the workflow to request OIDC tokens from GitHub's token service
- `contents: read`: Allows the workflow to checkout your repository code

Without these permissions, Azure login will fail with authentication errors.

### GitHub Configuration

#### Required Secrets

Navigate to **Settings** → **Secrets and variables** → **Actions** in your GitHub repository:

**Authentication Secrets:**

- `AZURE_CLIENT_ID`: App ID from service principal creation
- `AZURE_TENANT_ID`: Your Azure tenant ID (`az account show --query tenantId -o tsv`)
- `AZURE_SUBSCRIPTION_ID`: Your Azure subscription ID
- `AZURE_RESOURCE_GROUP`: Resource group for Function App deployment

**APIM Configuration Secrets:**

- `APIM_VALIDATION_KEY`: Base64-encoded APIM validation key
- `APIM_PORTAL_URL`: APIM Developer Portal URL (e.g., `https://contoso.developer.azure-api.net`)

**OIDC Provider Secrets:**

- `OIDC_ISSUER`: Identity provider URL (e.g., `https://your-domain.okta.com`)
- `OIDC_CLIENT_ID`: OAuth client ID
- `OIDC_CLIENT_SECRET`: OAuth client secret

**Optional Secrets (for cross-subscription scenarios):**

- `APIM_ACCESS_TOKEN`: Azure Bearer token for cross-subscription APIM access

#### Optional Variables

Navigate to **Settings** → **Secrets and variables** → **Actions** → **Variables**:

**Deployment Configuration:**

- `APP_NAME`: Application name (default: `apim-delegation`)
- `AZURE_LOCATION`: Azure region (default: `eastus2`)
- `AZURE_SKU`: Function App SKU (default: `FC1`)
- `AZURE_OS_TYPE`: OS type (default: `linux`)
- `RUNTIME`: Function runtime (default: `node`)

**APIM Configuration (same-subscription scenarios):**

- `APIM_RESOURCE_GROUP`: APIM resource group name
- `APIM_SERVICE_NAME`: APIM service name

**Custom OIDC Endpoints (if auto-discovery fails):**

- `OIDC_AUTHORIZATION_ENDPOINT`: Custom auth endpoint path
- `OIDC_TOKEN_ENDPOINT`: Custom token endpoint path  
- `OIDC_USERINFO_ENDPOINT`: Custom userinfo endpoint path
- `OIDC_END_SESSION_ENDPOINT`: Custom logout endpoint path

## Security Best Practices

1. **Never commit `.env.*` files** - They're git-ignored by default
2. **Use different secrets per environment** - Never share secrets between dev/prod
3. **Rotate secrets regularly** - Especially OIDC client secrets and APIM keys
4. **Use Azure Key Vault in production** - For additional secret protection
5. **Limit access** - Only give deployment credentials to necessary users/services
