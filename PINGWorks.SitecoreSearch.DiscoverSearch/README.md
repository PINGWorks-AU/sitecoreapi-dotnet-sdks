# PINGWorks.SitecoreSearch.DiscoverSearch

A .NET SDK for the **Sitecore Discover** platform (formerly **Reflektion**, or RFK).
A single package covers the three Discover surfaces — **Search**, **Events**, and
**Ingestion** — behind a single DI entry point and a consistent client pattern.

For an overview of Sitecore Discover, see the [official documentation](https://doc.sitecore.com/discover/en/).

This package coexists cleanly with `PINGWorks.SitecoreSearch.ExperienceSearch` — both can be
registered in the same DI container without conflict.

## Features

- Three typed, DI-friendly clients in one package: Search, Events, Ingestion
- Single DI entry point: `AddSitecoreDiscoverSearchSdk( opt => ... )`
- Async-first APIs with `CancellationToken` support
- `netstandard2.1` for wide compatibility
- Standard resilience handler (retries, circuit breaker, timeouts) on every HTTP client
- In-memory response cache for Search (5-minute default); per-widget cache-aware batching
- Automatic `__ruid` cookie management — **JS-SDK-compatible format**, so the same visitor is tracked under the same identifier whether the request comes from the browser SDK or our server-side SDK
- Pluggable `IDiscoverSessionProvider` for Blazor authentication integration
- Auto-fires `view_widget` events after every Search / Recommend / SearchBatch — zero-code widget impression tracking
- Full Discover event surface: widget, identity, commerce, navigation, plus a catch-all for custom events
- Strongly-typed document types from the source generator shipped in `PINGWorks.SitecoreSearch.Common`
- Semantic-search routing driven by the document type's `[SitecoreSearchSource]` metadata
- Recommendation `recipe` overrides and `rule` (boost / bury / blacklist) clauses
- Questions endpoint (AI Q&A) support

## Getting started

### Prerequisites

You will need:

- A **Sitecore Discover** account with at least one configured source
- Your **Discover customer key** — the single value from CEC of form `{domainId}-{domainHash}` (e.g. `"11111-2222222"`)
- A **Discover API key**
- The regional **endpoint URLs** for Search, Events, and Ingestion (these vary by data centre — for example `discover-apse2.sitecorecloud.io` for APSE2)

### Install

```bash
dotnet add package PINGWorks.SitecoreSearch.DiscoverSearch
```

`PINGWorks.SitecoreSearch.Common` (which carries the source generator) is brought in
automatically as a transitive dependency.

### AppSettings

```json
{
  "SitecoreSearch": {
    "Discover": {
      "CustomerKey":       "11111-2222222",
      "ApiKey":            "[Secret. Your Discover API key]",
      "SearchEndpoint":    "https://discover-apse2.sitecorecloud.io",
      "EventsEndpoint":    "https://events-apse2.sitecorecloud.io",
      "IngestionEndpoint": "https://ingestion-apse2.sitecorecloud.io"
    }
  }
}
```

> **Don't split the customer key.** Copy it verbatim from the CEC. The SDK extracts
> the parts internally — Search and Ingestion URLs use the `domainId` (left half), the
> Events URL uses the whole value, and the `__ruid` visitor cookie embeds the `domainHash`
> (right half). Splitting it manually is the most common source of misconfiguration.

We recommend Visual Studio's **User Secrets** for `ApiKey` during development.

### Register services

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder( args );

builder.Services.AddSitecoreDiscoverSearchSdk( opt =>
    builder.Configuration.GetSection( "SitecoreSearch:Discover" ).Bind( opt )
);
```

This one call registers all three clients — Search, Events, Ingestion — plus the session
provider and supporting infrastructure.

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

#### Service lifetimes

| Service | Lifetime | Why |
| - | - | - |
| `ISitecoreDiscoverSearchSdk`, `ISitecoreDiscoverEventsSdk`, `ISitecoreDiscoverIngestionSdk` | Transient (per `AddHttpClient<,>`) | The typed-client default — keeps the underlying `HttpMessageHandler` rotation from `IHttpClientFactory` working correctly. |
| `IDiscoverSessionProvider` | Scoped | Reads / writes the per-request `__ruid` cookie. |
| `IMemoryCache` | Singleton | Process-wide cache. |
| `IHttpContextAccessor` | Singleton | Per-request `HttpContext` resolved through ambient state. |

**Consuming the SDK from a Singleton service** (e.g. a hosted background worker): the SDK
clients are intentionally not Singleton — the search client has a scoped session dependency,
and the others rely on `HttpClient` from `IHttpClientFactory` which would suffer handler-rotation
problems if captured for the application lifetime. Use the standard `IServiceScopeFactory`
pattern instead:

```csharp
public class CatalogRefresher( IServiceScopeFactory scopes ) : BackgroundService
{
    protected override async Task ExecuteAsync( CancellationToken ct )
    {
        while ( !ct.IsCancellationRequested )
        {
            using var scope = scopes.CreateScope();
            var search = scope.ServiceProvider.GetRequiredService<ISitecoreDiscoverSearchSdk>();

            await search.Search<ProductsDocument>( "..." );
            await Task.Delay( TimeSpan.FromMinutes( 5 ), ct );
        }
    }
}
```

> **Fail-fast validation.** `AddSitecoreDiscoverSearchSdk` synchronously snapshots the
> options you configure and throws `InvalidOperationException` immediately if any are
> missing or malformed — `CustomerKey` (with format check), `ApiKey`, and all three
> endpoint URLs. You'll find out at app startup, not at the first failed search call.
> If you genuinely don't use one of the three sub-clients (e.g. an ingestion-only
> consumer), supply the regional URL anyway — the HTTP client is built lazily and never
> actually contacted if the corresponding SDK method isn't called.

### Author `sitecore-search.json`

Define your Discover sources in `discoverSources`. Every Discover source **must** carry
an `rfkId` — it's the widget identifier required on every API call. See the
`PINGWorks.SitecoreSearch.Common` README for the full schema.

```json
{
  "discoverSources": [
    {
      "id": "abcd1234-aaaa-bbbb-cccc-1111deadbeef",
      "name": "Products",
      "rfkId": "rfkid_products_search",
      "searchSettings": {
        "semanticRanking": { "enabled": true }
      },
      "fields": [
        { "name": "product_id", "type": "string", "retrieve": true, "key": true },
        { "name": "title",      "type": "string", "search": true,   "retrieve": true },
        { "name": "price",      "type": "number", "filter": true,   "sort": true, "retrieve": true },
        { "name": "in_stock",   "type": "bool",   "filter": true,   "retrieve": true }
      ]
    }
  ]
}
```

The generator emits a strongly-typed `ProductsDocument` record. If
`searchSettings.semanticRanking.enabled` is true, the SDK automatically routes calls
to the `/semantic-search` endpoint.

## Visitor identity — how the SDK tracks who's who

This SDK mirrors the Sitecore JS SDK's visitor-identity model so that the same browser
visitor is tracked under the same identifier regardless of which SDK fires the request.

### It works out of the box — no consumer code required

`AddSitecoreDiscoverSearchSdk(...)` automatically registers `DefaultDiscoverSessionProvider`
as the implementation of `IDiscoverSessionProvider`. It handles **all** visitor identity
plumbing for you:

- **Reads the `__ruid` cookie** from the active `HttpContext` request.
- **Generates a fresh `__ruid` value** in the exact JS-SDK format if absent, and writes it
  to the response with appropriate cookie flags (`SameSite=Lax`, `Secure` on HTTPS,
  `HttpOnly=false` so the JS SDK can read it back, 1-year expiry).
- **Surfaces a known-user override** when `Options.UserId` is configured (more on this below).
- **Falls back gracefully** when no `HttpContext` is available (background workers, etc.) by
  generating a per-call id rather than throwing.

For typical anonymous-tracking use you don't need to know any of this exists — the SDK
just does the right thing on every request.

### Why the cookie value looks the way it does

The `__ruid` value uses the exact format the Sitecore Discover JS SDK produces in
`@sitecore-discover/data/api/siteInfo.js :: getDomainUid()`:

```
{domainHash}-{xx}-{xx}-{4x}-{1p}-{20 base-36 chars}-{TIME_ms}
```

Example: `2222222-a3-b7-4f-1p-3kw7q9hd5x2p8nf1v6yz-1735689600000`

The format isn't arbitrary — each part carries meaning that downstream Discover services
read:

| Part | Source | Meaning |
| - | - | - |
| `domainHash` | the right half of your `CustomerKey` | Identifies the Discover domain this id belongs to. |
| `xx-xx-4x-1p` | random base-36 + literal device marker | Type 4 UUID variant marker (`4`) plus device-class hint (`p` = "pc", because every request from this server-side SDK is a non-mobile origin from Discover's perspective). |
| 20-char block | five concatenated `Math.random()` segments | The actual entropy — uniquely identifies this visitor. |
| `TIME_ms` | `Date.now()` at cookie creation | Lets Discover infer first-seen time without a separate field. |

**Why match the JS format byte-for-byte?** Because a Discover deployment could run
*both* the JS SDK (on the browser, for interactive widgets) *and* this server-side SDK
(for SSR / API) across more than one site. If our cookie didn't match the JS shape, the
JS SDK would either reject it (treating the visitor as anonymous) or overwrite it
(splitting the visitor's history across two ids). Either way, personalisation breaks.
By emitting the exact JS format, we ensure a single visitor tracks under a single id
whether the request originated from the browser or from our server.

### Known-user tracking — three options

When the user is authenticated, Discover prefers `user.user_id` (the consumer's system
identifier) over `user.uuid` for personalisation. There are three ways to surface this:

#### Option 1 — sticky single-user override

For CLI tools, admin pages, or background workers where the SDK instance serves a single
known user across its lifetime. Set `Options.UserId` in your IoC config; every request
carries that value as `user.user_id`.

```csharp
builder.Services.AddSitecoreDiscoverSearchSdk( opt => {
    builder.Configuration.GetSection( "SitecoreSearch:Discover" ).Bind( opt );
    opt.UserId = "some-permanent-system-user-id";
});
```

#### Option 2 — per-circuit Blazor authentication (the typical web-app pattern)

When the user identity varies per request / per circuit — e.g. a Blazor SSR app where each
visitor is a different authenticated principal — **inherit from `DefaultDiscoverSessionProvider`**
and override **only** `GetCurrentUserId()` to read from `AuthenticationStateProvider`:

```csharp
internal sealed class AuthAwareDiscoverSessionProvider( 
    IHttpContextAccessor http,
    IOptions<DiscoverSearchSdkOptions> opts,
    AuthenticationStateProvider auth
) : DefaultDiscoverSessionProvider( http, opts )
{
    public override string? GetCurrentUserId()
    {
        var task = auth.GetAuthenticationStateAsync();
        if ( !task.IsCompletedSuccessfully )
            return base.GetCurrentUserId();

        var user = task.Result.User;
        return user.Identity?.IsAuthenticated == true
            ? user.FindFirst( ClaimTypes.NameIdentifier )?.Value
            : base.GetCurrentUserId();
    }
}

// In Program.cs, AFTER AddSitecoreDiscoverSearchSdk:
builder.Services.AddScoped<IDiscoverSessionProvider, AuthAwareDiscoverSessionProvider>();
```

The SDK registers `DefaultDiscoverSessionProvider` via `TryAddScoped`, so this subsequent
`AddScoped<IDiscoverSessionProvider, ...>` call wins for resolution.

> [!NOTE]
> **Inherit, don't reimplement.** `IDiscoverSessionProvider` is a small interface, but the
> *default implementation* is doing nontrivial work: JS-SDK-compatible cookie generation,
> read/write through `IHttpContextAccessor` with the correct cookie flags, fallback when
> no `HttpContext` is available, and base-36 random generation tuned to Discover's parser.
> If you implement `IDiscoverSessionProvider` from scratch you have to reproduce all of that
> correctly, and any drift from the JS format silently breaks cross-SDK visitor continuity.
> Inheriting from `DefaultDiscoverSessionProvider` keeps you on the well-tested path — you
> only override the one method whose behaviour you actually want to change.
>
> In nearly all cases that one method is `GetCurrentUserId()`. Don't override
> `GetOrCreateAnonymousId()` unless you have a very specific reason — get it wrong and the
> JS SDK on the same page will treat your visitors as strangers.

#### Option 3 — fire `UserLogin` at sign-in time

Leave the per-request flow on `uuid` and call `events.UserLogin(...)` once at login (see
the **Events** section below). Discover backfills all prior anonymous activity to the
now-known user server-side; subsequent traffic continues to send `uuid`, but Discover knows
who's behind it.

This is the lowest-friction option when you don't need the user id present on every API
call — and it's the one the JS SDK starter kit demonstrates by default.

### `AuthenticationStateChanged` integration

If your Blazor app fires `AuthenticationStateChanged` (login / logout), wire it to a
hosted service that fires the appropriate identity events:

```csharp
public sealed class DiscoverAuthBridge( 
    AuthenticationStateProvider authProvider,
    ISitecoreDiscoverEventsSdk events
) : IDisposable
{
    public DiscoverAuthBridge()
    {
        authProvider.AuthenticationStateChanged += OnChanged;
    }

    private async void OnChanged( Task<AuthenticationState> stateTask )
    {
        var state = await stateTask;

        if ( state.User.Identity?.IsAuthenticated == true )
        {
            _ = events.UserLogin( new DiscoverUserInfo
            {
                Id    = state.User.FindFirst( ClaimTypes.NameIdentifier )!.Value,
                Email = state.User.FindFirst( ClaimTypes.Email )?.Value
            } );
        }
        else
        {
            _ = events.UserLogout();
        }
    }

    public void Dispose() => authProvider.AuthenticationStateChanged -= OnChanged;
}
```

## Events consent — gating events on per-request cookie consent

Every call on `ISitecoreDiscoverEventsSdk` — explicit and auto-fired `view_widget` alike —
is gated by `IDiscoverSessionProvider.HasEventsConsent()`. The default implementation
checks the `CheckEventsConsent` callback on `DiscoverSearchSdkOptions`; if unset it returns
`true` (events flow freely). To gate on a cookie-consent signal (GDPR, CCPA, etc.), supply
the callback at registration time — no subclass required.

### Option A — `CheckEventsConsent` callback (recommended)

The callback receives the scoped `IServiceProvider` of the current request / Blazor circuit
(the default session provider is itself Scoped), so it can resolve any registered service —
including Scoped ones — at consent-check time. Set it inline at registration:

```csharp
builder.Services.AddSitecoreDiscoverSearchSdk( o => {
    o.CustomerKey        = "11111-2222222";
    o.ApiKey             = builder.Configuration["Discover:ApiKey"];
    o.CheckEventsConsent = _ => MyConsentService.HasConsent;   // your singleton flag
} );
```

For a per-request signal — typically a cookie read through `IHttpContextAccessor` —
resolve it from the scoped provider passed to the callback:

```csharp
builder.Services.AddSitecoreDiscoverSearchSdk( o => {
    o.CustomerKey = "11111-2222222";
    o.ApiKey      = builder.Configuration["Discover:ApiKey"];
    // ... endpoints ...

    o.CheckEventsConsent = ioc => {
        var ctx = ioc.GetRequiredService<IHttpContextAccessor>().HttpContext;
        // No HTTP context (background worker): default to no consent.
        return ctx is not null
            && ctx.Request.Cookies.TryGetValue( "pw#eventConsent", out var v )
            && v == "1";
    };
} );
```

### Bridging from BlazorSDK's `ISitecoreTracking`

If you're using `PINGWorks.SitecoreAI.BlazorSDK`, the per-circuit `ISitecoreTracking`
service already snapshots the consent state from your cookie banner into
`Context.EventsEnabled` (it runs the cookie-check + per-circuit handler internally). Just
forward it:

```csharp
builder.Services.AddSitecoreDiscoverSearchSdk( o => {
    o.CustomerKey = "11111-2222222";
    o.ApiKey      = builder.Configuration["Discover:ApiKey"];
    // ... endpoints ...

    // ISitecoreTracking is registered Scoped by AddSitecoreBlazorSDK, so resolve it per call.
    // Context is null on the first request before SetTrackingFromContext has run — treat that
    // as no consent.
    o.CheckEventsConsent = ioc =>
        ioc.GetRequiredService<ISitecoreTracking>().Context?.EventsEnabled ?? false;
} );
```

This is preferred over reading `IOptions<SitecoreAnalyticsSettings>` + the cookie directly:
`ISitecoreTracking` already does the cookie read once per circuit, owns the
"is analytics enabled at all" gate, and is the canonical source of truth for whether the
current circuit is allowed to fire events.

When consent isn't granted, `Publish()` short-circuits with `HttpStatusCode.NoContent`;
the SDK returns immediately without making any HTTP call to Discover.

> **What's gated.** Only the **events** endpoint — the analytic side of Discover. The visitor's
> anonymous id (`__ruid` cookie write) and the personalised search responses themselves still
> flow; what gets suppressed is just the model-training signal Discover would otherwise receive.
> If you also want to suppress personalisation entirely, gate the call sites — don't call
> `Search` / `Recommend` until consent is granted.

### Option B — Subclass `DefaultDiscoverSessionProvider`

The callback above is sufficient for consent on its own. Reach for a custom provider only
when you need to override **multiple** pieces of session behaviour together — most commonly
when you want auth-aware user resolution (`GetCurrentUserId`) at the same time as consent.
The two concerns share the same provider so they compose naturally:

```csharp
internal sealed class ConsentAndAuthAwareDiscoverSessionProvider(
    IHttpContextAccessor http,
    IOptions<DiscoverSearchSdkOptions> opts,
    IOptions<SitecoreAnalyticsSettings> analytics,
    AuthenticationStateProvider auth
) : DefaultDiscoverSessionProvider( http, opts )
{
    public override string? GetCurrentUserId()
    {
        var task = auth.GetAuthenticationStateAsync();
        if ( !task.IsCompletedSuccessfully )
            return base.GetCurrentUserId();

        var user = task.Result.User;
        return user.Identity?.IsAuthenticated == true
            ? user.FindFirst( ClaimTypes.NameIdentifier )?.Value
            : base.GetCurrentUserId();
    }

    public override bool HasEventsConsent()
    {
        // ... same cookie check as above ...
    }
}
```

## Usage

### Search

```csharp
public class ProductCatalog( ISitecoreDiscoverSearchSdk search )
{
    public async Task<DiscoverSearchResult<ProductsDocument>?> Find( string query )
    {
        var response = await search.Search<ProductsDocument>(
            query,
            new SearchQuery<ProductsDocument>
            {
                Limit = 20,
                Filters = [ new SearchFilter( ProductsDocument.Fields.Filterable.InStock, "eq", "true" ) ],
                Sort    = [ new SearchSort( ProductsDocument.Fields.Sortable.Price, SortDirection.Asc ) ]
            }
        );

        if ( !response.IsSuccessful )
            return null;

        // Server-side spell correction? Surface a "Showing results for X" UI message.
        if ( response.Result is { Keyphrase: var k, OriginalKeyphrase: var o } && k != o && !string.IsNullOrEmpty( o ) )
            Logger.LogInformation( "Discover corrected '{Original}' to '{Corrected}'", o, k );

        // Configured search redirect? Issue an HTTP redirect instead of rendering.
        if ( !string.IsNullOrEmpty( response.Result?.RedirectUrl ) )
            return null;

        return response.Result;
    }
}
```

`DiscoverSearchResult<TDoc>` extends `SearchResult<TDoc>` with widget envelope metadata
returned by the Discover Search API: `RfkId`, `Entity`, `WidgetType`, `Keyphrase`,
`OriginalKeyphrase`, `RedirectUrl`, `AvailableSorts`, `Errors`.

### Batched search (multi-widget pages)

When one page needs results from several widgets — main results plus "trending in category"
plus "recently viewed" — use `SearchBatch` to pack them into one HTTP call. Cache hits are
served from cache without going over the wire; only cache-miss widgets go in the batch.
`view_widget` fires for every widget in the batch (cache hits included).

```csharp
var batchResp = await search.SearchBatch( b => b
    .Add<ProductsDocument>( categoryId, new SearchQuery<ProductsDocument> { Limit = 24 } )
    .Add<TrendingDocument>( categoryId )
    .Add<RecentlyViewedDocument>( "" )
);

if ( batchResp.IsSuccessful )
{
    var main = batchResp.Result?.Get<ProductsDocument>();
    var trending = batchResp.Result?.Get<TrendingDocument>();
}
```

### Recommendations

Recommendation widgets take no keyphrase — they personalise from the visitor profile
plus optional **page context** and **anchoring document IDs**. The shape of the response
is identical to search.

```csharp
var related = await search.Recommend<RelatedProductsDocument>( new RecommendationContext
{
    PageUri     = pageUri,
    DocumentIds = new Dictionary<string, IReadOnlyList<string>>
    {
        [ "product" ] = new[] { productSku }
    },
    Limit       = 6
} );
```

Pure visitor-profile recommendation (no context):

```csharp
var picks = await search.Recommend<HomepagePicksDocument>();
```

#### Recipe override

Override the widget's configured ML strategy (find the recipe ID/version in CEC under
**Global Resources → Recipes**):

```csharp
var picks = await search.Recommend<HomepagePicksDocument>( new RecommendationContext
{
    RecipeOverride = new DiscoverRecipeOverride( "548980", version: 1 )
} );
```

#### Rule override (boost / bury / blacklist)

Apply custom ranking rules on top of those configured against the widget. Use the
`DiscoverRecommendationRule` factory helpers:

```csharp
var rule = DiscoverRecommendationRule.Of(
    DiscoverRecommendationRule.Boost( 
        new SearchFilter( "brand", "eq", "premium" ), 
        weight: 3.0 ),
    DiscoverRecommendationRule.Blacklist( 
        new SearchFilter( "in_stock", "eq", "false" ) )
);

var picks = await search.Recommend<HomepagePicksDocument>( new RecommendationContext
{
    RuleOverride = rule
} );
```

#### Important: recommendations are not cached

Unlike `Search`, `Recommend` calls **always go over the wire**. The model re-personalises
every call against the live visitor profile and current page context — caching would defeat
the engine.

#### Critical: click attribution drives recommendation quality

The recommendation model's training signal is **the difference between "shown" and "clicked"**.
The SDK auto-fires `view_widget` for you on every `Recommend` and `Search` call, but
**`click_widget` is your responsibility** — only your UI knows when a user actually clicks
a recommended item.

> **If you skip click attribution, recommendation quality degrades.** The model can't tell
> "shown but ignored" from "shown and engaged with", so personalisation drifts toward the
> average user instead of toward this user. This matters more for `Recommend` than for
> `Search` (where the keyphrase already tells the engine what the user wanted).

The Blazor click-handler pattern:

```razor
@inject ISitecoreDiscoverEventsSdk Events

<ul class="recommendation-rail">
    @foreach ( var product in Recommendations.Items )
    {
        <li>
            <a href="@product.ProductUrl" @onclick="@(() => RecordClick(product))">
                <img src="@product.ImageUrl" alt="@product.Title" />
                <span>@product.Title</span>
            </a>
        </li>
    }
</ul>

@code {
    [Parameter] public DiscoverSearchResult<RelatedProductsDocument> Recommendations { get; set; } = null!;

    private void RecordClick( RelatedProductsDocument product )
    {
        // Fire-and-forget — SafeSend swallows all exceptions internally.
        _ = Events.ClickWidget( Recommendations.RfkId, product.ProductId! );
    }
}
```

Three things to call out in that snippet:

1. **Use `Recommendations.RfkId`, not a hard-coded string.** The widget envelope returns the
   `rfk_id` that produced the result.
2. **Pass the item's document key as the second argument** (the `key`-flagged field).
3. **Discard the task with `_ =`** — never `await` before navigation.

### Questions (AI Q&A)

For widgets configured with a `questions` strategy in CEC:

```csharp
var qa = await search.Questions<FaqDocument>( limit: 5, includeSources: true );

if ( qa.IsSuccessful )
    foreach ( var item in qa.Result!.Items )
        Console.WriteLine( $"{item.Question}: {item.Answer}" );
```

### Suggestions (type-ahead / autocomplete)

For widgets configured with one or more suggester blocks in CEC. Pass the partial keyphrase
the visitor has typed and the suggester block name(s) you want to invoke:

```csharp
public class SearchBox( ISitecoreDiscoverSearchSdk search )
{
    public async Task<IReadOnlyList<DiscoverSuggestion>> Complete( string partial )
    {
        var resp = await search.Suggest<ProductsDocument>( partial, new SuggestOptions
        {
            SuggesterNames    = new[] { "product_titles" },
            Max               = 10,
            KeyphraseFallback = true
        } );

        if ( !resp.IsSuccessful )
            return Array.Empty<DiscoverSuggestion>();

        // Pull suggestions for the suggester we asked about.
        return resp.Result!.Suggestions.TryGetValue( "product_titles", out var values )
            ? values
            : Array.Empty<DiscoverSuggestion>();
    }
}
```

Requesting multiple suggesters in one call (e.g. product names AND category names):

```csharp
var resp = await search.Suggest<ProductsDocument>( partial, new SuggestOptions
{
    SuggesterNames = new[] { "product_titles", "category_names" },
    Max            = 5
} );

var titles    = resp.Result!.Suggestions["product_titles"];
var categories = resp.Result!.Suggestions["category_names"];
```

**Suggestions are not cached.** The partial keyphrase changes on every keystroke; caching
would only delay UX without helping. Debounce the call on the consumer side instead — most
type-ahead inputs use a 150–300ms debounce.

### Batched search + recommendation (mixed)

`SearchBatch` accepts both search and recommendation items in the same call via
`AddRecommendation<TDoc>()`. The SDK constructs the right wire clause per item, dispatches
one HTTP call (or two if the batch mixes semantic-search and standard endpoints), and
demuxes the results by document type. **Search items participate in the cache; recommend
items always dispatch live** so personalisation stays fresh.

```csharp
var batchResp = await search.SearchBatch( b => b
    .Add<ProductsDocument>( categoryId, new SearchQuery<ProductsDocument> { Limit = 24 } )
    .AddRecommendation<TrendingDocument>( new RecommendationContext
    {
        PageUri = "/category/" + categoryId,
        Limit   = 6
    } )
    .AddRecommendation<RecentlyViewedDocument>()
);

if ( batchResp.IsSuccessful )
{
    var main       = batchResp.Result?.Get<ProductsDocument>();
    var trending   = batchResp.Result?.Get<TrendingDocument>();
    var recent     = batchResp.Result?.Get<RecentlyViewedDocument>();
}
```

> **Shared context constraint.** The Discover wire format carries one `context` object per
> request, so all recommendation items in a single batch share the same `PageUri` and
> `DocumentIds`. The SDK takes them from the FIRST recommendation item added. If you need
> different page contexts per recommendation, issue them as separate batches.

### Events

`view_widget` fires automatically after every successful `Search`, `SearchBatch`, and
`Recommend` call. Every other event is the consumer's responsibility — fire them when the
corresponding user interaction or business event happens. Each typed method is a thin
wrapper around the catch-all `Publish` method.

```csharp
public class CommerceFlow( ISitecoreDiscoverEventsSdk events )
{
    public Task OnAddedToCart( DiscoverCartItem item )
        => events.AddToCart( new[] { item } );

    public Task OnCheckoutStarted( IReadOnlyList<DiscoverCartItem> cart )
        => events.CheckoutStart( cart );

    public Task OnOrderConfirmed( Order order )
        => events.Purchase(
            orderId: order.Id,
            items: order.Items.Select( i => new DiscoverCartItem
            {
                Sku      = i.Sku,
                Quantity = i.Quantity,
                Price    = i.Price,
                Currency = "AUD"
            } ).ToList(),
            total: order.Total,
            currency: "AUD"
        );
}

public class IdentityFlow( ISitecoreDiscoverEventsSdk events )
{
    public Task OnUserLoggedIn( ClaimsPrincipal user )
        => events.UserLogin( new DiscoverUserInfo
        {
            Id    = user.FindFirst( ClaimTypes.NameIdentifier )!.Value,
            Email = user.FindFirst( ClaimTypes.Email )?.Value
        } );

    public Task OnUserLoggedOut()
        => events.UserLogout();
}
```

### Ingestion

```csharp
public class CatalogSync( ISitecoreDiscoverIngestionSdk ingestion )
{
    public async Task<string> Upsert( ProductsDocument product )
    {
        var resp = await ingestion.Upsert( "products", product.ProductId!, product );

        if ( !resp.IsSuccessful )
            throw new InvalidOperationException( $"Ingestion failed: {resp.Status}" );

        return resp.Result!.IncrementalUpdateId;
    }
}
```

## Available services

### `ISitecoreDiscoverSearchSdk`

| Method | Description |
| - | - |
| `Search<TDoc>( query, options?, ct? )` | Single-widget keyphrase search. Returns `ApiResponse<DiscoverSearchResult<TDoc>>`. Auto-fires `view_widget`. |
| `SearchBatch( configure, ct? )` | Multi-widget batched search and/or recommendation. Cache-aware for search items; recommend items always dispatch live. Returns `ApiResponse<DiscoverSearchBatch>` — use `Get<TDoc>()` per widget. |
| `Recommend<TDoc>( context?, ct? )` | Recommendation widget (no keyphrase). **Not cached**. Supports recipe + rule overrides. |
| `Questions<TDoc>( limit?, includeSources?, ct? )` | AI Q&A widget. |
| `Suggest<TDoc>( partial, options, ct? )` | Type-ahead / autocomplete. **Not cached** — debounce on the consumer side. Returns `ApiResponse<DiscoverSuggestResult>` keyed by suggester name. |

### `ISitecoreDiscoverEventsSdk`

Widget events:

| Method | Description |
| - | - |
| `ViewWidget( rfkId, ct? )` | Auto-fired by Search/Recommend; manual fire is rarely needed. |
| `ClickWidget( rfkId, itemId, ct? )` | **Essential for recommendation training.** |

User identity events:

| Method | Description |
| - | - |
| `UserLogin( user, ct? )` | **Critical for cross-device personalisation** — links anonymous `__ruid` to a known user id. Fire ONCE at login. |
| `UserRegister( user, ct? )` | Typically followed by `UserLogin` in the same flow. |
| `UserLogout( ct? )` | Reverts subsequent traffic to anonymous. |

Commerce events:

| Method | Description |
| - | - |
| `AddToCart( items, ct? )` | Strong personalisation signal. |
| `RemoveFromCart( items, ct? )` | Negative signal. |
| `CartView( items, ct? )` | Cart-abandonment modelling. |
| `CheckoutStart( items, ct? )` |  |
| `Purchase( orderId, items, total?, currency?, ct? )` | **Strongest possible signal** for cross-sell / repeat-purchase models. |

Navigation:

| Method | Description |
| - | - |
| `Pageview( pageUri, ct? )` | Rarely needed — page context is already carried via `RecommendationContext.PageUri`. |

Catch-all:

| Method | Description |
| - | - |
| `Publish( events, ct? )` | Submit any batch of typed `DiscoverEvent`s — useful for custom event types not modelled above. |

### `ISitecoreDiscoverIngestionSdk`

| Method | Description |
| - | - |
| `Upsert<TDoc>( sourceId, documentId, document, ct? )` |  |
| `BulkUpsert<TDoc>( sourceId, documents, ct? )` | Multiple documents per call. |
| `Delete( sourceId, documentId, ct? )` |  |
| `GetStatus( sourceId, incrementalUpdateId, ct? )` | Poll a job's progress. |

### Pluggable infrastructure

- **`IDiscoverSessionProvider`** — controls the `__ruid` cookie and the `user_id` resolution. Replace with a custom implementation to integrate with `AuthenticationStateProvider` (see "Visitor identity" above).

## Known issues

None.

## Related libraries

- [`PINGWorks.SitecoreSearch.Common`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.Common/) — shared abstractions and source generator (transitive dependency of this package)
- [`PINGWorks.SitecoreSearch.ExperienceSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.ExperienceSearch/) — sibling SDK for Sitecore ExperienceSearch
