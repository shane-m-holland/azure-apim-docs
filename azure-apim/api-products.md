# Azure API Management (APIM)

## Products - Overview

Products are a way to bundle one or more APIs together for publishing.

Key aspects:

- Products control how APIs are exposed to developers.
- Products can be published (available to developers) or unpublished (not yet visible).
- Access to a product is managed through subscriptions.

APIM supports **two approaches** for handling product subscriptions:

1. **Native Subscription** – Handled entirely within APIM.
2. **Delegated Subscription** – Redirects subscription requests to an external system for custom handling.

## 1. Native Product Subscription

### How It Works

- A developer signs in to the **Developer Portal**.
- They view available products (visibility based on group membership).
- They click **"Subscribe"** on a product page.
- Depending on product settings:
  - **Instant Approval** – Keys are generated immediately.
  - **Manual Approval** – Admin approves the request in the Azure Portal before keys are issued.

### Key Features

- **No coding required** — Fully managed by APIM.
- **Automatic key management** — Generates primary and secondary subscription keys.
- **Simple approval option** — Toggle between instant or manual.
- **Policy integration** — Apply quotas, rate limits, and security settings at the product level.

### Pros

- Fast to implement.
- Low maintenance overhead.
- Seamless Developer Portal integration.

### Cons

- Limited customization (UI and workflow are fixed).
- Cannot integrate with external systems like billing or CRM.
- Single-step approval only.

### Best Use Cases

- Internal APIs with trusted audiences.
- Public APIs with low access risk.
- Simple partner integrations without contractual complexity.

### Microsoft Documentation

- [Create and manage subscriptions](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-create-subscriptions)
- [Add and configure products](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-add-products)

## 2. Delegated Product Subscription

### How It Works

Delegation moves the subscription approval process to **your external system**.

**Flow:**

1. Developer clicks **"Subscribe"** in the Developer Portal.
2. APIM redirects the request to your configured **Delegation URL**.
3. Your system:
   - Validates the request.
   - Applies business rules (e.g., payment verification, legal approval).
4. If approved:
   - Your system calls the **APIM REST API** to create the subscription.
   - Keys are generated and returned to the developer.
5. User is redirected back to the Developer Portal (or another page you specify).

### Configuration Steps

1. **Enable Delegation**
   - Azure Portal → APIM instance → **Developer portal** → **Delegation**.
2. **Set Delegation URL**
   - Endpoint of your external system that will handle subscription requests.
3. **Secure with HMAC Signature**
   - Configure a **shared secret** in APIM.
   - APIM sends an HMAC signature with each request; your backend verifies it.
4. **Implement External Endpoint**
   - Handle incoming parameters (`operation`, `productId`, `userId`, `returnUrl`, `sig`).
   - Perform your custom workflow.
   - Call APIM REST API to approve/deny subscription.

### Pros

- Fully customizable approval workflows.
- Integrates with billing, CRM, ERP, or identity platforms.
- Complete UI/UX control.
- Supports advanced security and compliance checks.

### Cons

- Requires development and hosting of an external service.
- Higher operational complexity.
- More failure points (external system + APIM dependency).

### Best Use Cases

- Paid API subscriptions requiring payment verification.
- Partner APIs needing contractual/legal approval before access.
- Highly regulated APIs with strict compliance checks.
- Membership-based APIs tied to external directories or user databases.

### Microsoft Docs

- [Setup Delegation in Developer Portal](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-setup-delegation)
- [Create or Update Subscription via REST API](https://learn.microsoft.com/en-us/rest/api/apimanagement/current-ga/subscription/create-or-update)

---

## Native vs. Delegated — Side-by-Side Feature Matrix

| Feature / Criteria | **Native Subscription** | **Delegated Subscription** |
|--------------------|------------------------|----------------------------|
| **Definition** | Subscription requests handled entirely within APIM’s workflow. | Subscription requests redirected to an external system you control. |
| **Setup Effort** | Minimal — configure in Azure Portal. | Higher — requires custom endpoint + API integration. |
| **Approval Workflow** | Instant or manual (single-step). | Fully customizable, multi-step possible. |
| **Integration with External Systems** | No. | Yes (billing, CRM, ERP, identity). |
| **User Experience Control** | Fixed Developer Portal flow. | Fully customizable UI/UX. |
| **Security Checks** | APIM policies only. | Custom rules + APIM security. |
| **Key Provisioning** | Automatic by APIM. | Must call APIM REST API after approval. |
| **Maintenance Overhead** | Low. | Higher — maintain external service. |
| **Speed of Implementation** | Fast. | Slower — requires dev work. |
| **Customization Level** | Low. | High. |

---

## Decision Guidance

| Choose **Native** when: | Choose **Delegated** when: |
|-------------------------|---------------------------|
| APIs are internal or low-risk. | You require complex workflows (multi-step, approval, payment). |
| You need quick setup. | You want deep integration with external systems. |
| Minimal customization is fine. | You want full UI control. |

---

## Security Notes for Delegated Subscription

- Always validate the HMAC signature from APIM before processing.
- Use HTTPS for your delegation endpoint.
- Keep shared keys secret and rotate periodically.
- Log subscription attempts for auditing.

---
