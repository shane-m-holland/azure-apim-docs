# Azure API Management (APIM) 

## Revisions

### Why It Matters

* Revisions = Draft updates to an existing version *before* publishing.

### When to Use Revisions

* Making __non__-breaking changes
* You want to test updates before making them *live*
* You want the ability to roll back if needed

### How to Create a Revision

1. In the API list, select your API version (e.g., ```Products API v1```)
2. Click the "Revisions" tab (top bar)
3. Click "+ Add Revision"
4. Enter a name and description (e.g., ```rev2 - added query param```)
5. Click "Save"
6. You should now see a yellow banner warning that:  ```You are viewing a non-current revision.```
7. Now you can make your changes
    * Add new operations
    * Update request/response definitions
    * Modify policies
    * Test the changes safely

### Taking a revision *Live*

* Click the "Revisions" tab again
* Select the revision you wish to "push" out to the public
* Click "Make Current"

The changes are now *live*, and the previous revision is still recoverable!

### Rolling Back a Revision in Azure APIM Beginning to End

> Keep in mind:
> * __Revisions__ are _drafts_ of an existing API
> * Only __one revision__ is marked as "Current" at a time (i.e. the live version).
> * Rolling back is making an _older_ revision current again.

1. Open Your APIM Instance
    * Go to your [Azure Portal](https://portal.azure.com)
    * Navigate to __API Management Services__
    * Select your APIM instance
2. Locate the __Target__ API
    * From the left menu, click APIs
    * Find and click on the API version you want to roll back
3. Open the Revisions Tab
    * In the top menu, click "Revisions"
    * You'll see a list of _all_ revisions (e.g., ```rev1```, ```rev2```, etc.)
    * ```Current``` = live version
    * Others = available for rollback.
4. Select the Older Revision
    * Find the revision you want to restore (e.g., ```rev1```)
    * Click the revision row to open it
    * Review the contents if needed
5. Make it current
    * At the top, click "Make current" and confirm the action in the popup window.
6. The revision is now live!
    * The selected revision becomes active.
    * Any dependent clients will use this new "Current" revision immediately.

### Tips

* Always test a revision in the Test tab before making it current to avoid unexpected behavior.
* Use clear descriptions when creating revisions to make rollbacks easier in the future.
* Avoid editing the live __(current)__ revision directly, use revisions to reduce the risk of interrupted service and unexpected behavior.
