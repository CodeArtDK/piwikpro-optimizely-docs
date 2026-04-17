# Configuration Reference

All Piwik PRO Optimizely Connector settings are configured under the `PiwikPRO:Connector` section in `appsettings.json`.

```json
{
  "PiwikPRO": {
    "Connector": {
      "BaseUrl": "https://your-instance.piwik.pro",
      "ClientId": "your-client-id",
      "ClientSecret": "your-client-secret",
      "WebSiteId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    }
  }
}
```

## Required Settings

These settings must be provided for the connector to function.

| Setting | Type | Description |
|---------|------|-------------|
| `BaseUrl` | `string` | The URL of your Piwik PRO instance (e.g., `https://your-instance.piwik.pro`). |
| `WebSiteId` | `string` | UUID of the website or app configured in Piwik PRO. |
| `ClientId` | `string` | OAuth client ID used to authenticate with the Piwik PRO API. Required unless `AccessToken` is set. |
| `ClientSecret` | `string` | OAuth client secret used to authenticate with the Piwik PRO API. Required unless `AccessToken` is set. |
| `AccessToken` | `string` | Static access token as an alternative to `ClientId` + `ClientSecret`. If set, `ClientId` and `ClientSecret` are not needed. |

## Tracking Settings

Control how and what the connector tracks on your site.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Enabled` | `bool` | `true` | Master switch for the entire connector. Set to `false` to disable all tracking and dashboard features. |
| `ContainerId` | `string` | | Container UUID for Piwik PRO Tag Manager. When set with `InjectTrackingScript`, injects the Tag Manager container script. When omitted, falls back to the standard `ppas.js` tracking script. |
| `InjectTrackingScript` | `bool` | `false` | Automatically inject the Piwik PRO tracking script into every page. Requires `ContainerId` to be set. |
| `TrackLoggedInUserAsUserId` | `bool` | `false` | Send the authenticated user's name as the Piwik PRO User ID, enabling cross-device tracking for logged-in users. |
| `TrackContentBlocks` | `bool` | `false` | Enable tracking for Optimizely content blocks. When enabled, analytics are aggregated across all pages that reference a given block. |

## Visitor Group / Audience Tracking

Track Optimizely Visitor Group matches as Piwik PRO events or custom dimensions.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `TrackAudiencesAsEvents` | `bool` | `false` | Track Optimizely Visitor Group matches as Piwik PRO events. |
| `AudienceDimensionId` | `int?` | `null` | Custom dimension ID in Piwik PRO to use for storing audience/visitor group data. Leave `null` to disable dimension-based audience tracking. |

## Custom Dimension Auto-Tracking

Automatically send content metadata as Piwik PRO custom dimensions.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `ContentTypeDimensionId` | `int?` | `null` | Custom dimension ID for tracking the Optimizely content type name. Leave `null` to disable. |
| `ContentLanguageDimensionId` | `int?` | `null` | Custom dimension ID for tracking the content language. Leave `null` to disable. |

## Advanced Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `UseSimulatedLiveMapData` | `bool` | `false` | Use simulated data for the Live Map dashboard. Useful for demos and development environments without real traffic. |
| `SiteTimezone` | `string?` | `null` | IANA timezone identifier (e.g., `Europe/Warsaw`) used to localize real-time event timestamps on the Live Map. |

## Code-Based Configuration

You can configure the connector in code by passing a delegate to `AddPiwikPRO()` in your `Startup.cs` or `Program.cs`:

```csharp
services.AddPiwikPRO(options =>
{
    options.Enabled = true;
    options.InjectTrackingScript = true;
    options.ContentTypeDimensionId = 2;
    options.ContentLanguageDimensionId = 3;
    options.TrackAudiencesAsEvents = true;
    options.AudienceDimensionId = 1;
    options.TrackLoggedInUserAsUserId = true;
    options.SiteTimezone = "Europe/Warsaw";
});
```

> **Note:** Options set in code take precedence over values defined in `appsettings.json`. This allows you to use `appsettings.json` for environment-specific defaults while overriding specific settings programmatically.
