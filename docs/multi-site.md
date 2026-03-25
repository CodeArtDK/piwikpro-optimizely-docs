# Multi-Site Support

Optimizely CMS installations often host multiple sites. The Piwik PRO connector lets you map each Optimizely site to a different Piwik PRO app (website), so that analytics data is tracked and displayed for the correct property.

## Configuring Site Mappings

1. In the Optimizely CMS admin interface, navigate to **Piwik PRO > Settings**.
2. Each Optimizely site defined in your installation is listed with a dropdown of available Piwik PRO apps.
3. Select the Piwik PRO app that corresponds to each site.
4. Click **Save**.

The connector uses the mapping to determine which Piwik PRO Website ID to use when injecting the tracking script and when querying analytics data. If no mapping exists for a site, the connector falls back to the default `WebSiteId` from the `PiwikPRO:Connector` configuration section.

## How Mappings Are Stored

Site-to-app mappings are persisted in Optimizely's Dynamic Data Store (DDS). This means:

- Mappings survive application restarts and redeployments.
- They are stored in the CMS database alongside other Optimizely DDS data.
- No additional database tables or migration steps are required.

## API Endpoints

The connector exposes REST endpoints under `/episerver/PiwikPRO/api/` for managing site mappings programmatically. These endpoints require the `piwikpro:admin` authorization policy (see [Authorization](authorization.md)).
