# OIDC Authentication and Delegation

## Overview

The APIM Developer Portal can integrate with external OIDC (OpenID Connect) identity providers like Okta, Auth0, Azure AD B2C, or any OIDC-compliant provider through the delegation pattern. This enables seamless single sign-on (SSO) while maintaining your existing identity infrastructure.

## What is Delegation?

Delegation moves authentication workflows from APIM's built-in user management to an external system you control. Instead of managing users directly in APIM, the Developer Portal redirects authentication requests to your identity provider.

## Why Use OIDC Delegation?

### Business Benefits

- **Unified Identity** - Leverage existing enterprise identity providers
- **Single Sign-On** - Developers use existing credentials across applications
- **Centralized Management** - Manage users in one place, not multiple systems
- **Enhanced Security** - MFA, conditional access, and advanced security features
- **Compliance** - Meet regulatory requirements for identity management

### Technical Benefits

- **Provider Agnostic** - Works with any OIDC-compliant provider
- **Automatic Provisioning** - Users created in APIM on first login
- **Flexible Authentication** - Support complex authentication flows
- **Token Management** - Leverage OAuth 2.0 / OIDC token standards

## Architecture

### High-Level Flow

```
1. User clicks "Sign In" on Developer Portal
   ↓
2. Portal redirects to Delegation Function App
   ↓
3. Function validates APIM signature
   ↓
4. Function redirects to Identity Provider (Okta/Auth0/Azure AD)
   ↓
5. User authenticates with Identity Provider
   ↓
6. Identity Provider redirects back to Function with auth code
   ↓
7. Function exchanges code for tokens
   ↓
8. Function provisions/updates user in APIM
   ↓
9. Function generates APIM SSO token
   ↓
10. User redirected back to Developer Portal (authenticated)
```

### Components

**Developer Portal**
- Initiates authentication requests
- Receives SSO tokens
- Displays authenticated experience

**Delegation Function App**
- Validates APIM requests using HMAC signature
- Orchestrates OAuth/OIDC flow
- Provisions users in APIM
- Generates SSO tokens

**Identity Provider**
- Authenticates users
- Issues OAuth tokens
- Provides user profile information

**APIM Service**
- Validates SSO tokens
- Manages user records
- Controls API access

## Implementation Guide

### Prerequisites

Before implementing OIDC delegation, ensure you have:

- **Azure APIM instance** - Standard tier or higher
- **Identity Provider** - Configured OIDC application
- **Azure Function App** - For hosting delegation logic
- **Shared Secret** - Generated in APIM for signature validation

### Step 1: Configure Identity Provider

Set up an OIDC application in your identity provider:

#### Okta Configuration

```yaml
Application Settings:
  Application Type: Web
  Sign-in redirect URIs:
    - https://delegation-func.azurewebsites.net/api/auth-callback
  Sign-out redirect URIs:
    - https://{apim-name}.developer.azure-api.net
  Assignments: All users or specific groups
  Grant Types:
    - Authorization Code
    - Refresh Token
```

**Required Scopes:**
- `openid` - OIDC authentication
- `profile` - User profile information
- `email` - User email address

**Collect Configuration:**
```
Client ID: {your-client-id}
Client Secret: {your-client-secret}
Issuer URL: https://{your-domain}.okta.com
Authorization Endpoint: https://{your-domain}.okta.com/oauth2/v1/authorize
Token Endpoint: https://{your-domain}.okta.com/oauth2/v1/token
UserInfo Endpoint: https://{your-domain}.okta.com/oauth2/v1/userinfo
```

#### Azure AD B2C Configuration

```yaml
Application Registration:
  Name: APIM Developer Portal
  Redirect URI: https://delegation-func.azurewebsites.net/api/auth-callback
  Implicit grant: ID tokens
  Allow public client flows: No

User Flows:
  Sign-up and sign-in (B2C_1_signupsignin)
  Profile editing (optional)
  Password reset (optional)

Claims:
  - email
  - given_name
  - family_name
  - display_name
```

### Step 2: Deploy Delegation Function App

The delegation function app handles the OAuth flow and user provisioning.

#### Function App Architecture

Three Azure Functions provide the delegation logic:

**1. Delegation Function** (`/api/delegation`)
- Entry point for APIM delegation requests
- Validates HMAC signature from APIM
- Initiates OAuth flow with identity provider

**2. Auth Callback Function** (`/api/auth-callback`)
- Receives OAuth callback from identity provider
- Exchanges authorization code for tokens
- Provisions or updates user in APIM
- Generates APIM SSO token
- Redirects user back to portal

**3. Health Check Function** (`/api/health`)
- Provides health monitoring endpoint
- Validates configuration
- Returns status of dependencies

#### Deployment Options

**Option 1: Use Reference Implementation**

The reference implementation is available at:
[azure-apim-delegation-function-app](https://github.com/shane-m-holland/azure-apim-delegation-function-app)

```bash
# Clone repository
git clone https://github.com/shane-m-holland/azure-apim-delegation-function-app.git
cd azure-apim-delegation-function-app

# Deploy using Azure CLI
az functionapp deployment source config-zip \
  --resource-group <resource-group> \
  --name <function-app-name> \
  --src ./publish.zip
```

**Option 2: Deploy with Bicep**

```bicep
resource functionApp 'Microsoft.Web/sites@2021-03-01' = {
  name: 'delegation-func'
  location: location
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'APIM_SERVICE_NAME'
          value: apimServiceName
        }
        {
          name: 'APIM_DELEGATION_KEY'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/apim-delegation-key/)'
        }
        {
          name: 'OIDC_CLIENT_ID'
          value: oidcClientId
        }
        {
          name: 'OIDC_CLIENT_SECRET'
          value: '@Microsoft.KeyVault(SecretUri=${keyVault.properties.vaultUri}secrets/oidc-client-secret/)'
        }
        {
          name: 'OIDC_ISSUER_URL'
          value: oidcIssuerUrl
        }
        {
          name: 'OIDC_REDIRECT_URI'
          value: 'https://delegation-func.azurewebsites.net/api/auth-callback'
        }
      ]
    }
  }
}
```

#### Configuration Settings

Configure the function app with these environment variables:

```bash
# APIM Settings
APIM_SERVICE_NAME=your-apim-instance
APIM_RESOURCE_GROUP=your-resource-group
APIM_DELEGATION_KEY=<from-apim-delegation-settings>

# OIDC Provider Settings
OIDC_ISSUER_URL=https://your-domain.okta.com
OIDC_CLIENT_ID=<your-client-id>
OIDC_CLIENT_SECRET=<your-client-secret>
OIDC_REDIRECT_URI=https://delegation-func.azurewebsites.net/api/auth-callback
OIDC_SCOPES=openid profile email

# Optional: OIDC Endpoint Override (if auto-discovery doesn't work)
OIDC_AUTHORIZATION_ENDPOINT=https://your-domain.okta.com/oauth2/v1/authorize
OIDC_TOKEN_ENDPOINT=https://your-domain.okta.com/oauth2/v1/token
OIDC_USERINFO_ENDPOINT=https://your-domain.okta.com/oauth2/v1/userinfo

# Azure Settings (for APIM user provisioning)
AZURE_TENANT_ID=<your-tenant-id>
AZURE_SUBSCRIPTION_ID=<your-subscription-id>

# Managed Identity (recommended)
AZURE_CLIENT_ID=<managed-identity-client-id>
```

### Step 3: Configure APIM Delegation

Enable delegation in your APIM instance:

#### Azure Portal Configuration

1. Navigate to your APIM instance in Azure Portal
2. Select **Developer portal** > **Delegation**
3. Enable **Delegate sign-in & sign-up**
4. Set **Delegation endpoint URL**: `https://delegation-func.azurewebsites.net/api/delegation`
5. Generate and copy **Validation key** (save this securely)
6. Save configuration

#### Using Azure CLI

```bash
# Enable delegation
az apim delegation set \
  --resource-group <resource-group> \
  --service-name <apim-service> \
  --sign-in-enabled true \
  --delegation-url "https://delegation-func.azurewebsites.net/api/delegation"

# Generate validation key
az apim delegation validation-key regenerate \
  --resource-group <resource-group> \
  --service-name <apim-service>
```

#### Using Bicep

```bicep
resource portalDelegation 'Microsoft.ApiManagement/service/portalsettings@2021-08-01' = {
  name: 'delegation'
  parent: apimService
  properties: {
    subscriptions: {
      enabled: false  // Keep subscriptions in APIM
    }
    userRegistration: {
      enabled: true  // Delegate user sign-in/sign-up
    }
    url: 'https://delegation-func.azurewebsites.net/api/delegation'
    validationKey: delegationValidationKey
  }
}
```

### Step 4: Configure Managed Identity

Grant the Function App managed identity permissions to manage APIM users:

```bash
# Enable managed identity on Function App
az functionapp identity assign \
  --name <function-app-name> \
  --resource-group <resource-group>

# Get the managed identity principal ID
PRINCIPAL_ID=$(az functionapp identity show \
  --name <function-app-name> \
  --resource-group <resource-group> \
  --query principalId -o tsv)

# Assign API Management Service Contributor role
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "API Management Service Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ApiManagement/service/<apim-service>
```

### Step 5: Test Authentication Flow

1. **Navigate to Developer Portal**
   ```
   https://{apim-name}.developer.azure-api.net
   ```

2. **Click "Sign In"**
   - Portal redirects to delegation function
   - Function validates request and redirects to identity provider

3. **Authenticate with Identity Provider**
   - Enter credentials
   - Complete MFA if required
   - Consent to requested scopes

4. **Verify Redirect**
   - Redirected back to Developer Portal
   - Signed in as authenticated user
   - User profile visible in portal

5. **Check APIM User Creation**
   ```bash
   az apim user list \
     --resource-group <resource-group> \
     --service-name <apim-service> \
     --query "[?email=='user@example.com']"
   ```

## Security Considerations

### Signature Validation

Always validate the HMAC signature from APIM to prevent unauthorized access:

```javascript
const crypto = require('crypto');

function validateSignature(query, validationKey) {
  const { sig, ...params } = query;

  // Build string to sign (sorted parameters)
  const sortedKeys = Object.keys(params).sort();
  const stringToSign = sortedKeys
    .map(key => `${key}=${params[key]}`)
    .join('&');

  // Compute HMAC-SHA512 signature
  const hmac = crypto.createHmac('sha512', Buffer.from(validationKey, 'base64'));
  const computedSig = hmac.update(stringToSign).digest('base64');

  // Compare signatures
  return crypto.timingSafeEqual(
    Buffer.from(sig, 'base64'),
    Buffer.from(computedSig, 'base64')
  );
}
```

### Key Management

- **Store secrets in Azure Key Vault** - Never commit secrets to source control
- **Use managed identities** - Avoid storing Azure credentials
- **Rotate keys regularly** - Update delegation validation key and client secrets
- **Limit Function App access** - Restrict network access where possible

### HTTPS Everywhere

- **Enforce HTTPS** - All endpoints must use TLS
- **Validate certificates** - Don't disable certificate validation
- **Use secure cookies** - Set secure and httpOnly flags

### Token Security

- **Validate token claims** - Check issuer, audience, expiration
- **Use short-lived tokens** - Configure appropriate token lifetimes
- **Don't log tokens** - Redact sensitive data from logs

## Troubleshooting

### Common Issues

#### Signature Validation Failures

**Symptoms:**
- "Invalid signature" errors in function logs
- Users redirected back to portal without authentication

**Causes:**
- Incorrect validation key configured
- Parameter encoding issues
- Clock skew between systems

**Solutions:**
```bash
# Regenerate validation key
az apim delegation validation-key regenerate \
  --resource-group <rg> \
  --service-name <apim>

# Update Function App configuration
az functionapp config appsettings set \
  --name <function-app> \
  --resource-group <rg> \
  --settings "APIM_DELEGATION_KEY=<new-key>"
```

#### OIDC Provider Connection Failures

**Symptoms:**
- Timeout errors during authentication
- "Unable to connect to identity provider" errors

**Causes:**
- Incorrect OIDC configuration endpoints
- Network connectivity issues
- Invalid client credentials

**Solutions:**
```bash
# Test OIDC discovery endpoint
curl https://your-domain.okta.com/.well-known/openid-configuration

# Verify client credentials
# Test with curl or Postman

# Check Function App logs
az functionapp log tail \
  --name <function-app> \
  --resource-group <rg>
```

#### User Provisioning Failures

**Symptoms:**
- User authenticated but not created in APIM
- "Access denied" errors after authentication

**Causes:**
- Missing managed identity permissions
- Invalid APIM configuration
- Network access restrictions

**Solutions:**
```bash
# Verify managed identity has correct role
az role assignment list \
  --assignee <managed-identity-principal-id> \
  --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ApiManagement/service/<apim>

# Test APIM user creation manually
az apim user create \
  --resource-group <rg> \
  --service-name <apim> \
  --user-id test-user \
  --email test@example.com \
  --first-name Test \
  --last-name User
```

### Debugging Tips

1. **Enable Application Insights**
   ```bash
   az functionapp config appsettings set \
     --name <function-app> \
     --resource-group <rg> \
     --settings "APPINSIGHTS_INSTRUMENTATIONKEY=<key>"
   ```

2. **Review Function Logs**
   ```bash
   az functionapp log tail \
     --name <function-app> \
     --resource-group <rg>
   ```

3. **Test OIDC Flow Manually**
   - Use browser dev tools to inspect redirects
   - Check network tab for failed requests
   - Review response headers and payloads

4. **Validate APIM Configuration**
   ```bash
   az apim delegation show \
     --resource-group <rg> \
     --service-name <apim>
   ```

## Advanced Scenarios

### Custom User Attributes

Map additional claims from identity provider to APIM user properties:

```javascript
async function provisionUser(userInfo, apimClient) {
  const user = {
    email: userInfo.email,
    firstName: userInfo.given_name,
    lastName: userInfo.family_name,
    // Map custom claims
    note: JSON.stringify({
      department: userInfo.department,
      employeeId: userInfo.employee_id,
      roles: userInfo.roles
    })
  };

  return await apimClient.user.createOrUpdate(userId, user);
}
```

### Group Membership Synchronization

Automatically assign users to APIM groups based on identity provider roles:

```javascript
async function syncUserGroups(userInfo, userId, apimClient) {
  const idpRoles = userInfo.roles || [];

  // Map identity provider roles to APIM groups
  const groupMappings = {
    'api-admin': 'administrators',
    'api-developer': 'developers',
    'api-partner': 'partners'
  };

  for (const [idpRole, apimGroup] of Object.entries(groupMappings)) {
    if (idpRoles.includes(idpRole)) {
      await apimClient.groupUser.create(apimGroup, userId);
    }
  }
}
```

### Multi-Tenant Support

Support multiple identity providers for different user populations:

```javascript
function selectIdentityProvider(context) {
  const domain = context.req.query.domain;

  // Route to appropriate provider based on domain
  const providers = {
    'internal': {
      issuer: 'https://login.microsoftonline.com/tenant-id',
      clientId: process.env.AZURE_AD_CLIENT_ID
    },
    'partners': {
      issuer: 'https://partners.okta.com',
      clientId: process.env.OKTA_CLIENT_ID
    }
  };

  return providers[domain] || providers['internal'];
}
```

## Monitoring and Operations

### Key Metrics

Monitor these metrics to ensure healthy operation:

- **Authentication Success Rate** - Track successful logins
- **Provisioning Success Rate** - User creation in APIM
- **Response Times** - Delegation function performance
- **Error Rates** - Authentication and provisioning errors

### Application Insights Queries

```kusto
// Authentication success rate
requests
| where name == "Delegation" or name == "AuthCallback"
| summarize
    Total = count(),
    Success = countif(success == true),
    SuccessRate = 100.0 * countif(success == true) / count()
  by bin(timestamp, 1h)

// Average authentication duration
requests
| where name == "AuthCallback"
| summarize AvgDuration = avg(duration) by bin(timestamp, 5m)

// Error analysis
exceptions
| where outerMessage contains "OIDC" or outerMessage contains "APIM"
| summarize Count = count() by outerMessage
| order by Count desc
```

### Alerting

Configure alerts for critical scenarios:

```bash
# Alert on high error rate
az monitor metrics alert create \
  --name "delegation-high-error-rate" \
  --resource <function-app-resource-id> \
  --condition "avg Percentage >= 5" \
  --window-size 5m \
  --evaluation-frequency 1m
```

## Best Practices

### Configuration
- Use Key Vault for all secrets
- Enable managed identity for Azure access
- Configure proper CORS settings
- Set appropriate timeout values

### Security
- Validate all signatures
- Use HTTPS everywhere
- Implement rate limiting
- Log security events

### Performance
- Cache OIDC discovery metadata
- Use connection pooling
- Implement retry logic with backoff
- Monitor response times

### Operations
- Enable Application Insights
- Configure alerts for failures
- Automate deployment
- Document configuration

## Related Documentation

- [Developer Portal Overview](dev-portal-overview.md) - Portal capabilities
- [User Management](user-management.md) - Managing users and groups
- [Monitoring and Observability](../monitoring-and-observability/) - Operational insights

## External Resources

- [APIM Delegation Documentation](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-setup-delegation)
- [OpenID Connect Specification](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0 Authorization Code Flow](https://oauth.net/2/grant-types/authorization-code/)
- [Reference Implementation Repository](https://github.com/shane-m-holland/azure-apim-delegation-function-app)
