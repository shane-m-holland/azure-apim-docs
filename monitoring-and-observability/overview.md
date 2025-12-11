# Monitoring and Observability - Overview

## Introduction

Monitoring and observability are critical for ensuring that Azure API Management (APIM) instances and the APIs they host are reliable, performant, and secure. APIM provides built-in monitoring capabilities and integrates with Azure observability services such as **Azure Monitor**, **Application Insights**, **Event Hub**, and **Log Analytics**.

Effective monitoring helps teams:

- Detect performance bottlenecks and errors before they impact users
- Track API usage patterns and consumer behavior
- Enforce SLAs and compliance requirements
- Provide actionable insights to both technical and business stakeholders
- Make data-driven decisions about capacity and optimization

## Observability vs. Monitoring

### Monitoring
**Monitoring** is the practice of collecting and analyzing predefined metrics and logs to track system health and performance.

**Characteristics:**
- Known-knowns: tracking expected metrics
- Dashboard and alert-based
- Reactive: respond to defined thresholds

### Observability
**Observability** is the ability to understand internal system state by examining outputs, enabling exploration of unknown-unknowns.

**Characteristics:**
- Ask arbitrary questions of your system
- Correlate events across distributed services
- Proactive: discover issues you didn't anticipate

**APIM combines both:** predefined metrics (monitoring) with detailed tracing and logging (observability).

## Azure Observability Services

### Azure Monitor

**Purpose:** Platform-level metrics and alerting

**Capabilities:**
- Collects platform metrics (CPU, memory, request throughput)
- Integrates with dashboards for real-time visibility
- Supports alerts based on thresholds or anomalies
- Provides insights into resource health

**Use Cases:**
- High-level health monitoring
- Capacity planning
- Cost tracking
- Alert management

### Application Insights

**Purpose:** Deep application telemetry and distributed tracing

**Capabilities:**
- Provides detailed telemetry for requests and responses
- Tracks dependencies (backend calls) and latency
- Enables distributed tracing for end-to-end diagnostics
- Offers intelligent alerting with anomaly detection

**Use Cases:**
- Performance troubleshooting
- Dependency mapping
- Request correlation across microservices
- User analytics

### Log Analytics Workspace

**Purpose:** Centralized log storage and querying

**Capabilities:**
- Queryable logs using Kusto Query Language (KQL)
- Long-term log retention
- Cross-resource queries
- Integration with Azure Sentinel for security

**Use Cases:**
- Troubleshooting and root cause analysis
- Compliance and audit trails
- Trend analysis
- Custom reporting

### Event Hub

**Purpose:** Real-time event streaming

**Capabilities:**
- Streams telemetry for ingestion into SIEM or custom platforms
- High-throughput event processing
- Integration with third-party tools
- Real-time analytics pipelines

**Use Cases:**
- Streaming to Dynatrace, Splunk, or Datadog
- Custom analytics processing
- Real-time dashboards
- Audit log streaming

## Built-in Observability Features

| Feature | Description | Cost | Best Use |
|---------|-------------|------|----------|
| **Azure Monitor Integration** | Enables metric collection and dashboarding | Included (metrics free up to limits) | Monitor health, availability, and usage trends of APIM |
| **Diagnostic Logging** | Send request/response logs to Log Analytics, Event Hubs, or Storage | Billed per destination | Troubleshooting, audit trails, API usage analysis |
| **Application Insights** | Deep diagnostics for live traffic and backend dependencies | Billed per telemetry volume | Trace distributed calls, visualize request timelines |
| **Metrics in Azure Portal** | Built-in charts for throughput, latency, and failures | Included | Quick performance checks, alert setup |
| **Alerts** | Threshold-based alerts via Azure Monitor | Included | Notify on failures, performance drops, or usage spikes |
| **Activity Logs** | Track operations on the APIM instance itself (e.g., scaling, config) | Included | Audit administrative changes and system events |

## Key Metrics to Monitor

### Gateway Performance

- **Request Volume** - Total calls per API or product
- **Response Time** - Average and percentile latency (p50, p95, p99)
- **Error Rates** - 4xx (client errors) and 5xx (server errors)
- **Throughput** - Requests per second
- **Concurrent Connections** - Active connections to gateway

### Backend Health

- **Backend Duration** - Time spent in backend calls
- **Backend Errors** - Failed backend requests
- **Backend Timeout Rate** - Requests exceeding timeout thresholds
- **Dependency Availability** - Uptime of backend services

### Policy Execution

- **Throttling Events** - Requests blocked due to quotas or rate limits
- **Cache Hit Ratio** - Effectiveness of caching policies
- **Policy Failures** - Errors in policy execution
- **Transformation Time** - Time spent in policy processing

### Capacity

- **Capacity Utilization** - Gateway unit usage percentage
- **CPU Usage** - Processor utilization
- **Memory Usage** - RAM consumption
- **Network Throughput** - Bandwidth usage

## Monitoring by Role

### Developers

**Focus:** API performance and errors

**Key Metrics:**
- Response times for their APIs
- Error rates and types
- Consumer feedback and issues
- API usage patterns

**Tools:**
- Application Insights
- API-specific dashboards
- Log Analytics queries

### Operations Team

**Focus:** Platform health and availability

**Key Metrics:**
- Gateway capacity and scaling
- Overall error rates
- Backend health
- Alert management

**Tools:**
- Azure Monitor dashboards
- Alert rules and action groups
- Resource health

### Business Stakeholders

**Focus:** Usage trends and business impact

**Key Metrics:**
- API adoption rates
- Top consumers and APIs
- Geographic distribution
- Trend analysis

**Tools:**
- Custom Power BI dashboards
- Executive summaries
- Usage reports

### Security Team

**Focus:** Threats and compliance

**Key Metrics:**
- Failed authentication attempts
- Anomalous traffic patterns
- Policy violations
- Audit logs

**Tools:**
- Azure Sentinel
- Log Analytics
- Security alerts

## Logging and Diagnostics Destinations

APIM supports multiple destinations for diagnostic logs:

| Destination | Description | Best Use | Retention | Cost Model |
|-------------|-------------|----------|-----------|------------|
| **Azure Monitor Logs (Log Analytics)** | Centralized queryable logging and dashboards | Troubleshooting, analysis, custom queries | Configurable (30-730 days) | Per GB ingested + retention |
| **Event Hubs** | Stream to external systems (e.g., SIEM, Dynatrace, Splunk) | Real-time processing, third-party integration | N/A (streaming) | Per throughput unit |
| **Blob Storage** | Long-term retention and archival | Compliance, historical analysis | Years | Per GB stored per month |

## What to Log

### Standard Logging

For most environments, log:
- Request method, path, and query parameters
- Response status codes
- Duration and timestamps
- Client IP and user agent
- Subscription key (hashed or truncated)
- Errors and exceptions

### Verbose Logging

For troubleshooting, temporarily enable:
- Request and response headers
- Request and response bodies (masked)
- Policy execution details
- Backend call details

### What NOT to Log

Never log:
- Full authentication tokens or API keys
- Passwords or credentials
- Personally Identifiable Information (PII) unless required
- Credit card or payment information
- Any data subject to regulatory protection

## Monitoring Costs

Monitoring and observability can become expensive. Key cost drivers:

1. **Application Insights Data Ingestion** - Charged per GB (first 5 GB free per month)
2. **Log Analytics Ingestion** - Charged per GB ingested
3. **Log Retention** - Additional cost beyond 30 days
4. **Event Hub Throughput** - Charged per throughput unit
5. **Storage** - Minimal cost for blob archival

### Cost Optimization Strategies

- Use **sampling** to reduce data volume (5-20% sampling typically sufficient)
- Set appropriate **retention periods** (30-90 days for most logs)
- **Filter** logs to exclude unnecessary data
- Use **blob storage** for long-term archival
- **Alert** on anomalies rather than collecting everything

See [Cost Optimization](./cost-optimization.md) for detailed strategies.

## Getting Started Checklist

- [ ] Enable Azure Monitor metrics collection
- [ ] Create Application Insights resource
- [ ] Configure diagnostic settings for APIM
- [ ] Set up Log Analytics workspace
- [ ] Define critical alert rules
- [ ] Create operational dashboard
- [ ] Implement sampling strategy
- [ ] Document logging standards
- [ ] Train team on querying and troubleshooting
- [ ] Review and optimize costs monthly

## Next Steps

1. **Configure Azure Monitor** - Start with [Azure Monitor Integration](./azure-monitor.md)
2. **Add Application Insights** - See [Application Insights](./application-insights.md)
3. **Enable Diagnostic Logs** - Review [Diagnostic Logging](./diagnostic-logging.md)
4. **Integrate Third-Party Tools** - Explore [Dynatrace Integration](./dynatrace-integration.md)
5. **Monitor Developer Portal** - See [Developer Portal Analytics](./developer-portal-analytics.md)

## Additional Resources

- [Azure Monitor Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/)
- [APIM Diagnostics Overview](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
- [Application Insights for APIM](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights)
