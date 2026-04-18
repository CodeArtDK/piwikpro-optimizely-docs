# Authorization

The Piwik PRO API endpoints are protected by an authorization policy. By default, only users in the **WebAdmins** role have access. View-rendering pages (the dashboard tabs and the content analytics widget) rely on Optimizely's standard `/episerver/` shell authentication.

## Default Behavior

When you call `services.AddPiwikPRO()` without specifying a policy, the connector registers a policy named `piwikpro:admin` that requires the `WebAdmins` role. The constant is exposed as `PiwikProPolicies.Admin` for use from external code.

## Customizing the Authorization Policy

You can override the default policy by passing a second parameter to `AddPiwikPRO`. This parameter is an `Action<AuthorizationPolicyBuilder>` that lets you define any policy supported by ASP.NET Core authorization.

### Allow Multiple Roles

To grant access to both Editors and Administrators:

```csharp
services.AddPiwikPRO(
    options => { },
    policy => policy.RequireRole("Editors", "Administrators")
);
```

### Require a Specific Claim

To restrict access based on a custom claim (for example, an `analytics-access` claim):

```csharp
services.AddPiwikPRO(
    options => { },
    policy => policy.RequireClaim("analytics-access", "true")
);
```

### Combine Requirements

You can chain multiple requirements. All of them must be satisfied:

```csharp
services.AddPiwikPRO(
    options => { },
    policy => policy
        .RequireAuthenticatedUser()
        .RequireRole("Editors")
        .RequireClaim("department", "marketing")
);
```

## Where the Policy Is Applied

- `[Authorize(Policy = PiwikProPolicies.Admin)]` is applied **per-action on API endpoints only** (e.g. site mapping CRUD, page goal assignment, dashboard data endpoints).
- View-rendering actions decorated with `[MenuItem]` (the dashboard tabs) and the content analytics widget rely on Optimizely's `/episerver/` shell authentication instead. Optimizely's shell already requires an authenticated CMS user with access to the `/episerver/` area before any menu item is reachable.
- Class-level `[Authorize]` is intentionally **not** applied to controllers that carry `[MenuItem]` actions. Doing so breaks Optimizely's `ReflectingMenuItemProvider` (used to enumerate menu items for navigation rendering) and crashes the top navigation. The per-action pattern on API endpoints achieves the same protection without that side effect.

## Notes

- The policy name used internally is `piwikpro:admin`. External code can reference it via the public constant `PiwikProPolicies.Admin`.
- If you use an external identity provider, make sure the roles or claims you reference are present in the tokens or identity issued to CMS users.
