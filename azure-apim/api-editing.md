# Azure API Management (APIM)

## Editing an Existing API

1. Navigate to the Azure APIM instance
    * Go to [Your Azure Portal](https://portal.azure.com)
    * Select your APIM Instance
2. Find and Select the API you wish to edit
    * In the left menu under APIs click "APIs"
    * You'll see a list of APIs with options to group or filter by specific Tags 
    * Select the API you want to edit

    ![step2-a](../images/step2.png)
3. Edit API Settings
    * Click "Settings" in the API navigation panel to update the following:
        * Display Name
        * API URL suffix (e.g. ``` api/v1/products ```)
        * API versioning settings (path, query, or header-based)

    ![step3](../images/step3.png)
4. Edit Operations/Endpoints
    * From the API overview page, you'll see a list of operations (GET, POST, etc.)
    * Click on the operation you want to edit. Here you can change the following information:
        * Change the summary, description, and operation ID
        * Add or edit query parameters, path parameters, or headers
        * Modify request body schema (if applicable)
        * Define responses, status codes, and example payloads
        * Add representations (e.g., ``` application/xml``` )

    ![step4](../images/step4.png)
5. Edit Policies (XML-Based Rules)
    * In the operation view, go to the "Inbound processing" or "Outbound processing" tab
    * You can also click "Code editor" to edit the XML directly

    ![step5](../images/policy.png)

 ```xml
<inbound>
    <base />
    <set-header name="X-Custom-Header" exists-action="override">
        <value>hello</value>
    </set-header>
</inbound>
```

6. Test the API
    * Navigate to the "Test" tab at the operation or API level
    * Add any required headers (e.g. ``` Ocp-Apim-Subscription-Key ```)
    ![Step6-testing](../images/step6.png)
7. Save and Publish Changes
    * Once tested, click "Save" at the top of any modified screen
    * If you're using API revisions, don't forget to "Make Revision Current" when ready.
8. Update the Developer Portal
    * If you've added new operations or changed the OpenAPI spec:
        * Go to Developer Portal from the left menu
        * Click "refresh" to ensure updates are synced
        * You can further customize text or branding in the CMS editor.