# Piwik PRO Connector for Optimizely CMS

Bring your Piwik PRO analytics data directly into the Optimizely CMS editorial experience. Dashboards, per-content analytics, live traffic visualization, visitor group tracking, and per-page goal assignment — all inside the CMS.

**[Get a free Piwik PRO trial](https://piwik.pro/business-plan/?utm_campaign=codeart)** to get started.

![Dashboard Overview](docs/images/dashboard-overview.png)

## Features

- **Automatic Tracking Script Injection** — Piwik PRO container script injected into every front-end page, with edit/preview mode excluded automatically
- **Event, Goal & Custom Dimension Tracking** — Programmatic API (`IPiwikProTrackingService`) to track events, goals, custom dimensions, and user IDs from your code
- **Visitor Group / Audience Tracking** — Automatically captures Optimizely Visitor Group membership as Piwik PRO events or custom dimensions (CMS 12 only)
- **Analytics Dashboard** — 8 pages in the CMS admin UI: Overview, Live Map, Sessions, Acquisition, Behavior, Visitors, Goals, Settings
- **Content Analytics Widget** — Per-page and per-block analytics in the CMS assets pane with KPIs, traffic sources, funnels, audiences, conversions, and session logs
- **Live Traffic Map** — Interactive D3.js force-directed graph showing page hierarchy with real-time traffic indicators
- **Per-Page Goal Tracking** — Editors pick a Piwik PRO goal to fire when a specific page is viewed
- **Multi-Site Support** — Map individual Optimizely sites to different Piwik PRO website IDs
- **Localized UI** — Translated into 11 languages: English, Swedish, Norwegian, Danish, Finnish, German, French, Spanish, Dutch, Japanese, Chinese (Simplified)

## Requirements

One of:

- **Optimizely CMS 12** (EPiServer.CMS 12.x) on **.NET 8**, or
- **Optimizely CMS 13** (EPiServer.CMS 13.x) on **.NET 10**.

Plus a Piwik PRO account with API access — **[start a free trial](https://piwik.pro/business-plan/?utm_campaign=codeart)**.

The NuGet ships both `lib/net8.0/` and `lib/net10.0/` — the install command is identical on both CMS versions. Audience (Visitor Group) tracking is currently a no-op on CMS 13; every other feature is fully supported on both versions.

## Quick Start

```
dotnet add package PiwikPRO.Optimizely.Connector
```

Then follow the [Getting Started Guide](docs/getting-started.md) for step-by-step setup instructions.

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](docs/getting-started.md) | Installation, registration, configuration, and first run |
| [Configuration Reference](docs/configuration.md) | All available settings with descriptions and defaults |
| [Dashboard](docs/dashboard.md) | Overview of all analytics dashboard pages inside the CMS |
| [Content Analytics Widget](docs/content-analytics-widget.md) | Per-page and per-block analytics in the editor |
| [Tracking API](docs/tracking-api.md) | Programmatic event, goal, custom dimension, and audience tracking |
| [Extending Tracking](docs/extending-tracking.md) | All auto-tracked signals at a glance, plus the custom tracking API and opt-out patterns |
| [Multi-Site & Settings](docs/multi-site.md) | Mapping Optimizely sites to Piwik PRO apps |
| [Authorization](docs/authorization.md) | Customizing who can access the dashboard |
| [Troubleshooting](docs/troubleshooting.md) | Common issues and how to resolve them |

## License

Copyright (c) Piwik PRO. All rights reserved.
