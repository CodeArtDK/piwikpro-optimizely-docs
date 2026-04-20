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

The built-in signals above cover common content metadata, but most real implementations need to track product-, campaign- or domain-specific details as well. Inject `IPiwikProTrackingService` anywhere you have a DI context -- controller, middleware, Razor view, page/block controller, view component, tag helper, filter. The service is scoped to the current request and accumulates instructions that the `<piwikpro-tracking />` tag helper flushes into a single `<script>` block at the end of the rendered page.

```csharp
tracking.TrackEvent(category, action, name, value);     // all but category/action optional
tracking.TrackGoal(goalId);                             // numeric id
tracking.TrackGoal(goalUuid);                           // UUID string (recommended)
tracking.SetCustomDimension(id, value);                 // immediate value
tracking.SetCustomDimension(id, valueFactory);          // Func<string?>
tracking.SetCustomDimension(id, contextFactory);        // Func<HttpContext?, IContent?, string?>
tracking.SetCustomVariable(slot, name, value, scope);   // scope defaults to "page"
tracking.AddAudience(name);                             // tracked as event and/or dimension per config
```

See the [Tracking API reference](tracking-api.md) for method-by-method docs and rendered-JS examples.

### Recipe: add your own custom dimension

Say you want a `Product Category` dimension that fires whenever a visitor views a product page.

**1. Create the dimension in Piwik PRO.** In the admin UI go to **Administration → Sites & apps → [your site] → Custom dimensions** and click **Add an event dimension** (use *session* dimension instead if the value should persist across the whole visit). Name it `ProductCategory` and note the **Dimension ID** column — that's the globally-unique integer you pass to the connector, *not* the Slot number. See [Configuration → Custom Dimension Auto-Tracking](configuration.md#custom-dimension-auto-tracking) for the full primer on scopes and ID vs. Slot.

**2. Push a value on page view.** From the product page controller or view:

```csharp
public class ProductPageController : PageController<ProductPage>
{
    private readonly IPiwikProTrackingService _tracking;

    public ProductPageController(IPiwikProTrackingService tracking) => _tracking = tracking;

    public IActionResult Index(ProductPage currentPage)
    {
        // Dimension ID from the Piwik PRO admin (NOT the Slot column).
        _tracking.SetCustomDimension(12, currentPage.Category?.Name ?? "uncategorised");
        return View(currentPage);
    }
}
```

Or equivalently from the Razor view:

```cshtml
@using PiwikPRO.Optimizely.Connector.Services
@inject IPiwikProTrackingService Tracking

@{ Tracking.SetCustomDimension(12, Model.CurrentPage.Category?.Name ?? "uncategorised"); }
```

The connector renders this as:

```js
_ppas.push(['setCustomDimensionValue', 12, 'Mountain Bikes']);
```

**3. Prefer `Func<>` overloads when the value is expensive or context-aware.** The factory isn't invoked until the tag helper flushes, so you can wire it up in a middleware or base controller without paying the cost on every request where the value ends up unused:

```csharp
_tracking.SetCustomDimension(12, (http, content) =>
    content is ProductPage p ? p.Category?.Name : null);
```

When the factory returns `null` or an empty string, nothing is emitted for that dimension on that request.

### Recipe: fire your own custom events

Events are great for interactions that aren't page views -- search submits, video plays, CTA clicks, form steps. No Piwik PRO setup is needed up front; events are free-form and show up in the Analytics → Custom events report automatically.

**From a controller** (server-side, renders on the resulting page):

```csharp
public IActionResult Search(string query)
{
    _tracking.TrackEvent(
        category: "Search",
        action:   "Submitted",
        name:     query,            // optional
        value:    null);            // optional numeric
    return View("SearchResults", _service.Run(query));
}
```

**From a Razor view** (for conditional, render-time events):

```cshtml
@inject IPiwikProTrackingService Tracking
@{ if (Model.IsSubscribed) Tracking.TrackEvent("Newsletter", "StateRestored"); }
```

**Directly from client-side JS** (for interactions that happen after the page has loaded -- clicks, scroll depth, video progress). Bypass the connector and push straight to the Piwik PRO queue:

```html
<button type="button" onclick="_ppas.push(['trackEvent', 'CTA', 'Click', 'Hero-FreeTrial']);">
    Start free trial
</button>
```

> **`RedirectToAction` caveat:** the tracking service is scoped per request, so anything queued before a redirect is lost with that request. Queue tracking on the response that actually renders to the user (the thank-you/confirmation page), not on the POST handler that redirects.

### Recipe: track a goal on form submit

Goals are just named, optionally-revenue-bearing events that Piwik PRO aggregates into conversion reports. Create the goal in **Administration → Goals**, then reference it by UUID (preferred -- stable across environments) or numeric ID:

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

For per-page goals that don't need code (e.g. "visiting this page = conversion"), use the **Page Goal** picker in the Content Analytics widget instead -- see [Per-Page Goal Mapping](#per-page-goal-mapping-via-the-widget) below.

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
| `AddPiwikPRO(options => { ... })` with demo flags on | `src/AlloySampleSite/Startup.cs` |
| Custom dimension from Razor via `IPiwikProTrackingService` | `src/AlloySampleSite/Views/Shared/Layouts/_Root.cshtml` |
| Server-side `TrackEvent` from controller | `src/AlloySampleSite/Controllers/SearchPageController.cs` |
| Content-block markup with `data-track-content` | `src/AlloySampleSite/Views/Shared/Blocks/TeaserBlock.cshtml` (and other block templates) |
| Audience-as-events config | `options.TrackAudiencesAsEvents = true` in `Startup.cs` |
| Content block impression tracking | `options.TrackContentBlocks = true` + `data-track-content` in the block templates |

### Running the sample locally

The sample site is intentionally checked in with a clean `appsettings.json` -- tenant credentials are not committed. Set them via .NET user-secrets from the `src/AlloySampleSite` folder:

```bash
dotnet user-secrets set "PiwikPRO:Connector:BaseUrl"      "https://your.piwik.pro"
dotnet user-secrets set "PiwikPRO:Connector:WebSiteId"    "<site-guid>"
dotnet user-secrets set "PiwikPRO:Connector:ClientId"     "<oauth-client-id>"
dotnet user-secrets set "PiwikPRO:Connector:ClientSecret" "<oauth-client-secret>"
```

Non-secret demo values (dimension IDs, tracking flags) live in `Startup.cs` as in-code `AddPiwikPRO(...)` defaults, so the site exercises every tracking surface without any `appsettings.json` edits once credentials are in user-secrets. Dimension IDs in `Startup.cs` reference dimensions that you must first create in the demo Piwik PRO tenant -- see [Configuration → Custom Dimension Auto-Tracking](configuration.md#custom-dimension-auto-tracking).

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
