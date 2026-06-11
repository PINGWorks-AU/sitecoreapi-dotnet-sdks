# PINGWorks.SitecoreSearch.ExperienceSearch

A .NET SDK for working with Sitecore ExperienceSearch — the search system bundled with
Sitecore AI, hosted at `edge-platform.sitecorecloud.io`.

For an overview of ExperienceSearch, see the [official documentation](https://doc.sitecore.com/sai/en/developers/sitecoreai/experience-search.html).

This package coexists cleanly with [`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/)
— both can be registered in the same DI container without conflict. Each SDK registers
only its own contracts, so neither overwrites the other.

## Features

- Typed, DI-friendly client for the ExperienceSearch endpoint (`edge-platform.sitecorecloud.io`)
- Async-first APIs with `CancellationToken` support
- `netstandard2.1` for wide compatibility
- Standard resilience handler (retries, circuit breaker, timeouts) via `Microsoft.Extensions.Http.Resilience`
- Strongly-typed document types from the source generator shipped in `PINGWorks.SitecoreSearch.Common`
- Single DI entry point: `AddSitecoreExperienceSearchSdk( opt => ... )`
- Automatic `x-sitecore-contextid` header — the SDK sends it on every request; consumers never set it

## Getting started

### Prerequisites

You will need:

- An SAI environment with at least one search source configured in the Sitecore Content
  Editing Console (CEC)
- The environment's context id (`EdgeContextId`) — find it in SitecoreAI Deploy →
  Projects → [Environment] → Developer Settings. This is the only credential the Search
  endpoint requires; it is transmitted automatically as the `x-sitecore-contextid` request
  header. There is no separate API key or bearer token.

### Install

```bash
dotnet add package PINGWorks.SitecoreSearch.ExperienceSearch
```

`PINGWorks.SitecoreSearch.Common` (which carries the source generator) is brought in
automatically as a transitive dependency.

### AppSettings

The SDK is configured through a strongly-typed `ExperienceSearchSdkOptions` POCO.
Bind it from configuration however you like — there is no required section name.

```json
{
  "SitecoreSearch": {
    "ExperienceSearch": {
      "ServiceEndpoint": "https://edge-platform.sitecorecloud.io",
      "EdgeContextId":   "[From SitecoreAI Deploy > Projects > Environment > Developer Settings]"
    }
  }
}
```

| Key | Default | Description |
| - | - | - |
| `ServiceEndpoint` | `https://edge-platform.sitecorecloud.io` | Base URL for the ExperienceSearch API. Override only if Sitecore provides a regional URL. |
| `EdgeContextId` | _(none — required)_ | Environment context id. |

We recommend Visual Studio's User Secrets for `EdgeContextId` during development, and your
platform's secret store (Azure Key Vault, AWS Secrets Manager, etc.) in production.

### Register services

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder( args );

builder.Services.AddSitecoreExperienceSearchSdk( opt =>
    builder.Configuration.GetSection( "SitecoreSearch:ExperienceSearch" ).Bind( opt )
);
```

The `x-sitecore-contextid` header is added automatically on every request from the value
in `EdgeContextId`. No additional HTTP client configuration is needed.

> [!NOTE]
> Fail-fast validation. `AddSitecoreExperienceSearchSdk` synchronously snapshots the
> options you configure and throws `InvalidOperationException` immediately if `EdgeContextId`
> is missing or if `ServiceEndpoint` is null. Misconfiguration surfaces at app startup, not
> at the first failed search call.

> [!TIP]
> HttpClient defaults. This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to every HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

### Author `sitecore-search.json`

Drop a `sitecore-search.json` file at the project root describing each search source you
want to use. The generator picks it up at
build time and emits one strongly-typed document record per source. See the
[`PINGWorks.SitecoreSearch.Common` README](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.Common/)
for the full schema.

The source `id` and field names come from the SitecoreAI portal — navigate to Content →
your search index. Copy the source GUID and each attribute name verbatim; ExperienceSearch
attributes are typically PascalCase (`PageTitle`, `MetaDescription`).

```json
{
  "experienceSources": [
    {
      "id": "8c5e839b-997b-4fdd-8c5a-5e638a4f9bbe",
      "name": "Blogs",
      "fields": [
        { "name": "PageTitle",       "type": "string",   "search": true, "retrieve": true },
        { "name": "PageContentText", "type": "string",   "search": true, "retrieve": true },
        { "name": "Category",        "type": "string",   "filter": true, "facet": true, "retrieve": true },
        { "name": "Date",            "type": "datetime", "filter": true, "facet": true, "retrieve": true, "sort": true }
      ]
    }
  ]
}
```

The generator emits a `BlogsDocument` record with typed properties for every `retrieve: true`
field plus nested `Fields.Searchable`, `Fields.Filterable`, `Fields.Sortable` constant classes
for use at call sites.

## Usage

Inject `ISitecoreExperienceSearchSdk` wherever you need it. Every call returns
`ApiResponse<SearchResult<TDoc>>` — check `IsSuccessful` before reading `Result`. The SDK
never throws for HTTP or network errors; everything is captured on the response object.

### Search

```csharp
public class BlogSearchPage( ISitecoreExperienceSearchSdk search, ILogger<BlogSearchPage> logger )
{
    public async Task<SearchResult<BlogsDocument>?> Find( string query, int page = 0, CancellationToken ct = default )
    {
        var response = await search.Search<BlogsDocument>(
            query,
            new SearchQuery<BlogsDocument>
            {
                Limit  = 10,
                Offset = page * 10,
                Sort   = [ new SearchSort( BlogsDocument.Fields.Sortable.Date, SortDirection.Desc ) ]
            },
            ct
        );

        if ( !response.IsSuccessful )
        {
            logger.LogWarning( "Search failed: {Status} {Title}", response.Status, response.Error?.Title );
            return null;
        }

        return response.Result;
    }
}
```

Iterate `response.Result.Items` for hits and read `response.Result.Total` for the total count
across all pages.

### Error handling

The SDK maps all HTTP and network failures to the `ApiResponse` envelope — no exceptions escape
for transport-layer problems. The pattern is the same everywhere:

```csharp
var response = await search.Search<BlogsDocument>( query );

if ( !response.IsSuccessful )
{
    // response.Status  — the HTTP status code (e.g. HttpStatusCode.Unauthorized)
    // response.Error   — structured error detail parsed from the response body
    // response.Error?.Title  — short human-readable summary
    // response.Error?.Detail — longer description of this specific occurrence
    logger.LogError(
        "ExperienceSearch returned {Status}: {Title} — {Detail}",
        response.Status,
        response.Error?.Title,
        response.Error?.Detail
    );
    return null;
}

foreach ( var item in response.Result!.Items )
    Console.WriteLine( item.PageTitle );

Console.WriteLine( $"Total: {response.Result.Total}" );
```

## Response type reference

### `ApiResponse<SearchResult<TDoc>>`

| Property | Type | Description |
| - | - | - |
| `IsSuccessful` | `bool` | `true` when `Status` is in the 2xx range (below 299). Check this before reading `Result`. |
| `Status` | `HttpStatusCode` | The HTTP status code returned by the API. |
| `Result` | `SearchResult<TDoc>?` | The search result payload. Non-null when `IsSuccessful` is `true`. |
| `Error` | `ErrorResponse?` | Structured error detail. Non-null when `IsSuccessful` is `false`. |
| `Headers` | `HttpResponseHeaders?` | Raw response headers from the HTTP call. |

### `SearchResult<TDoc>`

| Property | Type | Description |
| - | - | - |
| `Items` | `IReadOnlyList<TDoc>` | Hit documents for the current page. Empty list (never null) when there are no matches. |
| `Total` | `int` | Total matching documents across all pages. Use with `Limit` and `Offset` to implement pagination. |
| `Facets` | `IReadOnlyList<SearchFacetGroup>?` | Always `null` for ExperienceSearch — see Limitations below. |

### `ErrorResponse`

| Property | Type | Description |
| - | - | - |
| `Title` | `string?` | Short human-readable summary of the problem. |
| `Detail` | `string?` | Longer description of this specific occurrence. |
| `Status` | `HttpStatusCode` | The HTTP status as an enum value. |
| `Type` | `Uri?` | URI reference identifying the problem type. |
| `Instance` | `string?` | URI identifying the specific resource involved. |

## API reference

### `ISitecoreExperienceSearchSdk`

| Method | Description |
| - | - |
| `Search<TDoc>( query, options?, ct? )` | Execute a keyphrase search against the source identified by `TDoc`. Returns `ApiResponse<SearchResult<TDoc>>`. `options` is optional; `ct` defaults to `CancellationToken.None`. |

### `SearchQuery<TDoc>` — accepted options

| Property | Type | Sent to API | Notes |
| - | - | - | - |
| `Limit` | `int?` | Yes | Maximum results per page. |
| `Offset` | `int?` | Yes | Zero-based result offset for pagination. |
| `Sort` | `IReadOnlyList<SearchSort>?` | Yes | One or more sort directives applied in order. Use `TDoc.Fields.Sortable.*` constants for field names. |
| `Filters` | `IReadOnlyList<SearchFilter>?` | No | Ignored by ExperienceSearch — see Limitations. |
| `Facets` | `IReadOnlyList<string>?` | No | Ignored by ExperienceSearch — see Limitations. |
| `RetrieveFields` | `IReadOnlyList<string>?` | No | Ignored by ExperienceSearch — see Limitations. |

### Service lifetime

`ISitecoreExperienceSearchSdk` is registered as a typed HTTP client (transient via
`AddHttpClient<,>`). This keeps the underlying `HttpMessageHandler` rotation from
`IHttpClientFactory` working correctly.

Consuming from a singleton (e.g. a hosted background worker): use `IServiceScopeFactory` to
create a scope per operation rather than capturing the SDK directly.

```csharp
public class SearchIndexer( IServiceScopeFactory scopes ) : BackgroundService
{
    protected override async Task ExecuteAsync( CancellationToken ct )
    {
        while ( !ct.IsCancellationRequested )
        {
            using var scope = scopes.CreateScope();
            var search = scope.ServiceProvider.GetRequiredService<ISitecoreExperienceSearchSdk>();

            await search.Search<BlogsDocument>( "latest", ct: ct );
            await Task.Delay( TimeSpan.FromMinutes( 5 ), ct );
        }
    }
}
```

## Limitations

### Query options not supported by the ExperienceSearch endpoint

The shared `SearchQuery<TDoc>` type exposes `Filters`, `Facets`, and `RetrieveFields`
because those properties are used by the DiscoverSearch backend. ExperienceSearch's
`/v1/search` endpoint does not support them in the documented wire format — mapping them
produced 4xx errors in testing against a live tenant, so the SDK intentionally discards
them. As a consequence:

- Passing `Filters` to `Search<TDoc>` has no effect.
- Passing `Facets` to `Search<TDoc>` has no effect. `response.Result.Facets` is always
  `null`.
- Passing `RetrieveFields` to `Search<TDoc>` has no effect.

The ExperienceSearch endpoint accepts only `keyphrase`, `limit`, `offset`, and `sort`.
Field-level filtering and facet aggregation must be performed client-side on the returned
`Items`, or by configuring source-level faceting in the SitecoreAI portal.

### Fuzzy search and semantic ranking — unconfirmed

The `sitecore-search.json` schema (defined by `PINGWorks.SitecoreSearch.Common`) accepts
`searchSettings.fuzzySearch.enabled` and `searchSettings.semanticRanking.enabled` flags
on `experienceSources`. These are metadata and are provided for potential future use only:
the ExperienceSearch SDK does not send any flags to the `/v1/search` endpoint to enable or
disable fuzzy matching or semantic ranking. Whether the ExperienceSearch endpoint performs
fuzzy or semantic ranking is controlled entirely by the source configuration in the SitecoreAI
portal — the SDK has no mechanism to toggle it per request. The wire format does not expose
a per-request semantic or fuzzy flag for this backend.

If you require per-request semantic routing, use
[`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/)
which supports the `+semsearch` flag on the DiscoverSearch endpoint.

## Related libraries

- [`PINGWorks.SitecoreSearch.Common`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.Common/) — shared abstractions and source generator (transitive dependency of this package)
- [`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/) — sibling SDK for Sitecore Discover (Search + Events + Ingestion)
- [`PINGWorks.SitecoreExperienceEdge.Common`](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common/) — shared HTTP client infrastructure, resilience handler, and `ApiResponse` types
