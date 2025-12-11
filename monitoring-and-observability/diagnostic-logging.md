# Diagnostic Logging

## Overview

Diagnostic logging in Azure API Management captures detailed request and response information, enabling troubleshooting, audit trails, and usage analysis. Unlike platform metrics, diagnostic logs provide granular, request-level details.

## What Gets Logged

Diagnostic logs can capture:

### Request Information
- HTTP method (GET, POST, etc.)
- URL path and query parameters
- Request headers (configurable)
- Request body (configurable, with size limits)
- Client IP address
- User agent string
- Timestamp

### Response Information
- HTTP status code
- Response headers (configurable)
- Response body (configurable, with size limits)
- Total duration
- Backend duration

### Policy Execution
- Policies applied
- Policy execution order
- Policy failures or errors
- Transformations performed

### Authentication/Authorization
- Subscription key used (hashed)
- User identity
- Authorization result
- OAuth/JWT validation results

## Log Destinations

APIM supports three destinations for diagnostic logs:

### 1. Log Analytics Workspace

**Best For:** Querying, analysis, and alerting

**Capabilities:**
- Query logs with Kusto Query Language (KQL)
- Create custom dashboards and workbooks
- Set up log-based alerts
- Cross-resource queries
- Integration with Azure Sentinel

**Cost:** Per GB ingested + retention charges

**Configuration:**
```bicep
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'apim-diagnostics'
  scope: apimService
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      {
        category: 'GatewayLogs'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30
        }
      }
    ]
  }
}
```

### 2. Event Hubs

**Best For:** Real-time streaming to external systems

**Capabilities:**
- Stream logs to SIEM (Splunk, Dynatrace, etc.)
- Real-time processing pipelines
- Integration with Azure Stream Analytics
- Custom event processors

**Cost:** Per throughput unit

**Use Cases:**
- Streaming to Dynatrace for full-stack observability
- Feeding logs into Splunk or Elasticsearch
- Real-time anomaly detection systems
- Custom analytics pipelines

### 3. Azure Storage (Blob)

**Best For:** Long-term archival and compliance

**Capabilities:**
- Cost-effective long-term storage
- Compliance and audit requirements
- Archival for regulatory purposes
- Infrequent access patterns

**Cost:** Per GB stored per month (~$0.02/GB)

**Use Cases:**
- Compliance requirements (e.g., retain 7 years)
- Historical analysis
- Cold storage for completed investigations
- Backup and disaster recovery

## Configuration Levels

Diagnostic logging can be configured at multiple levels:

### Global (Service-Level)

Applies to all APIs in the APIM instance:

**Azure Portal:**
1. APIM instance → **Diagnostic settings**
2. Add diagnostic setting
3. Select log categories and destination
4. Configure retention and sampling

### API-Level

Specific configuration for individual APIs:

**Enables:**
- Different sampling rates per API
- More verbose logging for critical APIs
- Reduced logging for high-volume, low-value APIs

### Operation-Level

Fine-grained control per operation:

**Example:** Enable verbose logging only for POST operations

## Configuring Diagnostic Settings

### Portal Configuration

1. Navigate to APIM instance
2. Select **Diagnostic settings** under Monitoring
3. Click **Add diagnostic setting**
4. Configure:
   - **Name**: Descriptive name (e.g., `logs-to-workspace`)
   - **Logs**: Select categories
     - GatewayLogs
     - WebSocketConnectionLogs
     - DeveloperPortalAuditLogs
   - **Destination**: Choose one or more
     - Log Analytics workspace
     - Event Hub
     - Storage Account
5. Set **Retention** (days)
6. Save

### Bicep Configuration

```bicep
resource diagnostics 'Microsoft.ApiManagement/service/diagnostics@2023-03-01-preview' = {
  parent: apimService
  name: 'azuremonitor'
  properties: {
    alwaysLog: 'allErrors'
    loggerId: logAnalyticsLogger.id
    sampling: {
      samplingType: 'fixed'
      percentage: 20
    }
    verbosity: 'information'
    logClientIp: true
    httpCorrelationProtocol: 'W3C'
    frontend: {
      request: {
        headers: ['Content-Type', 'User-Agent', 'Authorization']
        body: {
          bytes: 1024
        }
        dataMasking: {
          queryParams: [
            {
              value: 'api-key'
              mode: 'Hide'
            }
          ]
        }
      }
      response: {
        headers: ['Content-Type', 'Cache-Control']
        body: {
          bytes: 1024
        }
      }
    }
    backend: {
      request: {
        headers: ['Content-Type']
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
  }
}
```

## Sampling Strategies

Sampling reduces log volume and costs while maintaining statistical validity.

### Fixed Sampling

**Configuration:**
```xml
<diagnostics>
  <sampling>
    <percentage>20</percentage>
  </sampling>
</diagnostics>
```

Logs exactly 20% of requests.

### Conditional Sampling

Log based on conditions:

**Example: Always log errors, sample successes**
```xml
<diagnostics>
  <always-log>allErrors</always-log>
  <sampling>
    <percentage>10</percentage>
  </sampling>
</diagnostics>
```

### Sampling by API

Different rates for different APIs:

```bicep
// High-value API: 100% sampling
resource criticalApiDiagnostics 'Microsoft.ApiManagement/service/apis/diagnostics@2023-03-01-preview' = {
  parent: criticalApi
  name: 'azuremonitor'
  properties: {
    sampling: {
      percentage: 100
    }
  }
}

// High-volume API: 5% sampling
resource highVolumeApiDiagnostics 'Microsoft.ApiManagement/service/apis/diagnostics@2023-03-01-preview' = {
  parent: highVolumeApi
  name: 'azuremonitor'
  properties: {
    sampling: {
      percentage: 5
    }
  }
}
```

## Data Masking

Protect sensitive information in logs:

### Query Parameter Masking

```bicep
dataMasking: {
  queryParams: [
    {
      value: 'password'
      mode: 'Hide'  // Completely hide
    }
    {
      value: 'api-key'
      mode: 'Mask'  // Show as '***'
    }
  ]
}
```

### Header Masking

```bicep
dataMasking: {
  headers: [
    {
      value: 'Authorization'
      mode: 'Hide'
    }
    {
      value: 'X-API-Key'
      mode: 'Mask'
    }
  ]
}
```

### Best Practices

**Always mask:**
- Passwords and secrets
- API keys and tokens
- Authorization headers (or hash them)
- Credit card numbers
- Social security numbers
- Any PII or sensitive data

## Querying Logs

### Log Analytics (KQL)

**Recent Gateway Logs:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, Method, Url, ResponseCode, TotalTime, BackendTime
| order by TimeGenerated desc
| take 100
```

**Error Analysis:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(24h) and ResponseCode >= 400
| summarize ErrorCount = count() by ResponseCode, Url
| order by ErrorCount desc
```

**Slow Requests:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| where TotalTime > 1000  // > 1 second
| project TimeGenerated, Method, Url, TotalTime, BackendTime
| order by TotalTime desc
```

**Backend Performance:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| summarize 
    AvgBackendTime = avg(BackendTime),
    P95BackendTime = percentile(BackendTime, 95)
    by BackendUrl
| order by P95BackendTime desc
```

**Traffic by API:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(24h)
| summarize RequestCount = count() by ApiId
| order by RequestCount desc
```

## Log Categories

### GatewayLogs

Main category capturing API request/response data:
- All gateway traffic
- Request and response details
- Policy execution
- Backend calls

### WebSocketConnectionLogs

WebSocket-specific logs:
- Connection establishment
- Message flow
- Connection termination
- Errors and failures

### DeveloperPortalAuditLogs

Developer portal activity:
- User sign-ups
- Subscription requests
- API testing
- Portal customization changes

## Performance Impact

Logging has minimal performance impact when properly configured:

### Best Practices

1. **Use sampling** - Don't log 100% in production
2. **Limit body sizes** - Log only first 512-1024 bytes
3. **Select headers carefully** - Don't log all headers
4. **Avoid verbose in production** - Use information or warning level
5. **Buffer logs** - Use buffering to batch log writes

### Monitoring Logging Performance

Check these metrics:
- Request latency (shouldn't increase significantly)
- Gateway capacity utilization
- Log ingestion lag

## Cost Management

Diagnostic logging can be expensive. Control costs:

### Ingestion Costs

**Log Analytics:**
- First 5 GB/month: Free
- Additional: ~$2.30/GB

**Example Calculation:**
```
API Volume: 10M requests/day
Avg Log Size: 3 KB per request
Daily Data: 10M * 3KB = 30 GB/day
Monthly: 30 GB * 30 = 900 GB/month
Cost: (900 - 5) * $2.30 = ~$2,058/month

With 20% sampling:
Monthly: 900 * 0.2 = 180 GB
Cost: (180 - 5) * $2.30 = ~$403/month
```

### Retention Costs

Retention beyond 30 days incurs additional charges:
- 31-90 days: Included in ingestion cost
- 91-730 days: Additional per GB per day charge

### Cost Optimization

1. **Sampling**: Use 10-20% for high-volume APIs
2. **Retention**: Don't retain longer than necessary
3. **Archival**: Move old logs to cheap blob storage
4. **Filtering**: Exclude health checks and monitoring probes
5. **Selective logging**: Enable only for critical APIs

## Compliance and Audit

### Regulatory Requirements

Many industries require log retention:
- **Healthcare (HIPAA)**: 6 years
- **Finance (SOX)**: 7 years
- **PCI DSS**: 1 year (3 months online)

### Audit Trail

Logs serve as audit trails for:
- API access patterns
- Failed authentication attempts
- Administrative changes
- Data access by users

### Immutable Storage

For compliance, use:
- **Storage Account with immutability policies**
- **Write-once-read-many (WORM) storage**
- **Legal hold on logs**

## Troubleshooting with Logs

### Scenario: Slow API Performance

**Investigation:**
```kusto
ApiManagementGatewayLogs
| where ApiId == "my-api" and TimeGenerated > ago(1h)
| summarize 
    AvgTotal = avg(TotalTime),
    AvgBackend = avg(BackendTime),
    AvgApim = avg(TotalTime - BackendTime)
    by bin(TimeGenerated, 5m)
| render timechart
```

### Scenario: Increased Error Rate

**Investigation:**
```kusto
ApiManagementGatewayLogs
| where ResponseCode >= 400 and TimeGenerated > ago(1h)
| project TimeGenerated, Method, Url, ResponseCode, LastError
| order by TimeGenerated desc
```

### Scenario: Backend Service Issues

**Investigation:**
```kusto
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| summarize 
    TotalCalls = count(),
    Failures = countif(BackendResponseCode >= 500)
    by BackendUrl
| extend FailureRate = (Failures * 100.0) / TotalCalls
| where FailureRate > 0
| order by FailureRate desc
```

## Best Practices

1. **Enable early** - Set up logging from the start
2. **Always log errors** - Never sample out failures
3. **Use sampling** - Reduce costs for successful requests
4. **Mask sensitive data** - Never log PII or secrets
5. **Choose appropriate retention** - Balance compliance and cost
6. **Create alerts** - Proactive issue detection
7. **Review regularly** - Analyze logs weekly for trends
8. **Document queries** - Save common KQL queries for team
9. **Test log queries** - Ensure queries work before incidents
10. **Archive old logs** - Move to cheap storage for compliance

## Next Steps

1. **Configure diagnostic settings** - Set up log destinations
2. **Enable sampling** - Balance detail and cost
3. **Mask sensitive data** - Protect PII and secrets
4. **Create alerts** - Log-based alerting for critical issues
5. **Build queries** - Common troubleshooting KQL queries
6. **Test and validate** - Ensure logs contain expected data

## Additional Resources

- [Diagnostic Logs Schema](https://learn.microsoft.com/en-us/azure/api-management/gateway-log-schema-reference)
- [KQL Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [APIM Diagnostics](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
