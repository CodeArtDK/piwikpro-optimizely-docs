# Programmatic Tracking API

The Piwik PRO Optimizely Connector exposes `IPiwikProTrackingService` -- a scoped, per-request service that lets you track events, goals, custom dimensions, custom variables, and audience memberships from server-side C# code. All tracking instructions are accumulated during the request and rendered as `_ppas.push(...)` JavaScript calls in the page output.

## How Tracking Works

The tracking pipeline has four stages:

1. **Script injection** -- The Piwik PRO container loader JS is injected into the page header when you include `<required-client-resources area="Header" />` in your layout (requires `InjectTrackingScript: true`; `ContainerId` is needed only if you use Piwik PRO Tag Manager, otherwise the connector falls back to the standard `ppas.js` tracking script using `WebSiteId`).
2. **Accumulation** -- During request processing, your code calls methods on `IPiwikProTrackingService` to queue tracking instructions. The service stores them in memory for the duration of the request.
3. **Rendering** -- The `<piwikpro-tracking />` tag helper awaits `BuildClientSideScriptAsync()` (from inside its `ProcessAsync` override) and emits all accumulated instructions as an inline `<script>` block.
4. **Exclusion** -- Requests in CMS Edit mode, Preview mode, or under the Optimizely protected path (`/episerver/`) are automatically suppressed. No tracking calls are emitted for editors working in the CMS.

### Layout Setup

Both the client resources and the tracking tag helper must be present in your layout:

```html
<!DOCTYPE html>
<html>
<head>
    <!-- Injects the Piwik PRO container loader -->
    <required-client-resources area="Header" />
</head>
<body>
    @RenderBody()

    <!-- Renders all accumulated tracking calls -->
    <piwikpro-tracking />

    <required-client-resources area="Footer" />
</body>
</html>
```

## Injecting the Service

### Razor View Injection

```cshtml
@using PiwikPRO.Optimizely.Connector.Services
@inject IPiwikProTrackingService PiwikTracking

@{
    PiwikTracking.TrackEvent("Page", "View", Model.CurrentPage.Name);
}
```

### Constructor Injection

```csharp
using PiwikPRO.Optimizely.Connector.Services;

public class ArticlePageController : PageController<ArticlePage>
{
    private readonly IPiwikProTrackingService _tracking;

    public ArticlePageController(IPiwikProTrackingService tracking)
    {
        _tracking = tracking;
    }

    public IActionResult Index(ArticlePage currentPage)
    {
        _tracking.TrackEvent("Article", "View", currentPage.Name);
        return View(currentPage);
    }
}
```

## Available Methods

### TrackEvent

Track a custom event with a category, action, optional name, and optional numeric value.

```csharp
void TrackEvent(string category, string action, string? name = null, int? value = null);
```

**Examples:**

```csharp
// Basic event
PiwikTracking.TrackEvent("Downloads", "PDF", "Product Brochure");

// Event with a numeric value
PiwikTracking.TrackEvent("Video", "Play", "Homepage Hero", 1);

// Minimal event (category + action only)
PiwikTracking.TrackEvent("Navigation", "Click");
```

**Rendered output:**

```javascript
_ppas.push(['trackEvent', 'Downloads', 'PDF', 'Product Brochure']);
_ppas.push(['trackEvent', 'Video', 'Play', 'Homepage Hero', 1]);
```

### TrackGoal

Track a goal conversion, identified by either a UUID string or a numeric ID. An optional revenue value can be attached.

```csharp
void TrackGoal(string goalUuid, int? revenue = null);
void TrackGoal(int goalId, int? revenue = null);
```

**Examples:**

```csharp
// Track by UUID (recommended -- matches the Piwik PRO goal ID)
PiwikTracking.TrackGoal("a1b2c3d4-e5f6-7890-abcd-ef1234567890");

// Track by numeric ID with revenue
PiwikTracking.TrackGoal(1, revenue: 99);
```

**Rendered output:**

```javascript
_ppas.push(['trackGoal', 'a1b2c3d4-e5f6-7890-abcd-ef1234567890']);
_ppas.push(['trackGoal', 1, 99]);
```

### SetCustomDimension

Set a custom dimension value. There are three overloads for different levels of flexibility.

> **Note:** Dimension IDs must first be created in Piwik PRO under **Analytics > Custom Dimensions**. The `id` parameter corresponds to the dimension slot number assigned there.

#### Immediate value

```csharp
void SetCustomDimension(int id, string value);
```

```csharp
PiwikTracking.SetCustomDimension(4, "2024-01-15");
```

#### Factory (computed at render time)

```csharp
void SetCustomDimension(int id, Func<string?> valueFactory);
```

```csharp
PiwikTracking.SetCustomDimension(5, () => DateTime.UtcNow.ToString("yyyy-MM-dd"));
```

The factory is invoked when `BuildClientSideScriptAsync()` runs, so the value reflects the state at render time rather than at the point of registration.

#### Context-aware factory

```csharp
void SetCustomDimension(int id, Func<HttpContext?, IContent?, string?> valueFactory);
```

```csharp
PiwikTracking.SetCustomDimension(6, (httpContext, content) =>
    content?.ContentLink.ID.ToString() ?? "unknown");
```

This overload receives the current `HttpContext` and the resolved `IContent` for the request, making it useful for dimensions that depend on page-level or user-level data.

**Precedence:** If the same dimension ID is registered through multiple overloads, the resolution order is: immediate values, then simple factories, then context-aware factories. Later resolutions overwrite earlier ones for the same ID.

**Rendered output (all overloads):**

```javascript
_ppas.push(['setCustomDimensionValue', 4, '2024-01-15']);
_ppas.push(['setCustomDimensionValue', 5, '2026-03-25']);
_ppas.push(['setCustomDimensionValue', 6, '42']);
```

### SetCustomVariable

Set a custom variable in a specific slot and scope.

```csharp
void SetCustomVariable(int slot, string name, string value, string scope = "page");
```

**Examples:**

```csharp
// Page-scoped variable (default)
PiwikTracking.SetCustomVariable(1, "MemberType", "Premium");

// Visit-scoped variable
PiwikTracking.SetCustomVariable(1, "MemberType", "Premium", "visit");
```

**Rendered output:**

```javascript
_ppas.push(['setCustomVariable', 1, 'MemberType', 'Premium', 'page']);
_ppas.push(['setCustomVariable', 1, 'MemberType', 'Premium', 'visit']);
```

### AddAudience

Register a visitor group (audience) name for the current request. Audiences are tracked as events, custom dimensions, or both, depending on configuration.

```csharp
void AddAudience(string audienceName);
```

```csharp
PiwikTracking.AddAudience("Returning Visitors");
PiwikTracking.AddAudience("High-Value Customers");
```

When `TrackAudiencesAsEvents` is `true`:

```javascript
_ppas.push(['trackEvent', 'Visitor Groups', 'Returning Visitors']);
_ppas.push(['trackEvent', 'Visitor Groups', 'High-Value Customers']);
```

When `AudienceDimensionId` is set (e.g., to dimension 7):

```javascript
_ppas.push(['setCustomDimensionValue', 7, 'Returning Visitors,High-Value Customers']);
```

## Automatic Tracking

The connector automatically tracks several data points without any code changes, based on configuration settings. See the [Configuration Reference](configuration.md) for full details on each setting.

| What is tracked | Configuration required | Output |
|---|---|---|
| **Content type** | `ContentTypeDimensionId` set to a dimension slot | `setCustomDimensionValue` with the Optimizely content type name |
| **Content language** | `ContentLanguageDimensionId` set to a dimension slot | `setCustomDimensionValue` with the content language branch code |
| **Visitor groups** (as events) | `TrackAudiencesAsEvents: true` | `trackEvent` with category `Visitor Groups` for each matched group |
| **Visitor groups** (as dimension) | `AudienceDimensionId` set to a dimension slot | `setCustomDimensionValue` with a comma-separated list of matched groups |
| **Logged-in user** | `TrackLoggedInUserAsUserId: true` | `setUserId` with the authenticated user's name |
| **Per-page goals** | Goal assigned by an editor in the Content Analytics widget | `trackGoal` with the configured goal UUID |

## Examples

### Tracking CMS Metadata from a Layout

Track the publish date and categories of every article from a shared layout or view component:

```cshtml
@using PiwikPRO.Optimizely.Connector.Services
@using EPiServer.Core
@inject IPiwikProTrackingService PiwikTracking

@if (Model.CurrentPage is ArticlePage article)
{
    PiwikTracking.SetCustomDimension(10, article.PublishDate.ToString("yyyy-MM-dd"));
    PiwikTracking.SetCustomDimension(11, string.Join(",", article.Categories ?? Array.Empty<string>()));
    PiwikTracking.TrackEvent("Content", "View", article.ContentType);
}
```

### Tracking Form Submissions from a Controller

Track form submissions as both an event and a goal conversion:

```csharp
[HttpPost]
public IActionResult SubmitContactForm(ContactFormModel model)
{
    if (!ModelState.IsValid)
    {
        return View(model);
    }

    // Process the form...

    _tracking.TrackEvent("Forms", "Submit", "Contact Form");
    _tracking.TrackGoal("b2c3d4e5-f6a7-8901-bcde-f12345678901");

    return RedirectToAction("ThankYou");
}
```

> **Note:** Because `IPiwikProTrackingService` is scoped to the current request, tracking calls made before a `RedirectToAction` will be lost -- the redirect starts a new request. To track conversions across redirects, use client-side JavaScript on the thank-you page or track the goal on the POST request's own response page.

### Tracking E-commerce Actions

Track product views and purchases by combining events and custom dimensions:

```csharp
public class ProductPageController : PageController<ProductPage>
{
    private readonly IPiwikProTrackingService _tracking;

    public ProductPageController(IPiwikProTrackingService tracking)
    {
        _tracking = tracking;
    }

    public IActionResult Index(ProductPage currentPage)
    {
        _tracking.TrackEvent("Ecommerce", "ProductView", currentPage.ProductName);
        _tracking.SetCustomDimension(12, currentPage.ProductSku);
        _tracking.SetCustomDimension(13, currentPage.ProductCategory);
        _tracking.SetCustomDimension(14, currentPage.Price.ToString("F2"));

        return View(currentPage);
    }

    [HttpPost]
    public IActionResult AddToCart(ProductPage currentPage, int quantity)
    {
        _tracking.TrackEvent("Ecommerce", "AddToCart", currentPage.ProductName, quantity);
        _tracking.TrackGoal("c3d4e5f6-a7b8-9012-cdef-123456789012", revenue: (int)currentPage.Price);

        return View("Index", currentPage);
    }
}
```
