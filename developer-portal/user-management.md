# User Management

## Overview

This guide covers user and group management in the Azure API Management Developer Portal, including identity integration, access control, and subscription management.

## User Types

### Built-in Users

APIM provides built-in user types:

- **Administrators** - Full access to Azure Portal and Developer Portal
- **Developers** - Standard developer portal users
- **Guests** - Unauthenticated users with limited access

### Identity Sources

Users can authenticate through:

1. **Native APIM Identity** - Built-in user database
2. **Azure Active Directory** - Enterprise identity
3. **Azure AD B2C** - Consumer identity
4. **Delegation** - External OIDC providers (see [OIDC Delegation](oidc-delegation.md))

## Groups

### Built-in Groups

APIM includes default groups:

- **Administrators** - Full access to all APIs and portal configuration
- **Developers** - Standard access to published APIs
- **Guests** - Read-only access to public APIs

### Custom Groups

Create custom groups for fine-grained access control:

```bash
az apim group create \
  --resource-group <rg> \
  --service-name <apim> \
  --group-id partners \
  --display-name "Partner Developers"
```

### Group-Based Access

Control API visibility and product access by group membership:

**Product Configuration:**
```yaml
product:
  name: premium-api
  groups:
    - administrators
    - partners
  subscriptionRequired: true
```

## User Lifecycle

### User Creation

**Manual Creation:**
```bash
az apim user create \
  --resource-group <rg> \
  --service-name <apim> \
  --user-id john-doe \
  --email john.doe@example.com \
  --first-name John \
  --last-name Doe \
  --password "<secure-password>"
```

**Self-Service Registration:**
Users can sign up through the Developer Portal if enabled:
- Enable in Portal settings
- Configure email verification
- Optional admin approval

**Automated Provisioning:**
Use delegation to auto-provision users from external identity providers.

### User Updates

Update user properties:

```bash
az apim user update \
  --resource-group <rg> \
  --service-name <apim> \
  --user-id john-doe \
  --first-name "John" \
  --last-name "Smith"
```

### User Deletion

Remove users and their subscriptions:

```bash
az apim user delete \
  --resource-group <rg> \
  --service-name <apim> \
  --user-id john-doe \
  --delete-subscriptions true
```

## Subscription Management

### Subscription Keys

Each subscription provides:
- **Primary Key** - Active subscription key
- **Secondary Key** - For key rotation without downtime

### Key Rotation

Best practice for rotating keys:

1. Generate new secondary key
2. Update applications to use secondary key
3. Regenerate primary key
4. Update remaining applications to use new primary key

```bash
# Regenerate key
az apim subscription regenerate-key \
  --resource-group <rg> \
  --service-name <apim> \
  --subscription-id <subscription-id> \
  --key-type primary
```

### Subscription Approval

Configure approval requirements per product:

**Auto-Approval:**
```yaml
product:
  approvalRequired: false
```

**Manual Approval:**
```yaml
product:
  approvalRequired: true
```

Approve subscriptions:
```bash
az apim subscription update \
  --resource-group <rg> \
  --service-name <apim> \
  --subscription-id <id> \
  --state active
```

## Access Control Patterns

### Pattern 1: Tiered Access

Different products for different access levels:

```yaml
products:
  starter:
    displayName: "Starter Plan"
    groups: [developers]
    quota: 10000

  professional:
    displayName: "Professional Plan"
    groups: [developers]
    approvalRequired: true
    quota: 100000

  enterprise:
    displayName: "Enterprise Plan"
    groups: [partners]
    approvalRequired: true
    quota: 1000000
```

### Pattern 2: API-Level Restrictions

Assign specific APIs to groups:

```yaml
api:
  name: internal-api
  groups:
    - administrators
    - internal-developers
```

### Pattern 3: Operation-Level Security

Apply policies at operation level:

```xml
<policies>
  <inbound>
    <check-header name="X-Admin-Key" failed-check-httpcode="403">
      <value>@(context.Variables.GetValueOrDefault("admin-key", ""))</value>
    </check-header>
  </inbound>
</policies>
```

## Best Practices

### User Management

1. **Use external identity providers** - Delegate to enterprise identity
2. **Enable email verification** - Prevent fake accounts
3. **Regular access reviews** - Audit user permissions quarterly
4. **Automate provisioning** - Use delegation for user creation
5. **Document user types** - Clear guidelines for access levels

### Group Management

1. **Follow least privilege** - Grant minimum necessary access
2. **Use descriptive names** - Clear group purpose
3. **Document group purpose** - Maintain group documentation
4. **Regular review** - Audit group membership
5. **Automate assignments** - Sync from identity provider

### Subscription Management

1. **Rotate keys regularly** - 90-day rotation policy
2. **Monitor key usage** - Track active subscriptions
3. **Limit subscriptions** - Set max per user/product
4. **Require approval for production** - Manual approval for sensitive APIs
5. **Audit subscription changes** - Log all key operations

## Monitoring and Analytics

### User Analytics

Track user engagement:

```kusto
// Application Insights query
customEvents
| where name == "UserLogin"
| summarize LoginCount = count() by user_AuthenticatedId
| order by LoginCount desc
| take 10
```

### Subscription Analytics

Monitor subscription usage:

```bash
az apim subscription list \
  --resource-group <rg> \
  --service-name <apim> \
  --query "[?state=='active'].{id:name,userId:userId,product:scope}" \
  --output table
```

### Access Auditing

Review access patterns:

```kusto
requests
| where name startswith "API:"
| summarize RequestCount = count() by user_AuthenticatedId, name
| order by RequestCount desc
```

## Troubleshooting

### User Cannot Sign In

**Check:**
- User state is 'active'
- Identity provider configuration correct
- Password not expired
- Account not locked

### User Cannot See APIs

**Check:**
- User assigned to correct groups
- APIs published to appropriate products
- Products visible to user's groups

### Subscription Not Working

**Check:**
- Subscription state is 'active'
- Subscription key correct
- Product includes desired API
- Rate limits not exceeded

## Related Documentation

- [OIDC Delegation](oidc-delegation.md) - External authentication
- [Developer Portal Overview](dev-portal-overview.md) - Portal features
- [API Products](../azure-apim/api-products.md) - Product configuration
