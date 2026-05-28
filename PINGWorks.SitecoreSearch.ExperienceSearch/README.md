# PINGWorks.SitecoreSearch.ExperienceSearch

A .NET SDK for working with **Sitecore ExperienceSearch** (the search system bundled with
Sitecore XM Cloud, hosted at `edge-platform.sitecorecloud.io`).

For an overview of ExperienceSearch, see the [official documentation](https://doc.sitecore.com/sai/en/developers/sitecoreai/experience-search.html).

This package coexists cleanly with [`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/)
— both can be registered in the same DI container without conflict. Each SDK registers
only its own contracts, so neither overwrites the other.

## Features

- Typed, DI-friendly client for the ExperienceSearch endpoint
- Async-first APIs with `CancellationToken` support
- `netstandard2.1` for wide compatibility
- Standard resilience handler (retries, circuit breaker, timeouts) via `Microsoft.Extensions.Http.Resilience`
- In-memory response cache (5-minute default)
- Strongly-typed document types from the source generator shipped in `PINGWorks.SitecoreSearch.Common`
- Single DI entry point: `AddSitecoreExperienceSearchSdk( opt => ... )`

## Getting started

### Prerequisites

You will need:

- An **XM Cloud** environment with at least one search source configured in the Content Editing Console (CEC)
- A **Search Edge API key** for that environment, available from the [SitecoreAI Deployment Portal](https://deploy.sitecorecloud.io/credentials/environment)
- The environment's **context id** (also known as `EdgeContextId`)

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
      /* Base URL — defaults to https://edge-platform.sitecorecloud.io */
      "ServiceEndpoint": "https://edge-platform.sitecorecloud.io",

      /* Secret. The Search Edge API key for your XM Cloud environment. */
      "ApiKey": "[Created in the SitecoreAI Deploy portal's Credentials pane]",

      /* Secret. The environment context id, sent as the x-sitecore-contextid header. */
      "EdgeContextId": "[From SitecoreAI Deploy > Projects > Environment > Developer Settings]"
    }
  }
}
```

We recommend Visual Studio's **User Secrets** for `ApiKey` / `EdgeContextId` during
development, and your platform's secret store (Azure Key Vault, AWS Secrets Manager, etc.)
in production.

### Register services

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder( args );

builder.Services.AddSitecoreExperienceSearchSdk( opt =>
    builder.Configuration.GetSection( "SitecoreSearch:ExperienceSearch" ).Bind( opt )
);
```

> **Fail-fast validation.** `AddSitecoreExperienceSearchSdk` synchronously snapshots the
> options you configure and throws `InvalidOperationException` immediately if any required
> values are missing — `ApiKey` and `EdgeContextId`. Misconfiguration surfaces at app
> startup, not at the first failed search call.

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

### Author `sitecore-search.json`

Drop a `sitecore-search.json` file at the project root describing each search source you
want to use. The generator (shipped by `PINGWorks.SitecoreSearch.Common`) picks it up
at build time and emits one strongly-typed document record per source. See the
[`PINGWorks.SitecoreSearch.Common` README](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.Common/)
for full schema details.

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
        { "name": "Date",            "type": "datetime", "filter": true, "facet": true, "retrieve": true }
      ]
    }
  ]
}
```

## Usage

Inject `ISitecoreExperienceSearchSdk` wherever you need it. Every call returns
`ApiResponse<T>` (the same wrapper used by the rest of the PING Works SDK family) —
check `IsSuccessful` before reading `Result`. The SDK never throws for HTTP / network
errors; everything is captured on the response.

```csharp
public class BlogSearchPage( ISitecoreExperienceSearchSdk search )
{
    public async Task<SearchResult<BlogsDocument>?> Find( string query, int page = 0 )
    {
        var response = await search.Search<BlogsDocument>(
            query,
            new SearchQuery<BlogsDocument>
            {
                Limit = 10,
                Offset = page * 10,
                Filters = [
                    new SearchFilter( BlogsDocument.Fields.Filterable.Category, "eq", "engineering" )
                ],
                Facets = [ BlogsDocument.Fields.Facetable.Category, BlogsDocument.Fields.Facetable.Date ]
            }
        );

        if ( !response.IsSuccessful )
        {
            Logger.LogWarning( "Search failed: {Status} {Message}", response.Status, response.Error?.Message );
            return null;
        }

        return response.Result;
    }
}
```

Iterate over `response.Result.Items` for hits, `response.Result.Total` for the count,
and `response.Result.Facets` for facet groups.

## Available services

### Injectable interface `ISitecoreExperienceSearchSdk`

| Method | Description |
| - | - |
| `Search<TDoc>( query, options?, ct? )` | Execute a search against the source identified by `TDoc`. Returns `ApiResponse<SearchResult<TDoc>>` — the `Result` carries items, total count, and facet groups. |

## Related libraries

- [`PINGWorks.SitecoreSearch.Common`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.Common/) — shared abstractions and source generator (transitive dependency of this package)
- [`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/) — sibling SDK for Sitecore Discover (Search + Events + Ingestion)

## Known issues

None.
