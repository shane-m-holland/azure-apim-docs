# Azure Monitor Integration

## Overview

Azure Monitor is the platform-level monitoring service in Azure that collects metrics and activity logs for all Azure resources, including API Management. It provides the foundation for monitoring APIM health, performance, and capacity.

## Key Capabilities

- **Platform Metrics** - Automatically collected without configuration
- **Alerting** - Threshold and metric-based alerts
- **Dashboards** - Visual representations of metrics
- **Insights** - Pre-built views for common scenarios
- **Activity Logs** - Administrative operations on APIM resources

## Automatically Collected Metrics

Azure Monitor automatically collects these metrics for APIM (no configuration required):

### Request Metrics

- **Total Gateway Requests** - Count of all requests to the gateway
- **Successful Gateway Requests** - Requests with 2xx or 3xx responses
- **Unauthorized Gateway Requests** - 401 responses
- **Failed Gateway Requests** - 4xx and 5xx responses
- **Other Gateway Requests** - Requests that don't fall into above categories

### Capacity Metrics

- **Capacity** - Percentage of APIM capacity utilized
- **Duration** - Request processing time
- **Backend Duration** - Time spent in backend calls
- **Other Duration** - Time spent in APIM gateway processing

### Network Metrics

- **Network Connectivity Status** - Health of network connections
- **EventHub Successful Events** - Events successfully sent to Event Hub
- **EventHub Failed Events** - Failed Event Hub deliveries

## Viewing Metrics

### Azure Portal

1. Navigate to your APIM instance in Azure Portal
2. Select **Metrics** from the left menu
3. Choose metric(s) to visualize
4. Apply filters (time range, aggregation, splitting)
5. Save to dashboard or pin to overview

### Common Metric Queries

**Total Request Rate:**
```
Metric: Total Gateway Requests
Aggregation: Sum
Time Range: Last 24 hours
Chart Type: Line
```

**Error Rate Percentage:**
```
Metric: Failed Gateway Requests / Total Gateway Requests * 100
Aggregation: Average
Time Range: Last hour
Chart Type: Line
```

**Capacity Utilization:**
```
Metric: Capacity
Aggregation: Average
Time Range: Last 24 hours
Chart Type: Line
```

## Alerting

### Creating Alert Rules

**Via Azure Portal:**

1. Navigate to APIM instance
2. Select **Alerts** → **Create alert rule**
3. Define condition (metric, threshold, aggregation)
4. Configure action group (email, SMS, webhook)
5. Set alert rule name and severity
6. Create the rule

**Example Alert Rules:**

**High Error Rate:**
```
Condition: Failed Gateway Requests > 100 per 5 minutes
Severity: 2 (Warning)
Action: Email operations team
```

**Capacity Threshold:**
```
Condition: Capacity > 80% for 10 minutes
Severity: 1 (Error)
Action: Page on-call engineer
```

**Backend Performance:**
```
Condition: Backend Duration > 2000ms (p95)
Severity: 3 (Informational)
Action: Log to ticket system
```

### Alert Best Practices

1. **Set appropriate thresholds** - Base on baseline performance, not arbitrary numbers
2. **Use multi-dimensional alerts** - Alert per API or product
3. **Implement escalation** - Increase severity for persistent issues
4. **Avoid alert fatigue** - Too many alerts leads to ignoring them
5. **Test alerts** - Verify notifications reach the right people

## Dashboards

### Creating Custom Dashboards

**Portal Dashboard:**

1. Go to Azure Portal → **Dashboard**
2. Select **New dashboard** → **Blank dashboard**
3. Add tiles:
   - Metrics charts from APIM
   - Resource health
   - Recent alerts
   - Activity log
4. Arrange and resize tiles
5. Save and share dashboard

**Example Dashboard Layout:**

```
┌─────────────────────────────────────────────────┐
│  APIM Gateway Health                             │
├─────────────────┬───────────────┬───────────────┤
│  Request Rate   │  Error Rate   │  Capacity     │
│  (Line Chart)   │  (Line Chart) │  (Gauge)      │
├─────────────────┴───────────────┴───────────────┤
│  Response Time Distribution (Bar Chart)          │
├─────────────────────────────────────────────────┤
│  Recent Alerts        │  Backend Health          │
│  (List)               │  (Line Chart)            │
└───────────────────────┴─────────────────────────┘
```

### Pre-built Insights

Azure Monitor provides built-in insights for APIM:

- **Resource Health** - Overall availability status
- **Metrics Explorer** - Interactive metric analysis
- **Activity Log** - Recent administrative operations
- **Diagnostics** - Recommended alerts and configurations

## Action Groups

Action groups define what happens when an alert fires.

### Action Types

- **Email/SMS** - Notify individuals
- **Azure Function** - Custom automation
- **Logic App** - Complex workflows
- **Webhook** - External integrations
- **ITSM** - ServiceNow, Jira tickets
- **Automation Runbook** - Automated remediation

### Example Action Group

**Name:** `apim-critical-alerts`

**Actions:**
- Email: operations@company.com
- SMS: +1-555-0100 (On-call phone)
- Webhook: https://company.webhook.site/apim-alert
- Logic App: Create incident in ServiceNow

## Activity Logs

Activity logs track administrative operations on APIM:

### Common Activities

- **Write** - Configuration changes (create/update/delete)
- **Action** - Operations like publishing APIs, regenerating keys
- **Delete** - Resource deletions

### Viewing Activity Logs

1. Navigate to APIM instance
2. Select **Activity log**
3. Filter by:
   - Time range
   - Event severity (Critical, Error, Warning, Info)
   - Operation name
   - Initiated by (user/service principal)

### Activity Log Retention

- Default retention: **90 days**
- For longer retention: Export to Log Analytics or Storage

### Example Queries

**Recent Configuration Changes:**
```
Filter: Operation Name contains "write"
Time Range: Last 7 days
```

**Failed Operations:**
```
Filter: Event severity = Error
Time Range: Last 24 hours
```

## Integration with Other Services

### Log Analytics

Send metrics to Log Analytics for:
- Long-term storage
- Cross-resource queries
- Advanced analysis with KQL

**Configuration:**
1. APIM → Diagnostic settings
2. Add diagnostic setting
3. Select "Send to Log Analytics"
4. Choose workspace
5. Select categories (metrics, logs, or both)

### Azure Automation

Use metrics to trigger automation:

**Example: Auto-scale based on capacity**
```
IF Capacity > 80% for 15 minutes
THEN Execute runbook to add APIM units
```

## Capacity Planning

### Understanding Capacity Metric

The Capacity metric is a calculated value representing:
- CPU usage
- Memory consumption
- Network bandwidth
- Request queue length

**Interpretation:**
- **0-50%** - Normal operation
- **50-70%** - Monitor closely
- **70-85%** - Plan to scale
- **85%+** - Scale immediately

### Scaling Decisions

Monitor these metrics to inform scaling:

1. **Sustained high capacity** - Add units or upgrade tier
2. **Periodic spikes** - Consider autoscaling (Premium tier only)
3. **Low capacity** - Potential to downscale and reduce costs

## Multi-Region Monitoring

For Premium tier with multi-region deployment:

### Regional Metrics

Each region provides separate metrics:
- Filter by **Location** dimension
- Compare regional performance
- Identify regional issues

### Global View

Create dashboards showing:
- Aggregate metrics across regions
- Per-region breakdown
- Traffic distribution
- Regional health status

## Cost Considerations

Azure Monitor metrics are:
- **Free** up to platform metrics limits
- **No charge** for standard metric collection
- **Storage** incurred when exporting to Log Analytics or Storage

Alert rules incur minimal costs:
- First 5 alerts free
- $0.10 per additional alert evaluation per month

## Limitations

- **Metric retention** - 93 days maximum in Azure Monitor
- **Granularity** - 1 minute minimum for platform metrics
- **Dimensions** - Limited dimensional filtering
- **Historical analysis** - Export to Log Analytics for longer retention

## Troubleshooting with Metrics

### High Error Rate

**Check:**
1. Failed Gateway Requests metric - identify spike
2. Split by dimension (API, Operation) - isolate problematic API
3. Review Activity Log - recent configuration changes?
4. Check backend metrics - backend service issues?

### Performance Degradation

**Check:**
1. Duration metric - overall request time
2. Backend Duration - time spent in backend
3. Capacity metric - gateway overloaded?
4. Network metrics - connectivity issues?

### Capacity Issues

**Check:**
1. Capacity metric - current utilization
2. Request volume - unusual spike?
3. Backend Duration - slow backends causing queue buildup?
4. Cache hit ratio - caching working effectively?

## Best Practices

1. **Monitor proactively** - Set up alerts before issues occur
2. **Establish baselines** - Understand normal operating metrics
3. **Use dimensions** - Filter metrics by API, product, location
4. **Create dashboards** - Quick visibility for operations team
5. **Review regularly** - Weekly metric reviews to identify trends
6. **Document thresholds** - Record why alert thresholds were set
7. **Test alerts** - Ensure notifications reach intended recipients

## Next Steps

1. **Set up basic alerts** - Start with capacity and error rate
2. **Create operational dashboard** - Visibility for team
3. **Add Application Insights** - For deeper telemetry
4. **Export to Log Analytics** - For advanced analysis
5. **Implement autoscaling** - If using Premium tier

## Additional Resources

- [Azure Monitor Metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-platform-metrics)
- [APIM Monitoring](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
- [Azure Monitor Alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
