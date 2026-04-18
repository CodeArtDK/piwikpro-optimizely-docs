# Extending Tracking

What the connector tracks automatically, what you can configure, and how to push your own custom events, goals, and dimensions from code.

---

## Automatic Tracking

Once `InjectTrackingScript` is `true`, the Piwik PRO loader emits a page view on every request. The connector layers additional signals on top -- each opt-in via `PiwikPRO:Connector` configuration. See the [Configuration Reference](configuration.md) for full details on each key.

| Signal | Config key | What is emitted |
|---|---|---|
| Visitor groups as events | `TrackAudiencesAsEvents` (`bool`) | `trackEvent('Visitor Groups', '<audience-name>')` per match |
| Visitor groups as dimension | `AudienceDimensionId` (`int?`) | Comma-joined audience names on the given dimension |
| Content type | `ContentTypeDimensionId` (`int?`) | Optimizely content type name |
| Content language | `ContentLanguageDimensionId` (`int?`) | Content language branch (`en`, `sv`, ...) |
| Content section | `ContentSectionDimensionId` (`int?`) | Name of the first-level ancestor page under the start page |
| Content publish/change date | `ContentPublishDateDimensionId` (`int?`) | ISO-8601 `yyyy-MM-dd` from `Changed` (falling back to `StartPublish`) |
| Content author | `ContentAuthorDimensionId` (`int?`) | `CreatedBy` username -- **PII, see [GDPR caveat](configuration.md#pii--gdpr----trackloggedinuserasuserid)** |
| Content categories | `ContentCategoryDimensionId` (`int?`) | Comma-joined category names for `ICategorizable` content |
| Logged-in user ID | `TrackLoggedInUserAsUserId` (`bool`) | `setUserId(<username>)` -- **PII, see GDPR caveat** |
| Content block impressions | `TrackContentBlocks` (`bool`) | Adds `trackVisibleContentImpressions` to the loader so DOM-flagged blocks report impressions |
| Per-page goal | Assigned by an editor in the Content Analytics widget | `trackGoal(<goal-uuid>)` when a mapping exists for the content |

All dimension signals are skipped when the corresponding option is `null` and when the relevant source data is missing (e.g., no date, no author, no categories). Tracking never throws; on failure it logs at `Debug` and continues.

---

## Custom Tracking from Your Code

Inject `IPiwikProTrackingService` anywhere you have a DI context -- controller, middleware, Razor view, page/block controller, view component. The service is scoped to the current request and accumulates instructions that the `<piwikpro-tracking />` tag helper flushes into the rendered page.

```csharp
tracking.TrackEvent(category, action, name, value);     // all but category/action optional
tracking.TrackGoal(goalId);                             // numeric id
tracking.TrackGoal(goalUuid);                           // UUID string (recommended)
tracking.SetCustomDimension(id, value);                 // immediate
tracking.SetCustomDimension(id, valueFactory);          // Func<string?>
tracking.SetCustomDimension(id, contextFactory);        // Func<HttpContext?, IContent?, string?>
tracking.SetCustomVariable(slot, name, value, scope);   // scope defaults to "page"
tracking.AddAudience(name);                             // tracked as event and/or dimension per config
```

See the [Tracking API reference](tracking-api.md) for method-by-method docs and rendered-JS examples.

### Example: emit a section dimension from a Razor view

```cshtml
@using PiwikPRO.Optimizely.Connector.Services
@inject IPiwikProTrackingService Tracking

@{
    if (Model.CurrentPage != null)
    {
        Tracking.SetCustomDimension(7, Model.Section?.Name ?? "unknown");
    }
}
```

### Example: track a goal on form submit

```csharp
public IActionResult Submit(ContactForm form)
{
    // ...validate and persist...
    _tracking.TrackGoal("abc-123-goal-uuid");   // by UUID (recommended)
    // or:
    _tracking.TrackGoal(42, revenue: 0);         // by numeric id
    return View("ThankYou");
}
```

> **Note:** Because the service is scoped to the current request, tracking calls made before a `RedirectToAction` are lost -- the redirect starts a new request. Track on the POST response page, or fire client-side on the thank-you page.

---

## Per-Page Goal Mapping via the Widget

In the CMS edit view, open any page. The Piwik PRO widget in the assets pane has a **Page Goal** picker. Select a goal from the Piwik PRO manage-goals list and save. From then on, every visitor page view for that content will emit `trackGoal(<uuid>)` alongside the page view -- no code required.

---

## Opting Specific Pages Out

Both the loader script (when `InjectTrackingScript: true`) and the `<piwikpro-tracking />` tag helper emit on every request through Optimizely's client-resource pipeline. To opt out a specific page or area:

- **In your layout**, conditionally render `<piwikpro-tracking />` based on page type or a content property.
- **In the loader**, implement a custom `IClientResourceRegistrator` that runs after the connector's and suppresses the required resources when appropriate.
- **For editor/preview traffic**, no action needed -- requests under `/episerver/` and in CMS Edit/Preview mode are already excluded.

To separate editor-only traffic from real visitors in Piwik PRO, set a custom dimension that flags the Optimizely `PageEditing.PageIsInEditMode` state so you can filter it out in reports.

---

## PII / GDPR

Signals that carry personally-identifiable data are called out with a **PII** badge above. Before enabling any of them, confirm with your data protection officer that your Piwik PRO DPA, privacy notice, and retention settings cover identifiable user data. See the full [PII / GDPR note](configuration.md#pii--gdpr----trackloggedinuserasuserid) in the configuration reference.
