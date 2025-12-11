# Monitoring and Observability

This section provides comprehensive guidance on monitoring, logging, and observability for Azure API Management, including integration with Azure Monitor, Application Insights, and third-party tools.

## Table of Contents

- [Overview](./overview.md) - Introduction to monitoring and observability in APIM
- [Azure Monitor Integration](./azure-monitor.md) - Platform metrics and alerts
- [Application Insights](./application-insights.md) - Detailed telemetry and distributed tracing
- [Diagnostic Logging](./diagnostic-logging.md) - Log collection, destinations, and analysis
- [Dynatrace Integration](./dynatrace-integration.md) - Third-party observability platform integration
- [Developer Portal Analytics](./developer-portal-analytics.md) - Monitoring portal usage and performance
- [Cost Optimization](./cost-optimization.md) - Managing monitoring costs effectively

## Why Monitoring Matters

Effective monitoring and observability are critical for:

- **Detecting Issues Early** - Identify performance bottlenecks and errors before they impact users
- **Understanding Usage Patterns** - Track API consumption and consumer behavior
- **Enforcing SLAs** - Ensure APIs meet performance and availability targets
- **Capacity Planning** - Make data-driven decisions about scaling
- **Security** - Detect anomalous behavior and potential security threats
- **Troubleshooting** - Quickly diagnose and resolve issues

## Observability Stack

The APIM observability stack consists of:

### Azure Native Tools

1. **Azure Monitor** - Platform metrics, alerts, and dashboards
2. **Application Insights** - Deep application telemetry and tracing
3. **Log Analytics** - Queryable log storage with KQL
4. **Azure Dashboards** - Visualization and monitoring views

### Third-Party Integration

- **Dynatrace** - End-to-end observability with AI-powered insights
- **Splunk** - Log aggregation and analysis
- **Datadog** - Unified monitoring platform
- **Event Hub** - Stream telemetry to external systems

## Key Metrics to Monitor

### Gateway Metrics

- **Request Volume** - Total API calls per time period
- **Response Time** - Average and percentile latency (p50, p95, p99)
- **Error Rates** - 4xx client errors and 5xx server errors
- **Throughput** - Requests per second capacity
- **Throttling Events** - Blocked requests due to rate limits

### Backend Metrics

- **Backend Response Time** - Time spent waiting for backend services
- **Backend Availability** - Success rate of backend calls
- **Cache Hit Ratio** - Effectiveness of caching policies
- **Dependency Failures** - Backend service errors

### Capacity Metrics

- **Gateway Units** - APIM capacity utilization
- **CPU and Memory** - Resource consumption
- **Network** - Bandwidth usage
- **Storage** - Disk usage for caching and logs

## Logging Strategies

### Diagnostic Logs

APIM can send detailed request/response logs to:

- **Log Analytics** - Query with KQL, build workbooks
- **Event Hubs** - Stream to SIEM or custom processors
- **Blob Storage** - Long-term archival

### Log Levels

- **Verbose** - All request/response details (expensive, use for debugging)
- **Information** - Key events and successful operations
- **Warning** - Degraded performance or potential issues
- **Error** - Failed requests and exceptions

### Sensitive Data

Always mask or exclude:
- Authentication tokens and API keys
- Personally Identifiable Information (PII)
- Backend credentials
- Business-sensitive data

## Alert Configuration

### Critical Alerts

Set up immediate notifications for:

- Error rate > 5% for more than 5 minutes
- p95 latency > 2x baseline
- Gateway capacity > 80%
- Backend availability < 95%
- Any 503 Service Unavailable responses

### Warning Alerts

Monitor trends for:

- Gradual increase in error rates
- Slow degradation in response times
- Approaching rate limit thresholds
- Cache hit ratio declining

## Monitoring by Scenario

| Scenario | Recommended Tools | Key Metrics |
|----------|------------------|-------------|
| **API Performance** | Application Insights | Response time, dependencies, failures |
| **Gateway Health** | Azure Monitor | Capacity, throughput, availability |
| **Usage Analytics** | Log Analytics | Request volume, top APIs, consumers |
| **Cost Control** | Azure Cost Management | Resource consumption, data ingestion |
| **Security Monitoring** | Log Analytics + Sentinel | Failed auth, anomalies, threats |
| **Distributed Tracing** | Application Insights | End-to-end request flow |

## Getting Started

If you're new to monitoring APIM, follow this path:

1. **Enable Basic Metrics** - Start with [Azure Monitor Integration](./azure-monitor.md)
2. **Add Detailed Telemetry** - Configure [Application Insights](./application-insights.md)
3. **Set Up Logging** - Implement [Diagnostic Logging](./diagnostic-logging.md)
4. **Configure Alerts** - Define critical thresholds and notification channels
5. **Build Dashboards** - Create operational and business dashboards
6. **Optimize Costs** - Review [Cost Optimization](./cost-optimization.md) strategies

## Best Practices

### Design Phase

1. **Plan monitoring early** - Include observability in initial architecture
2. **Define SLAs** - Set clear performance and availability targets
3. **Identify key metrics** - Focus on metrics that matter to your business
4. **Consider costs** - Balance telemetry depth with budget

### Implementation

1. **Use sampling** - Reduce costs while maintaining statistical significance
2. **Correlate requests** - Use correlation IDs for distributed tracing
3. **Standardize logging** - Consistent formats across all APIs
4. **Mask sensitive data** - Never log secrets or PII

### Operations

1. **Review regularly** - Check dashboards and trends weekly
2. **Tune alerts** - Adjust thresholds to reduce noise
3. **Act on insights** - Use data to drive optimizations
4. **Document incidents** - Build runbooks from real issues

## Integration Patterns

### Application Insights

```xml
<policies>
    <inbound>
        <set-header name="Request-Id" exists-action="override">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>
    </inbound>
    <backend>
        <forward-request />
    </backend>
    <outbound>
        <trace source="apim-trace" severity="information">
            <message>@($"API: {context.Api.Name}, Operation: {context.Operation.Name}, Duration: {context.Elapsed.TotalMilliseconds}ms")</message>
        </trace>
    </outbound>
</policies>
```

### Event Hub Streaming

For high-volume environments, stream logs to Event Hub for:
- Real-time processing
- Integration with Dynatrace or Splunk
- Custom analytics pipelines
- Long-term archival

## Cost Management

Monitoring can become expensive. Control costs by:

1. **Sampling** - Use adaptive sampling (5-20%) for high-volume APIs
2. **Retention** - Set appropriate retention periods (30-90 days for most logs)
3. **Filtering** - Log only what's necessary
4. **Tiering** - Use cheaper storage for historical data

See [Cost Optimization](./cost-optimization.md) for detailed strategies.

## Compliance and Security

### Audit Requirements

For regulated industries:
- Enable activity logs for all APIM changes
- Retain diagnostic logs per compliance requirements
- Implement tamper-proof log storage
- Regular log reviews and audits

### Security Monitoring

Monitor for:
- Failed authentication attempts
- Unusual traffic patterns
- Access from unexpected locations
- Abnormal error rates

## Related Topics

- **Azure APIM** - See [Azure APIM](../azure-apim/) for core concepts
- **DevOps** - See [DevOps](../dev-ops/) for automated monitoring setup
- **Developer Portal** - See [Developer Portal Analytics](./developer-portal-analytics.md) for portal-specific monitoring

## Additional Resources

- [Azure Monitor Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/)
- [Application Insights for APIM](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights)
- [APIM Diagnostic Logging](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
