# Application Insights Integration

## Overview

Application Insights is Azure's application performance management (APM) service that provides deep telemetry for API Management. It enables distributed tracing, dependency tracking, and detailed diagnostics beyond what Azure Monitor provides.

## Why Use Application Insights with APIM?

**Azure Monitor provides:**
- High-level platform metrics
- Capacity and availability monitoring
- Basic alerting

**Application Insights adds:**
- Request-level details (headers, durations, results)
- Distributed tracing across services
- Dependency mapping and latency
- Custom events and metrics
- Intelligent anomaly detection
- User and session analytics

## Configuration

### Creating Application Insights Resource

**Via Azure Portal:**

1. Navigate to Azure Portal
2. Create new resource → **Application Insights**
3. Configure:
   - Resource Group: Same as APIM
   - Name: `apim-{environment}-insights`
   - Region: Same as APIM
   - Workspace-based: Select or create Log Analytics workspace
4. Create the resource
5. Copy **Instrumentation Key** or **Connection String**

### Connecting APIM to Application Insights

**Option 1: Azure Portal**

1. Navigate to APIM instance
2. Select **Application Insights** under Monitoring
3. Click **Add**
4. Select the Application Insights resource
5. Configure:
   - Enable: All APIs or specific APIs
   - Sampling: Percentage (recommended: 20%)
   - Always log errors: Yes
6. Save configuration

**Option 2: Bicep/ARM Template**

```bicep
resource apimLogger 'Microsoft.ApiManagement/service/loggers@2023-03-01-preview' = {
  parent: apimService
  name: 'appinsights-logger'
  properties: {
    loggerType: 'applicationInsights'
    credentials: {
      instrumentationKey: applicationInsights.properties.InstrumentationKey
    }
    isBuffered: true
    resourceId: applicationInsights.id
  }
}

resource apiDiagnostics 'Microsoft.ApiManagement/service/apis/diagnostics@2023-03-01-preview' = {
  parent: api
  name: 'applicationinsights'
  properties: {
    alwaysLog: 'allErrors'
    loggerId: apimLogger.id
    sampling: {
      samplingType: 'fixed'
      percentage: 20
    }
    frontend: {
      request: {
        headers: ['Content-Type', 'User-Agent']
        body: {
          bytes: 512
        }
      }
      response: {
        headers: ['Content-Type']
        body: {
          bytes: 512
        }
      }
    }
    backend: {
      request: {
        headers: ['Content-Type']
      }
      response: {
        headers: ['Content-Type']
      }
    }
  }
}
```

## Telemetry Collected

### Request Telemetry

**Automatically captured:**
- Request URL and method
- Response status code
- Duration (total and backend)
- Timestamp
- Success/failure flag
- Client IP and user agent

**Configurable:**
- Request headers (specify which ones)
- Request body (bytes to capture)
- Response headers
- Response body (bytes to capture)

### Dependency Telemetry

Tracks calls from APIM to backend services:
- Backend URL
- Duration
- Success/failure
- Response codes
- Correlation with parent request

### Exception Telemetry

Captures errors and exceptions:
- Exception type and message
- Stack trace
- Request context
- Custom properties

### Custom Telemetry

Add custom events and metrics via policies:
- Business events (e.g., "API key generated")
- Custom dimensions
- Performance counters
- Application-specific metrics

## Sampling

Sampling reduces telemetry volume and costs while maintaining statistical significance.

### Sampling Types

**Fixed Sampling:**
```
20% sampling = 1 in 5 requests logged
```

**Adaptive Sampling:**
- Automatically adjusts sampling rate
- Maintains data volume target
- Increases sampling during low traffic
- Decreases during high traffic

### Recommended Sampling Rates

| Environment | API Volume | Sampling Rate |
|-------------|------------|---------------|
| **Development** | Low | 100% |
| **Testing** | Medium | 50% |
| **Production** | High | 10-20% |
| **Production** | Very High (>1M/day) | 1-5% |

### Always Log Errors

Enable **Always Log Errors** to capture all failures regardless of sampling:

```xml
<diagnostics>
  <always-log>allErrors</always-log>
</diagnostics>
```

## Querying Telemetry

### Kusto Query Language (KQL)

Application Insights uses KQL for querying. Access via:
- Application Insights → Logs
- Log Analytics workspace

### Common Queries

**Request Volume Over Time:**
```kusto
requests
| where timestamp > ago(24h)
| summarize count() by bin(timestamp, 5m)
| render timechart
```

**Top 10 Slowest Operations:**
```kusto
requests
| where timestamp > ago(1h)
| summarize avg(duration), p95=percentile(duration, 95) by operation_Name
| top 10 by p95 desc
```

**Error Rate by API:**
```kusto
requests
| where timestamp > ago(24h)
| summarize Total=count(), Failures=countif(success == false) by operation_Name
| extend ErrorRate = (Failures * 100.0) / Total
| where ErrorRate > 0
| order by ErrorRate desc
```

**Backend Performance:**
```kusto
dependencies
| where timestamp > ago(1h)
| summarize avg(duration), p95=percentile(duration, 95) by target
| order by p95 desc
```

**Failed Requests with Details:**
```kusto
requests
| where timestamp > ago(1h) and success == false
| project timestamp, operation_Name, resultCode, duration, url
| order by timestamp desc
```

## Distributed Tracing

### End-to-End Transaction Tracking

Application Insights automatically correlates:
1. Client request to APIM
2. APIM processing (policies, transformations)
3. Backend API call
4. Backend processing
5. Response through APIM
6. Response to client

### Correlation IDs

APIM automatically propagates correlation headers:
- `Request-Id` - Unique request identifier
- `Request-Context` - Application Insights context
- `traceparent` - W3C Trace Context standard

### Viewing End-to-End Transactions

1. Application Insights → **Transaction search**
2. Select a request
3. Click **View end-to-end transaction**
4. See complete request flow across services

## Application Map

Visualizes dependencies and relationships:

**Shows:**
- APIM gateway
- Backend services
- Database calls
- External APIs
- Failure rates per dependency
- Average durations

**Access:**
- Application Insights → **Application Map**

## Performance Analysis

### Performance Tab

Shows request performance distribution:
- Operations by volume
- Operations by duration
- Slow requests
- Failed requests

### Metrics Explorer

Create custom charts:
- Request rate
- Average response time
- Dependency duration
- Server exceptions
- Custom metrics

## Alerting

### Smart Detection

AI-powered anomaly detection:
- Unusual spike in failures
- Degradation in response time
- Dependency performance anomalies
- Memory leak detection
- Security issues

### Custom Alerts

Create alerts on KQL queries:

**Example: High Error Rate**
```kusto
requests
| where timestamp > ago(5m)
| summarize ErrorRate = (countif(success == false) * 100.0) / count()
| where ErrorRate > 5
```

Alert when error rate > 5% for 5 consecutive minutes.

## Custom Telemetry

### Adding Custom Events

Use policies to log custom events:

```xml
<log-to-eventhub logger-id="appinsights">
    @{
        return new JObject(
            new JProperty("EventTime", DateTime.UtcNow.ToString()),
            new JProperty("ServiceName", context.Api.Name),
            new JProperty("Operation", context.Operation.Name),
            new JProperty("ClientIP", context.Request.IpAddress),
            new JProperty("UserId", context.User?.Id ?? "anonymous"),
            new JProperty("Duration", context.Elapsed.TotalMilliseconds)
        ).ToString();
    }
</log-to-eventhub>
```

### Custom Dimensions

Add context to requests:

```xml
<set-variable name="apiKey" value="@(context.Subscription?.Name ?? "none")" />
<trace source="custom-trace" severity="information">
    <message>API call completed</message>
    <metadata name="subscription" value="@((string)context.Variables["apiKey"])" />
    <metadata name="product" value="@(context.Product?.Name ?? "none")" />
</trace>
```

## Workbooks

### Creating Custom Workbooks

Workbooks are interactive reports combining:
- KQL queries
- Visualizations
- Text and links
- Parameters for filtering

**Example Workbook: API Performance Dashboard**

Sections:
1. **Overview** - Request volume, error rate, avg duration
2. **Top APIs** - Most called APIs
3. **Slowest Operations** - Performance bottlenecks
4. **Error Analysis** - Failed requests by type
5. **Backend Health** - Dependency performance

### Pre-built Templates

Application Insights provides templates:
- Performance analysis
- Failure analysis
- Usage analysis
- Browser performance

## Cost Optimization

### Data Ingestion Costs

Application Insights charges per GB ingested:
- First 5 GB/month: Free
- Additional data: ~$2.30/GB (varies by region)

### Reducing Costs

1. **Sampling** - Use 10-20% sampling for high-volume APIs
2. **Filtering** - Don't log non-essential headers/bodies
3. **Data Capping** - Set daily ingestion limits
4. **Retention** - Reduce from 90 days to 30 days if acceptable
5. **Separate Resources** - Use different App Insights for dev/prod

### Cost Estimation

Calculate expected costs:
```
Request Volume: 10M requests/day
Average Request Size: 2 KB
Daily Data: 10M * 2KB = 20 GB/day
Monthly Data: 20 GB * 30 = 600 GB/month
Cost: (600 - 5 GB free) * $2.30 = ~$1,368/month

With 20% sampling:
Monthly Data: 600 GB * 0.2 = 120 GB
Cost: (120 - 5) * $2.30 = ~$265/month
```

## Data Retention

### Default Retention

- **90 days** for detailed telemetry
- **90 days** for aggregated metrics

### Extended Retention

- Configure longer retention (up to 730 days)
- Additional cost per GB per month
- Consider exporting to cheaper storage for archival

### Continuous Export

Export data for long-term storage:
- **Destination**: Blob Storage, Event Hub
- **Use Cases**: Compliance, long-term analysis, backup
- **Cost**: Storage costs only

## Best Practices

1. **Enable for all APIs** - Consistent telemetry across platform
2. **Use appropriate sampling** - Balance cost and visibility
3. **Always log errors** - Never sample out failures
4. **Mask sensitive data** - Don't log PII or secrets
5. **Use correlation IDs** - Enable distributed tracing
6. **Create alerts** - Proactive issue detection
7. **Build workbooks** - Operational dashboards
8. **Review regularly** - Weekly telemetry analysis
9. **Optimize costs** - Monitor ingestion and adjust sampling

## Troubleshooting

### No Data Appearing

**Check:**
1. Application Insights correctly configured in APIM
2. Instrumentation key is valid
3. Diagnostics enabled for APIs
4. Firewall allows outbound to Application Insights endpoints
5. Check Activity Log for errors

### Incomplete Traces

**Check:**
1. Correlation headers propagated to backend
2. Backend supports distributed tracing
3. Network connectivity between services

### High Costs

**Actions:**
1. Review data ingestion volume
2. Increase sampling rate
3. Reduce logged headers/body size
4. Filter out health check endpoints
5. Set daily cap (with caution)

## Next Steps

1. **Configure Application Insights** - Connect to APIM
2. **Set appropriate sampling** - Balance cost and detail
3. **Create alerts** - Critical issues
4. **Build workbooks** - Operational dashboards
5. **Train team** - KQL and telemetry analysis

## Additional Resources

- [Application Insights Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [APIM + App Insights](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights)
- [KQL Quick Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
