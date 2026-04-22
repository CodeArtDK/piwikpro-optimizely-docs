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
| `InjectTrackingScript` | `bool` | `false` | Automatically inject the Piwik PRO tracking script into every page. Set `ContainerId` if you use Piwik PRO Tag Manager; otherwise the connector falls back to the standard `ppas.js` tracking script using `WebSiteId`. |
| `TrackLoggedInUserAsUserId` | `bool` | `false` | Send the authenticated user's name as the Piwik PRO User ID, enabling cross-device tracking for logged-in users. |
| `TrackContentBlocks` | `bool` | `false` | Enable tracking for Optimizely content blocks. When enabled, analytics are aggregated across all pages that reference a given block. |

## Visitor Group / Audience Tracking

Track Optimizely Visitor Group matches as Piwik PRO events or custom dimensions.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `TrackAudiencesAsEvents` | `bool` | `false` | Track Optimizely Visitor Group matches as Piwik PRO events. |
| `AudienceDimensionId` | `int?` | `null` | Custom dimension ID in Piwik PRO to use for storing audience/visitor group data. Leave `null` to disable dimension-based audience tracking. |

> **CMS 13 note:** audience tracking is a no-op on CMS 13 until the connector is updated for the new visitor-group pipeline. Both options above are accepted but do nothing when the host runs on CMS 13 / .NET 10. Every other signal on this page (custom dimensions, content-type tracking, user ID, etc.) works identically on both CMS versions.

## Custom Dimension Auto-Tracking

The connector can automatically enrich every page view with metadata about the current Optimizely content by pushing values into Piwik PRO **custom dimensions**. Each option below is `null` by default — pointing an option at a dimension ID turns the signal on, leaving it `null` turns it off.

### Before you set any of these: create the dimension in Piwik PRO

A dimension only exists once you create it in the Piwik PRO admin UI. The connector does **not** create dimensions for you.

1. In Piwik PRO, go to **Administration → Sites & apps → [your site] → Custom dimensions**.
2. Click **Add an event dimension** (for per-page-view signals like `ContentType`) or **Add a session dimension** (for signals that persist for the whole visit like `Audience`). See [Event vs. session scope](#event-vs-session-scope) below.
3. Give it a descriptive name (e.g. `ContentType`, `ContentLanguage`, `Audience`). The admin UI then shows it in a list with two numeric columns:
   - **Slot** — per-category index, starts at 1 for session dimensions and 1 for event dimensions.
   - **Dimension ID** — globally unique integer across both categories.
4. **Use the Dimension ID** in `appsettings.json`. The connector emits Piwik PRO's `setCustomDimensionValue(<id>, <value>)` JavaScript call, which expects the globally-unique ID. The Slot is a display/ordering concept only; using it will write values to a different dimension (or none at all).

Example: you create three dimensions and the admin shows:

| Category | Slot | Dimension ID | Name |
|----------|------|--------------|------|
| Session | 1 | 1 | Audience |
| Event | 1 | 2 | ContentType |
| Event | 2 | 3 | ContentLanguage |

The config that wires those up is:

```json
"AudienceDimensionId": 1,
"ContentTypeDimensionId": 2,
"ContentLanguageDimensionId": 3
```

### Event vs. session scope

Piwik PRO stores an **event** dimension value alongside each page view and an **session** dimension value once per visit (overwritten if the value changes). Choose the category when you create the dimension — it cannot be changed afterwards.

| Signal | Recommended scope | Why |
|--------|-------------------|-----|
| `Audience` (Visitor Groups) | Session | Visitor-group membership is a property of the visitor/session, not of any one page |
| `ContentType`, `ContentLanguage`, `ContentSection`, `ContentPublishDate`, `ContentAuthor`, `ContentCategory` | Event | Values change per page view |

### Configuration keys

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `ContentTypeDimensionId` | `int?` | `null` | Optimizely content type name (e.g. `StandardPage`, `ArticlePage`). Event-scoped. |
| `ContentLanguageDimensionId` | `int?` | `null` | Content language branch (`en`, `sv`, …). Event-scoped. |
| `ContentSectionDimensionId` | `int?` | `null` | Name of the first-level ancestor page under the start page (e.g. `News`, `Products`). Useful for grouping by top-level section. Event-scoped. |
| `ContentPublishDateDimensionId` | `int?` | `null` | ISO-8601 `yyyy-MM-dd` publish/change date, resolved from `IChangeTrackable.Changed` with fallback to `IVersionable.StartPublish`. Enables freshness analysis. Event-scoped. |
| `ContentAuthorDimensionId` | `int?` | `null` | `CreatedBy` username of the content. **Usernames are often PII — see the [GDPR note](#pii--gdpr----trackloggedinuserasuserid) before enabling.** Event-scoped. |
| `ContentCategoryDimensionId` | `int?` | `null` | Comma-joined Optimizely category names assigned to the content (requires the content to implement `ICategorizable`). Event-scoped. |

Values are skipped when the configured option is `null` or when the source data is missing for a given page (e.g. no `Changed` timestamp, no categories assigned). Tracking never throws — failures log at `Debug` and the page view still fires.

For pushing your own custom dimensions from code (e.g. a "FormSubmitted" dimension keyed off a controller), see [Extending Tracking → Custom Tracking from Your Code](extending-tracking.md#custom-tracking-from-your-code).

## Advanced Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `UseSimulatedLiveMapData` | `bool` | `false` | Use simulated data for the Live Map dashboard. Useful for demos and development environments without real traffic. |
| `SiteTimezone` | `string?` | `null` | IANA timezone identifier (e.g., `Europe/Warsaw`) used to localize real-time event timestamps on the Live Map. |

### Aggregation Tuning (block-heavy sites)

These advanced settings tune how the Content Analytics widget aggregates per-content data for non-page content (shared blocks). The defaults are appropriate for most sites; raise them only if heavily-shared blocks have their analytics truncated.

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `MaxPagesForAggregatedSessions` | `int` | `5` | Maximum number of referencing pages queried when aggregating sessions for non-page content. Each page triggers a separate Piwik PRO Sessions API call, so this caps per-request API load. |
| `SessionsPerPageForAggregation` | `int` | `20` | Sessions fetched per referencing page during aggregated session queries. Each page contributes up to this many rows to the merged result before de-duplication. |
| `MaxAggregatedSessionsTotal` | `int` | `100` | Hard upper bound on the total number of aggregated session rows returned for non-page content. Once the merged, de-duplicated list reaches this size, further pages are not queried. |

## PII / GDPR -- `TrackLoggedInUserAsUserId`

When `TrackLoggedInUserAsUserId` is `true` (default: `false`), the connector sends the authenticated Optimizely user's identity to Piwik PRO as a user ID on every tracked page view. In most CMS deployments this username is a personally-identifiable value (often an email address or real name).

Before enabling, confirm with your data protection officer that:

- Your Piwik PRO contract / DPA covers user-ID-based tracking of authenticated users.
- Your privacy notice and cookie banner reflect this data flow.
- Your data-retention settings in Piwik PRO are appropriate for identifiable data.

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
