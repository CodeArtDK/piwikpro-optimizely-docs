# Authorization

The Piwik PRO dashboard and all API endpoints are protected by an authorization policy. By default, only users in the **WebAdmins** role have access.

## Default Behavior

When you call `services.AddPiwikPRO()` without specifying a policy, the connector registers a policy named `piwikpro:admin` that requires the `WebAdmins` role. This means only Optimizely CMS administrators can view the dashboard or manage settings.

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

## Notes

- The policy name used internally is `piwikpro:admin` (defined in `Constants.PolicyName`).
- All dashboard controllers and API endpoints use `[Authorize(Policy = Constants.PolicyName)]`, so changing the policy applies everywhere consistently.
- If you use an external identity provider, make sure the roles or claims you reference are present in the tokens or identity issued to CMS users.
