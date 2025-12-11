# Add Application Insights to the Managed Developer Portal (Standard v2 & Classic)

## What you'll get

-   Automatic **page view** and **performance** telemetry for the portal
    (first paint, load, client timings).
-   Request/exception telemetry for the portal app itself
    (client-side).\
-   Data lands in **Azure Monitor / Application Insights** where you can
    query with KQL and build workbooks/alerts.

## What you won't get in v2

-   No custom HTML/widgets = you can't inject your own JS to track extra
    events inside the **managed** portal. For advanced tracking (custom
    events, click analytics plugin, setting user IDs), you'd need to
    **self-host** the portal or use custom widgets---both **not
    supported in v2**.

------------------------------------------------------------------------

## Step-by-step

### 0) Create (or pick) an App Insights resource

You can do this in the Portal, CLI, Bicep, etc. Grab the
**Instrumentation Key** (or Connection String; the portal integration
currently uses the key in the example).

### 1) Add App Insights to the portal configuration

Use the APIM **Content Item** REST API to update the portal's
**configuration** document with your **instrumentationKey**, then
**republish** the portal.

-   **GET** current config (for reference):

```{=html}
<!-- -->
```
    GET https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/contentTypes/document/contentItems/configuration?api-version=2021-08-01

-   **PUT** updated config (add the `integration.appInsights` node):

``` json
{
  "id": "/contentTypes/document/contentItems/configuration",
  "type": "Microsoft.ApiManagement/service/contentTypes/contentItems",
  "name": "configuration",
  "properties": {
    "nodes": [
      {
        "site": { "title": "Your Portal", "description": "..." },
        "integration": {
          "appInsights": {
            "instrumentationKey": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
          }
        }
      }
    ]
  }
}
```

-   **Republish** the portal (Portal → APIM → Developer portal →
    **Publish**).

------------------------------------------------------------------------

## Verify & Visualize

### Quick KQL starters (Application Insights → Logs)

**Page views over time**

``` kusto
pageViews
| where timestamp > ago(24h)
| summarize count() by bin(timestamp, 5m), pageTitle
| order by timestamp asc
```

**Top portal pages**

``` kusto
pageViews
| where timestamp > ago(7d)
| summarize Views=count() by pageTitle, pageUrl
| top 50 by Views desc
```

**Portal performance (client)**

``` kusto
pageViews
| where timestamp > ago(24h)
| summarize avg(duration), p95=percentile(duration, 95) by pageTitle
| order by p95 desc
```

**User engagement by session**

``` kusto
pageViews
| where timestamp > ago(7d)
| summarize Views=count() by session_Id
| summarize Sessions=count(), AvgViews=avg(Views)
```

> Tip: Build a **Workbook** for portal RUM (page load, availability, top
> pages, geo) and pin to a dashboard.

------------------------------------------------------------------------

## Security & Ops Considerations

-   **Key/connection string is client-visible.** Browser RUM requires a
    public key/connection string; plan for that (separate App Insights
    resource for portal telemetry, locked retention, sampling).
-   If your org requires **AAD-auth'ed ingestion only**, you can front
    App Insights ingestion with **APIM as a secure proxy** (managed
    identity) and point the RUM SDK to that endpoint. This is advanced,
    but Microsoft documented the pattern recently.
-   **Sampling**: Configure adaptive sampling (e.g., 5--20%) to control
    cost while preserving shape.
-   **Retention**: Set short retention (e.g., 30--90 days) for portal
    analytics unless compliance needs more.
-   **Alerting**: Add alerts on **pageViews** drop, **exceptions**
    spike, or **p95 duration** regression.

------------------------------------------------------------------------

## Limits in v2 & Ways Around Them

-   **No custom widgets / HTML** in managed portal (v2), so
    **click-level** events, identity correlation (e.g., userId), and
    custom dimensions via JS aren't directly supported in the
    **managed** experience. For richer telemetry:
    -   **Self-host** the portal or use **custom widgets** (available in
        classic tiers, not v2), then add the full **JS SDK** and plugins
        (e.g., **Click Analytics**).
    -   Alternatively, capture **API-level usage** (the traffic that
        really matters) with APIM's **server-side** App Insights
        integration---fully supported on v2---for requests, failures,
        latency, and backend dependencies.

