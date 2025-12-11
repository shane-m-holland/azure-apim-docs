# Cost Optimization for Monitoring

## Overview

Monitoring and observability are essential, but costs can escalate quickly without proper management. This guide provides strategies to optimize monitoring costs while maintaining visibility into APIM performance and health.

## Cost Drivers

### Data Ingestion

**Application Insights:**
- First 5 GB/month: Free
- Additional data: ~$2.30/GB (varies by region)

**Log Analytics:**
- First 5 GB/month: Free  
- Additional data: ~$2.76/GB

**Event Hub:**
- Throughput units: $0.015/hour per unit
- Data ingress: $0.028/GB (after 1 GB/day free)

### Data Retention

**Application Insights / Log Analytics:**
- 31-90 days: Included in ingestion cost
- 91-730 days: Additional $0.12/GB per month

**Blob Storage:**
- Hot tier: $0.0184/GB per month
- Cool tier: $0.01/GB per month
- Archive tier: $0.002/GB per month

### Metrics and Alerts

**Azure Monitor:**
- Platform metrics: Free
- Custom metrics: $0.10 per tracked time series
- Alert evaluations: First 5 free, $0.10 per additional per month

## Cost Estimation

### Example Scenarios

**Scenario 1: Small Deployment (1M requests/day)**

```
Request Volume: 1M requests/day
Log Size: 2 KB per request
Sampling: 20%

Daily Data: 1M * 2KB * 0.2 = 400 MB/day
Monthly Data: 400 MB * 30 = 12 GB/month

Application Insights Cost:
(12 GB - 5 GB free) * $2.30 = $16.10/month

Log Analytics (if also enabled):
(12 GB - 5 GB free) * $2.76 = $19.32/month

Total: ~$35/month
```

**Scenario 2: Medium Deployment (10M requests/day)**

```
Request Volume: 10M requests/day
Log Size: 3 KB per request
Sampling: 10%

Daily Data: 10M * 3KB * 0.1 = 3 GB/day
Monthly Data: 3 GB * 30 = 90 GB/month

Application Insights:
(90 - 5) * $2.30 = $195.50/month

Total: ~$200/month
```

**Scenario 3: Large Deployment (100M requests/day)**

```
Request Volume: 100M requests/day
Log Size: 3 KB per request
Sampling: 5%

Daily Data: 100M * 3KB * 0.05 = 15 GB/day
Monthly Data: 15 GB * 30 = 450 GB/month

Application Insights:
(450 - 5) * $2.30 = $1,023.50/month

With optimization (2% sampling):
Monthly Data: 180 GB
Cost: (180 - 5) * $2.30 = $402.50/month

Savings: $621/month (61%)
```

## Optimization Strategies

### 1. Sampling

**Impact:** Reduces data volume by collecting only a percentage of telemetry

**Implementation:**

```bicep
diagnostics: {
  sampling: {
    samplingType: 'fixed'
    percentage: 20  // Log 20% of requests
  }
  alwaysLog: 'allErrors'  // Always log failures
}
```

**Recommendations:**
- Development: 100%
- Staging: 50%
- Production (low volume): 20-50%
- Production (high volume): 5-20%
- Production (very high volume): 1-5%

**Important:** Always enable `alwaysLog: 'allErrors'` to capture all failures regardless of sampling.

### 2. Selective Logging

**Impact:** Log only what's necessary

**Exclude from logging:**
- Health check endpoints
- Monitoring probes
- Internal test traffic
- Static content requests

**Implementation:**

```xml
<choose>
  <when condition="@(context.Request.Url.Path.Contains("/health"))">
    <!-- Skip logging for health checks -->
  </when>
  <otherwise>
    <log-to-eventhub>@{ /* log request */ }</log-to-eventhub>
  </otherwise>
</choose>
```

### 3. Reduce Log Size

**Impact:** Smaller logs = lower ingestion costs

**Strategies:**

```bicep
frontend: {
  request: {
    headers: ['Content-Type', 'User-Agent']  // Only essential headers
    body: {
      bytes: 512  // Limit body size
    }
  }
  response: {
    headers: ['Content-Type']
    body: {
      bytes: 512
    }
  }
}
```

**Optimizations:**
- Log only essential headers (not all)
- Limit request/response body bytes
- Don't log binary content
- Compress logs before sending

### 4. Retention Management

**Impact:** Longer retention = higher costs

**Strategies:**
- **30 days**: For most operational logs
- **90 days**: For important APIs
- **1 year+**: Only for compliance requirements

**Implementation:**

```bicep
resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  properties: {
    logs: [
      {
        category: 'GatewayLogs'
        enabled: true
        retentionPolicy: {
          enabled: true
          days: 30  // Retain for 30 days only
        }
      }
    ]
  }
}
```

### 5. Archive Old Data

**Impact:** Move infrequently accessed data to cheaper storage

**Process:**
1. Export logs older than 90 days to Blob Storage
2. Use Cool or Archive tier for long-term retention
3. Query from blob storage when needed (slower but much cheaper)

**Cost Comparison:**
- Log Analytics (90+ days): $0.12/GB/month
- Blob Cool tier: $0.01/GB/month  
- Blob Archive tier: $0.002/GB/month

**Savings:** 90-98% for long-term retention

### 6. Separate Resources

**Impact:** Better cost allocation and management

**Strategy:**
Create separate Application Insights/Log Analytics per:
- Environment (dev, staging, prod)
- Team or business unit
- Cost center

**Benefits:**
- Accurate cost attribution
- Independent retention policies
- Easier to optimize per environment
- Clear billing

### 7. Data Capping

**Impact:** Hard limit on daily ingestion to prevent runaway costs

**Caution:** Can result in lost data if cap is reached

**Implementation:**

1. Application Insights → Usage and estimated costs
2. Set daily cap (e.g., 10 GB/day)
3. Configure alerts when approaching cap

**Recommendation:** Use as safety net, not primary cost control.

### 8. Optimize Queries

**Impact:** Faster queries = lower compute costs

**Best Practices:**
- Use time range filters
- Limit result sets with `take` or `top`
- Use `summarize` instead of returning raw logs
- Cache frequently used queries
- Avoid `select *`, specify columns

**Example:**

```kusto
// Bad: Returns all fields and rows
ApiManagementGatewayLogs
| where TimeGenerated > ago(24h)

// Good: Returns only needed fields, limited rows
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| project TimeGenerated, Method, Url, ResponseCode
| take 100
```

### 9. Monitor Monitoring Costs

**Impact:** Visibility into what's costing money

**Actions:**
1. Enable cost analysis for monitoring resources
2. Set budget alerts
3. Review monthly cost trends
4. Identify top data contributors

**Azure Cost Management:**
- View costs by resource
- Filter by service (Application Insights, Log Analytics)
- Export cost data for analysis

### 10. Use Platform Metrics Where Possible

**Impact:** Platform metrics are free (within limits)

**Strategy:**
- Use Azure Monitor metrics for high-level monitoring
- Reserve Application Insights for detailed troubleshooting
- Don't duplicate metrics in both systems

## Cost Monitoring Checklist

- [ ] Set monthly budget for monitoring
- [ ] Configure budget alerts (80%, 100%, 120%)
- [ ] Review data ingestion weekly
- [ ] Analyze top log contributors monthly
- [ ] Adjust sampling based on actual usage
- [ ] Archive logs older than retention period
- [ ] Remove unused Log Analytics workspaces
- [ ] Consolidate Application Insights resources
- [ ] Review and optimize expensive queries
- [ ] Educate team on cost-effective monitoring

## Common Cost Pitfalls

### Pitfall 1: No Sampling in Production

**Problem:** Logging 100% of high-volume API traffic

**Impact:** 5-20x higher costs than necessary

**Solution:** Enable sampling (10-20% for most scenarios)

### Pitfall 2: Logging All Headers and Bodies

**Problem:** Capturing full request/response data

**Impact:** Large log sizes, expensive ingestion

**Solution:** Log only essential headers, limit body sizes

### Pitfall 3: Excessive Retention

**Problem:** Keeping all logs for years in Log Analytics

**Impact:** High retention costs

**Solution:** Move old logs to blob storage, keep only 30-90 days in Log Analytics

### Pitfall 4: Duplicate Logging

**Problem:** Sending same data to multiple destinations

**Impact:** Paying multiple times for same data

**Solution:** Use one primary destination, export to others only when necessary

### Pitfall 5: No Cost Monitoring

**Problem:** Not tracking monitoring costs until bill arrives

**Impact:** Unexpected expenses, no proactive optimization

**Solution:** Set up Azure Cost Management alerts and reviews

## ROI of Monitoring

While optimization is important, remember the value monitoring provides:

**Cost of Downtime:**
- 1 hour outage of critical API could cost $10,000-$1,000,000+
- Monitoring cost: $200-$1,000/month

**Benefits:**
- Faster incident detection and resolution
- Proactive issue identification
- Better capacity planning
- Improved customer experience
- Reduced MTTR (Mean Time To Recover)

**ROI Calculation:**
```
Annual Monitoring Cost: $5,000
Prevented Incidents: 3
Cost per incident: $50,000
Total Prevented Cost: $150,000
ROI: ($150,000 - $5,000) / $5,000 = 2,900%
```

## Balancing Cost and Visibility

### Critical APIs (100% Visibility)

- High-value business APIs
- Customer-facing endpoints
- Payment processing
- Authentication services

**Approach:**
- Higher sampling (50-100%)
- Longer retention (90 days)
- Multiple monitoring tools
- Real-time alerting

### Standard APIs (Good Visibility)

- Internal APIs
- Medium-volume endpoints
- Standard CRUD operations

**Approach:**
- Moderate sampling (20-50%)
- Standard retention (30 days)
- Azure-native monitoring
- Standard alerting

### Low-Priority APIs (Basic Visibility)

- Test endpoints
- Internal tooling
- Low-volume APIs

**Approach:**
- Low sampling (5-10%)
- Minimal retention (7-14 days)
- Platform metrics only
- Aggregate alerting

## Best Practices Summary

1. **Enable sampling** - Start with 20% and adjust
2. **Always log errors** - Never sample out failures
3. **Limit log size** - Only essential headers and limited body
4. **Set appropriate retention** - 30 days for most logs
5. **Archive old data** - Move to blob storage after 90 days
6. **Separate resources** - Per environment and team
7. **Monitor costs** - Weekly reviews and budget alerts
8. **Use platform metrics** - Free and sufficient for many scenarios
9. **Optimize queries** - Fast queries, lower costs
10. **Balance cost and value** - Spend more on critical APIs

## Next Steps

1. **Calculate current costs** - Baseline monitoring expenses
2. **Set budget** - Define acceptable monthly spend
3. **Implement sampling** - Start with 20%
4. **Configure retention** - 30 days for standard logs
5. **Set up cost alerts** - At 80% and 100% of budget
6. **Review monthly** - Analyze costs and optimize
7. **Train team** - Cost-aware monitoring practices

## Additional Resources

- [Azure Monitor Pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/)
- [Application Insights Pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/)
- [Azure Cost Management](https://learn.microsoft.com/en-us/azure/cost-management-billing/)
