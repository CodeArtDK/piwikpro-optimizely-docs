# Getting Started

This guide walks you through installing, configuring, and verifying the Piwik PRO Optimizely Connector in your Optimizely CMS project. Both CMS 12 (on .NET 8) and CMS 13 (on .NET 10) are supported from the same NuGet package.

---

## 1. Prerequisites

Before you begin, make sure you have the following:

- An Optimizely CMS project set up and running, either:
    - **Optimizely CMS 12** (EPiServer.CMS 12.x) on **.NET 8**, or
    - **Optimizely CMS 13** (EPiServer.CMS 13.x) on **.NET 10**
- **A Piwik PRO account** with access to the Administration panel. If you do not have one yet, you can [sign up for a free trial](https://piwik.pro/business-plan/?utm_campaign=codeart).

> **CMS 13 note:** audience (Visitor Group) tracking is a no-op on CMS 13 until the connector is updated for the new visitor-group pipeline. Every other feature — tracking script injection, events/goals/custom dimensions, analytics dashboard, content analytics widget, multi-site mapping — is fully supported on both CMS versions.

---

## 2. Install the NuGet Package

Add the connector package to your Optimizely site project:

```bash
dotnet add package PiwikPRO.Optimizely.Connector
```

On the first build after installation, a `modules/_protected/PiwikPRO/` folder will be created in your project directory. This folder contains the module's static assets and should not be committed to source control. Add it to your `.gitignore`:

```gitignore
modules/_protected/PiwikPRO/
```

---

## 3. Register the Connector

Open your `Startup.cs` (or wherever you configure services) and call `services.AddPiwikPRO()` after `services.AddCms()`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddCms();
    services.AddPiwikPRO();
}
```

This single call registers all required services, binds configuration, sets up the `piwikpro:admin` authorization policy, and registers the Optimizely protected module.

---

## 4. Register the Tag Helper

Open your `_ViewImports.cshtml` file and add the following line:

```cshtml
@addTagHelper *, PiwikPRO.Optimizely.Connector
```

This makes the `<piwikpro-tracking />` tag helper available in all your Razor views.

---

## 5. Add Tracking to Your Layout

In your root layout file (typically `_Layout.cshtml`), add two elements:

1. The `<required-client-resources>` tag in the `<head>` section, which injects the Piwik PRO tracking script.
2. The `<piwikpro-tracking />` tag just before the closing `</body>` tag, which renders all accumulated tracking calls (events, goals, custom dimensions, audiences, and more).

Here is a complete example:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"]</title>

    <!-- Injects the Piwik PRO container tracking script -->
    <required-client-resources area="Header" />
</head>
<body>
    @RenderBody()

    <required-client-resources area="Footer" />

    <!-- Renders accumulated Piwik PRO tracking calls (events, goals, dimensions, etc.) -->
    <piwikpro-tracking />
</body>
</html>
```

---

## 6. Configure Credentials

Add the following section to your `appsettings.json`:

```json
{
  "PiwikPRO": {
    "Connector": {
      "Enabled": true,
      "BaseUrl": "https://your-instance.piwik.pro",
      "ClientId": "your-client-id",
      "ClientSecret": "your-client-secret",
      "WebSiteId": "your-website-id",
      "InjectTrackingScript": true
    }
  }
}
```

The table below describes each setting:

| Setting | Type | Description |
|---|---|---|
| `Enabled` | `bool` | Enables or disables the connector entirely. Set to `true` to activate. |
| `BaseUrl` | `string` | The base URL of your Piwik PRO instance (e.g., `https://your-instance.piwik.pro`). |
| `ClientId` | `string` | The OAuth Client ID used to authenticate with the Piwik PRO API. Required unless `AccessToken` is set. |
| `ClientSecret` | `string` | The OAuth Client Secret paired with the Client ID. Required unless `AccessToken` is set. |
| `AccessToken` | `string` | Static access token as an alternative to `ClientId` + `ClientSecret`. |
| `WebSiteId` | `string` | The GUID identifying which website (app) in Piwik PRO to track and query against. |
| `InjectTrackingScript` | `bool` | Defaults to `false`. Set to `true` to have the connector automatically inject the Piwik PRO container script into every page. Leave at `false` if you manage the tracking script yourself. |
| `ContainerId` | `string` | Optional. The Piwik PRO Tag Manager container ID. If omitted, defaults to the `WebSiteId`. |
| `TrackAudiencesAsEvents` | `bool` | Optional. When `true`, Optimizely Visitor Group matches are sent as Piwik PRO events. |
| `AudienceDimensionId` | `int` | Optional. Custom dimension ID used to store audience/visitor group information. |
| `ContentTypeDimensionId` | `int` | Optional. Custom dimension ID used to store the Optimizely content type name. |
| `ContentLanguageDimensionId` | `int` | Optional. Custom dimension ID used to store the content language. |
| `TrackContentBlocks` | `bool` | Optional. When `true`, tracks Optimizely content block impressions. |
| `TrackLoggedInUserAsUserId` | `bool` | Optional. When `true`, sets the Piwik PRO User ID to the logged-in Optimizely user. |

> **Security note:** Avoid committing `ClientId` and `ClientSecret` directly in `appsettings.json` for production environments. Use environment variables, Azure Key Vault, or another secrets manager instead.

---

## 7. Obtaining Piwik PRO Credentials

Follow these steps to get the API credentials and Website ID from your Piwik PRO account:

### Get your Website ID

1. Log in to your Piwik PRO instance.
2. Navigate to **Menu** (hamburger icon) and select **Administration**.
3. In the left sidebar, click **Sites & apps**.
4. Select the site you want to connect to Optimizely.
5. The **Website ID** (a GUID) is displayed on the site details page. Copy it.

### Create API Credentials (Client ID and Client Secret)

1. In the Piwik PRO Administration panel, navigate to **API Credentials** in the left sidebar (under **Profile** or **Integration**).
2. Click **Create new credentials**.
3. Give the credentials a descriptive name (e.g., "Optimizely CMS Connector").
4. Select the required permissions. The connector needs read access to Analytics data at a minimum.
5. Click **Create**.
6. Copy the **Client ID** and **Client Secret** that are displayed. The Client Secret is shown only once, so store it securely.

### Find your Base URL

Your Base URL is simply the root URL of your Piwik PRO instance, for example `https://your-instance.piwik.pro`. Do not include a trailing slash.

---

## 8. Verify It Works

Once everything is configured:

1. **Build your project:**

   ```bash
   dotnet build
   ```

2. **Run the site:**

   ```bash
   dotnet run
   ```

3. **Log in to the Optimizely CMS admin interface** (typically at `/episerver/cms`).

4. Look for **Piwik PRO** in the top navigation bar. Click it to open the analytics dashboard.

5. If the dashboard loads and displays data (or a message indicating it is waiting for data), the connector is working correctly.

6. Visit a page on the front end of your site and check your Piwik PRO instance to confirm that page views are being recorded.

If you see a configuration error in the dashboard, double-check that your `BaseUrl`, `ClientId`, `ClientSecret`, and `WebSiteId` values are correct and that the API credentials have sufficient permissions.

---

## Next Steps

- **[Configuration Reference](configuration.md)** -- Full details on all available configuration options, multi-site mapping, and advanced settings.
- **[Tracking API](tracking-api.md)** -- Learn how to send custom events, goals, dimensions, and audience data from your Optimizely code.
