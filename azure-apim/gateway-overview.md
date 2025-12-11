# Azure API Management (APIM)

## Key Component overview - Gateway

An **APIM Gateway** is the component that receives API requests, applies policies (like security, throttling, transformation), and routes them to the appropriate backend services.  

Key functions of the gateway:

- **Authentication & Authorization**: Validates subscription keys, OAuth 2.0 tokens, and client certificates.
- **Request Routing**: Forwards valid requests to the correct backend service.
- **Policy Enforcement**: Applies policies to requests and responses (rate limiting, caching, transformation, validation).
- **Observability**: Collects metrics, logs, and traces for monitoring and diagnostics.
- **Abstraction**: Shields backend systems from direct exposure and provides a unified API surface.

---

## Policies at the Gateway

Policies are executed at the gateway level in request/response flows:

- **Inbound policies**: Applied before the request is sent to the backend (e.g., authentication, IP restrictions, request transformation).
- **Backend policies**: Applied during communication with the backend (e.g., retry, timeout).
- **Outbound policies**: Applied to the response before returning to the consumer (e.g., response transformation, header injection).
- **On-error policies**: Applied when errors occur (e.g., return custom error messages).

---

## Scaling and Performance

- The gateway is scaled by adding units in the APIM tier.
- Premium tier supports multi-region deployment for global resiliency.
- Use **caching policies** to reduce backend load.
- Monitor **latency and throughput** to identify bottlenecks.

---

## Security Considerations

- Enforce **HTTPS-only** communication.
- Require **subscription keys** or OAuth tokens for access control.
- Use **mutual TLS (mTLS)** for certificate-based client authentication.
- Apply **rate limits and quotas** to prevent abuse.
- Isolate traffic using **VNET integration** or private endpoints when needed.

---

### 1. Managed Gateway (Default)

#### Hosted by Microsoft Azure

When you deploy an APIM instance, Azure provides a **Managed Gateway** by default — fully hosted, monitored, and scaled by Microsoft.

#### Pros

- Secure by default with built-in DDoS protection
- Highly available and globally scalable
- No infrastructure to manage
- Integrated monitoring (App Insights, Azure Monitor)

#### Cons

- Requires internet access (unless using VNet Premium tier)
- Limited customization compared to self-hosted
- No control over infrastructure or network behavior

### 2. Self-Hosted Gateway (Hybrid)

#### Runs outside Azure — in your **own environment**

You can deploy APIM's gateway runtime on:

- On-premises servers
- Docker containers
- Kubernetes clusters (AKS, OpenShift, etc.)
- Edge networks or IoT gateways

#### Pros

- Deploy in **restricted networks** (on-prem or private cloud)
- Edge/local traffic control = **low latency**
- Integrates with private backends not exposed to internet
- Customize deployment, scale, and hosting

#### Cons

- You manage infrastructure, updates, security
- More complex setup and operations
- Needs constant connection to APIM control plane for sync
- Limited built-in telemetry (must configure)

---

## Best Practices for the Gateway

- **Choose the right gateway type**:
  - Use managed gateway for simplicity and public-facing APIs.
  - Use self-hosted gateway for hybrid or on-premises scenarios.
- **Apply consistent policies** across APIs to standardize security and governance.
- **Optimize performance** with caching and throttling policies.
- **Deploy gateways regionally** to reduce latency for consumers.
- **Monitor and log gateway traffic** to identify trends and troubleshoot issues.
- **Automate configuration** of gateways, APIs, and policies using Bicep or ARM templates.

---

## Key Takeaways

- The APIM gateway is the **enforcement point** for security, transformation, and observability in API traffic.
- It can run as a **managed service in Azure** or as a **self-hosted gateway** in hybrid/multi-cloud environments.
- Policies are applied at the gateway to control request/response behavior.
- Following best practices ensures APIs are delivered securely, consistently, and with high performance.
