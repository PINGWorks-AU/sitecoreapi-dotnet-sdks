# PINGWorks.SitecoreExperienceEdge.Common

This library provides shared utility classes to PINGWorks.SitecoreExperienceEdge.* libraries. It is a
transient dependency of other `PINGWorks.SitecoreExperienceEdge.*` libraries.

## Getting started
You should not install this library directly. Instead, install an SDK library that provides 
service interfaces to access Sitecore Experience Edge's APIs, such as:

- PINGWorks.SitecoreExperienceEdge.AdminSDK for managing SitecoreAI configuration
- PINGWorks.SitecoreExperienceEdge.ContentSDK for accessing SitecoreAI content
- PINGWorks.SitecoreExperienceEdge.EventsSDK for recording SitecoreAI analytics

## Cross-cutting HttpClient defaults

Every `AddSitecore*Sdk(…)` call routes through `AddSitecoreEECommon()` (it's the shared setup
hook). That call uses `ConfigureHttpClientDefaults` to apply two cross-cutting defaults to
**every** named/typed HttpClient registered against the container — including any HttpClients
the host app registers itself for its own services:

| Default | Build | Notes |
| - | - | - |
| `AddStandardResilienceHandler()` | All builds | Microsoft's standard pipeline (retry, timeout, circuit breaker, concurrency limit). |
| `AddHttpMessageHandler<DebugLogger>()` | DEBUG only | Writes request URL + headers + body and response status + body to `Console.Out`. Stripped from Release. |

`AddSitecoreEECommon` is idempotent — calling it from multiple SDK registrations applies the
defaults exactly once.

### Customizing the resilience options for a specific client

Use the standard ASP.NET options-with-name pattern. The HttpClient name for a typed-client
registration is the unqualified type name of the service interface:

```csharp
// Increase the per-attempt timeout for ISitecoreAdminSdk only:
services.Configure<HttpStandardResilienceOptions>( nameof( ISitecoreAdminSdk ), opts =>
{
    opts.AttemptTimeout.Timeout      = TimeSpan.FromSeconds( 60 );
    opts.TotalRequestTimeout.Timeout = TimeSpan.FromMinutes( 5 );
} );
```

For the named OAuth-token client (registered by AgentSDK / AdminSDK / PagesSDK / PublishingSDK
/ SitesSDK), use the constant `DefaultTokenProvider.HttpClientName` as the options name.

### Opting out entirely

There is no built-in opt-out. If your app already configures resilience on its own HttpClients
and the duplicate handler would interfere, raise an issue — we can add a `ConfigureSitecoreEE`
overload that takes a `bool applyResilienceToAll` flag.

## Known Issues
None.
