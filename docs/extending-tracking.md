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

## Content Block Tracking

Piwik PRO can report impressions and interactions on individual HTML elements -- teasers, hero banners, videos, CTA buttons -- so you can see which blocks in your layouts actually pull their weight. Two pieces are required:

1. **Connector config** -- set `"PiwikPRO:Connector:TrackContentBlocks": true`. This tells the loader to call `trackVisibleContentImpressions()` on page load and after each scroll, so impressions fire only for blocks that actually enter the viewport.
2. **HTML markup** -- flag each trackable element with `data-track-content` (or the `.piwikTrackContent` class). Add the optional attributes below for richer reporting.

### Attributes

| Attribute / Class | Where it goes | Purpose |
|---|---|---|
| `data-track-content` or `.piwikTrackContent` | Outer element | Marks the element as a trackable content block |
| `data-content-name="..."` | Same element | Human-readable block name (shows up in reports) |
| `data-content-piece="..."` or `.piwikContentPiece` | Inner element (image/video) | Identifies the specific creative piece |
| `data-content-target="..."` or `.piwikContentTarget` | Link/button | Destination for interaction clicks |
| `data-content-ignoreinteraction` or `.piwikContentIgnoreInteraction` | Target element | Disables automatic interaction tracking for this element |

If `data-content-name` is omitted, Piwik PRO falls back to the `alt`/`title`/`src` of the inner piece. If `data-content-piece` is omitted, the image `src` or the inner text is used. If `data-content-target` is omitted, the closest `<a href>` is used.

### Example: teaser block with image and CTA

```cshtml
@model TeaserBlock

<div class="teaserblock" data-track-content data-content-name="@Model.Heading">
    <a href="@Url.ContentUrl(Model.TargetLink)" data-content-target="@Url.ContentUrl(Model.TargetLink)">
        <img src="@Url.ContentUrl(Model.Image)"
             alt="@Model.Heading"
             data-content-piece="@Url.ContentUrl(Model.Image)" />
        <h2>@Model.Heading</h2>
        <p>@Model.Text</p>
        <span class="cta">@Model.ButtonText</span>
    </a>
</div>
```

### Example: video block

```cshtml
@model VideoBlock

<div class="videoblock" data-track-content data-content-name="@Model.Title">
    <video controls
           src="@Url.ContentUrl(Model.VideoFile)"
           poster="@Url.ContentUrl(Model.PosterImage)"
           data-content-piece="@Url.ContentUrl(Model.VideoFile)">
    </video>
</div>
```

### Example: trackable CTA button

```cshtml
<a href="/signup"
   class="btn btn-primary"
   data-track-content
   data-content-name="Free Trial CTA"
   data-content-piece="Sidebar Button"
   data-content-target="/signup">
    Start your free trial
</a>
```

### Further reading

- [`trackVisibleContentImpressions`](https://developers.piwik.pro/docs/trackvisiblecontentimpressions) -- developer reference for the impression API
- [`trackContentInteraction`](https://developers.piwik.pro/docs/trackcontentinteraction) -- developer reference for the interaction API
- [Set up content tracking](https://help.piwik.pro/support/questions/set-up-content-tracking/) -- Piwik PRO help article with markup and reporting walkthrough

---

## Sample Site Walkthrough

The connector repository ships an `AlloySampleSite` that wires up every piece of the tracking pipeline against the standard Optimizely Alloy templates. Use it as a copy-paste reference when adding the same patterns to your own solution.

| Feature | File / Location |
|---|---|
| Connector config (all options) | `src/AlloySampleSite/appsettings.json` |
| `AddPiwikPRO(options => { ... })` setup | `src/AlloySampleSite/Startup.cs` |
| Custom dimension from Razor via `IPiwikProTrackingService` | `src/AlloySampleSite/Views/Shared/Layouts/_Root.cshtml` |
| Server-side event (`AddEvent`) from controller | `src/AlloySampleSite/Controllers/SearchPageController.cs` |
| Content-block markup with `data-track-content` | `src/AlloySampleSite/Views/Shared/TeaserBlock.cshtml` (and similar block templates) |
| Audience-as-events config | `TrackAudiencesAsEvents: true` in `appsettings.json` |
| Content block impression tracking | `TrackContentBlocks: true` + `data-track-content` in the block templates |

Clone the repo, drop your Piwik PRO credentials into `appsettings.json`, and run the site to see the dashboard, the per-content widget, and all of the tracking signals above firing against live content.

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
