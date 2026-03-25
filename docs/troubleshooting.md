# Troubleshooting

Common issues and their solutions when working with the Piwik PRO Optimizely Connector.

## "PiwikPRO is not configured"

The dashboard or widget displays a "not configured" message.

**Cause:** One or more required settings are missing from your configuration.

**Solution:** Verify that all four required settings are present in `appsettings.json` under `PiwikPRO:Connector`:

```json
{
  "PiwikPRO": {
    "Connector": {
      "BaseUrl": "https://your-instance.piwik.pro",
      "WebSiteId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "ClientId": "your-client-id",
      "ClientSecret": "your-client-secret"
    }
  }
}
```

The connector considers itself configured when `BaseUrl` and `WebSiteId` are set, and either an `AccessToken` or both `ClientId` and `ClientSecret` are provided.

## Dashboard Shows No Data

The dashboard loads but all charts and tables are empty.

**Possible causes:**

- **Wrong WebSiteId.** The Website ID does not match any app in your Piwik PRO instance. Double-check the value in your Piwik PRO administration panel under Sites & Apps.
- **No data for the selected date range.** Switch to a longer date range (e.g., Last 30 Days) to confirm data exists.
- **Insufficient API permissions.** The API credentials (ClientId/ClientSecret) may not have read access to the analytics data. Verify the credentials have the required scopes in Piwik PRO.
- **Multi-site mapping mismatch.** If you use multiple sites, ensure the correct Piwik PRO app is mapped to the site you are viewing. See [Multi-Site Support](multi-site.md).

## Tracking Script Not Appearing on the Front End

The Piwik PRO tracking script is not injected into your site's HTML.

**Check the following:**

1. **`InjectTrackingScript` is enabled.** This setting must be `true` (it defaults to `true` if omitted, but verify it has not been explicitly set to `false`).
2. **`ContainerId` is set.** The tracking script requires a valid Container ID to load the Piwik PRO tag manager container.
3. **Not in edit mode.** The tracking script is intentionally suppressed when content is viewed inside the Optimizely editor to avoid skewing analytics.
4. **Tag helper is registered.** Ensure the Piwik PRO tag helper is available in your `_ViewImports.cshtml`:
   ```
   @addTagHelper *, PiwikPRO.Optimizely.Connector
   ```
5. **`required-client-resources` is in the head.** Your layout must include `@Html.RequiredClientResources("Header")` in the `<head>` section for the tracking script to be injected.

## Content Analytics Widget Not Visible

The widget does not appear in the assets pane when editing content.

**Possible causes:**

- **Page is not published.** Analytics are only available for published content. Save and publish the page first.
- **Block has no referencing pages.** For blocks and non-page content, the widget aggregates analytics from pages that reference the content. If no pages use the block, there is no data to show.
- **Module not loaded.** Verify the Piwik PRO module is registered (see the startup exception section below).

## Authentication or Authorization Errors

API calls return 401 or 403 errors.

**Check the following:**

- **Credentials have not expired.** If using client credentials, verify they are still valid in your Piwik PRO instance. Regenerate them if necessary.
- **`BaseUrl` is correct.** The URL must point to your Piwik PRO instance (e.g., `https://your-instance.piwik.pro`) without a trailing slash or path.
- **User has the required role.** Dashboard access requires the `piwikpro:admin` policy, which defaults to the WebAdmins role. See [Authorization](authorization.md) to customize.

## Startup Exception About Missing Module

The application throws an exception at startup referencing the Piwik PRO module.

**Solution:**

1. Ensure `services.AddPiwikPRO()` is called in your `Startup.cs` or `Program.cs` service configuration.
2. Verify that the `module.config` file is included in the NuGet package and is being deployed to the `modules/PiwikPRO/` directory.
3. If you are running from source (not the NuGet package), confirm that the `MappingPhysicalFileProvider` is configured to serve the module's static files.
