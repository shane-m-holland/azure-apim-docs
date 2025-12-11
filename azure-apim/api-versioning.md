# Azure API Management (APIM)

## Versioning

### Why It Matters

* Versions = Different public releases of your API (e.g., v1, v2, etc.)

### When to Use Versioning

* You want to release breaking changes
* You want to let clients choose between v1, v2, etc.

### How to Add Versioning

1. Go to your APIM instance in the [Azure Portal](https://portal.azure.com)
2. Select "APIs" from the left menu
3. On the specific API you wish to create a new version for, click the three dot menu.
4. Click "+ Add version" on your existing API

![Add Version](../images/add_version.png)

5. Decide on a Versioning Scheme:
    * Path (most common practice) -> ``` https://api.com/v1/products ```
    * Query string -> ``` https://api.com/products?api-version=1.0 ```
    * Header -> Requires clients to send ``` api-version: 1.0 ``` header
6. Enter a version identifier (e.g., v2) 
7. Click "Create"

#### Why did we do this?

* You now have Separate APIs in the portal (e.g., ```Products API v1``` , ``` Products API v2 ```)
* Each version has its __own operations__, __policies__, and __documentation__
* Which means you can modify them independently!
