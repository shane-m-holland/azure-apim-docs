# Dynatrace Integration

## Overview

Dynatrace is a comprehensive observability platform that can be integrated with Azure API Management to extend monitoring beyond Azure-native tooling. This integration provides AI-powered insights, automatic root cause analysis, and unified observability across your entire stack.

## Why Integrate Dynatrace with APIM?

**Azure-native tools (Azure Monitor, Application Insights) provide:**
- Platform metrics and logs
- Basic alerting and dashboards
- Azure-specific integrations

**Dynatrace adds:**
- AI-powered anomaly detection and root cause analysis
- Automatic topology mapping
- End-user experience monitoring
- Business transaction tracking
- Unified observability across cloud and on-premises
- Advanced AI/ML-based insights

## Integration Options

Dynatrace doesn't support agent-based monitoring for APIM directly (as APIM is a PaaS service), but full API flow observability is possible using these approaches:

### 1. Event Hub Integration (Recommended)

**How It Works:**
1. APIM sends diagnostic logs to Azure Event Hub
2. Dynatrace ingests logs from Event Hub
3. Logs appear in Dynatrace for analysis

**Benefits:**
- Most robust approach
- Near real-time log forwarding
- Complete request/response details
- Minimal latency (seconds)

### 2. Application Insights Integration

**How It Works:**
Dynatrace can ingest Application Insights telemetry via Azure Monitor API.

**Limitations:**
- Additional API calls and potential rate limiting
- Some delay in data availability
- Requires both App Insights and Dynatrace ingestion costs

### 3. Backend Service Instrumentation with OneAgent

**How It Works:**
Deploy Dynatrace OneAgent on backend services called by APIM.

**Benefits:**
- Full backend service observability
- Automatic code-level insights
- Database and dependency tracking
- Request correlation

**Use Case:**
Combine with Event Hub integration for complete observability:
- Event Hub: APIM gateway metrics
- OneAgent: Backend service details
- Result: Full request flow from client → APIM → backend → database

## Cost Considerations

### Dynatrace Licensing

**Factors:**
- Host units (backend servers with OneAgent)
- Digital Experience Monitoring (DEM) units
- Log ingestion volume
- Custom metrics

### Azure Costs

**Event Hub:**
- Throughput units: $0.015/hour per unit
- Ingress: First 1 GB/day free, then $0.028/GB

**Typical Monthly Cost:**
- Event Hub: $10-50/month (Standard tier)
- Total additional Azure cost: Minimal

## Best Practices

1. **Start with Event Hub** - Most reliable APIM integration
2. **Add OneAgent to backends** - Complete observability
3. **Use correlation IDs** - Enable distributed tracing
4. **Leverage Davis AI** - Let AI do anomaly detection
5. **Create runbooks** - Document response to common alerts

## Next Steps

1. **Assess requirements** - Determine if Dynatrace needed
2. **Set up Event Hub** - Create and configure
3. **Configure APIM logging** - Send to Event Hub
4. **Enable Dynatrace integration** - Connect to Event Hub
5. **Deploy OneAgent** - On backend services
6. **Create dashboards** - APIM-specific views

## Additional Resources

- [Dynatrace Azure Integration Guide](https://www.dynatrace.com/support/help/shortlink/azure-monitor-integration)
- [Event Hub Documentation](https://learn.microsoft.com/en-us/azure/event-hubs/)
- [APIM Diagnostic Logging](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor)
