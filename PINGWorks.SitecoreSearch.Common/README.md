# PINGWorks.SitecoreSearch.Common

Shared abstractions for the **PING Works Sitecore Search** family. Holds the document
marker interface, the routing attribute, and the query / result types consumed by both
the **ExperienceSearch** and **DiscoverSearch** SDKs.

Crucially, this package also **bundles the `PINGWorks.SitecoreSearch.CodeGenerators`
Roslyn source generator**. Any project that references this package — directly, or
transitively through one of the leaf SDKs — automatically picks up `sitecore-search.json`
at build time and emits strongly-typed document records.

## Features

- `ISitecoreSearchDocument` marker interface for typed documents
- `[SitecoreSearchSource]` attribute carrying source-id, optional `rfk_id`, and the semantic-ranking flag
- `SearchQuery<TDoc>` / `SearchResult<TDoc>` / `SearchFilter` / `SearchSort` / `SearchFacetGroup`
- `SearchDocumentInfo.For<TDoc>()` — cached attribute lookup used by the SDK clients
- **Build-time source generator** — reads `sitecore-search.json`, emits typed records with constants for searchable / filterable / retrievable / facetable / sortable field names
- `netstandard2.1` — wide compatibility with modern .NET runtimes

## Getting started

You normally do not install this package directly. Install one (or both) of the leaf SDKs
and `Common` comes along automatically:

```bash
dotnet add package PINGWorks.SitecoreSearch.ExperienceSearch
dotnet add package PINGWorks.SitecoreSearch.DiscoverSearch
```

If you want only the generator and shared types (e.g. for a "Contracts" project that
defines documents but does no I/O), you can install Common on its own:

```bash
dotnet add package PINGWorks.SitecoreSearch.Common
```

### Author `sitecore-search.json`

Drop a `sitecore-search.json` file at the root of your consumer project. The source
generator is auto-wired via the `build/PINGWorks.SitecoreSearch.Common.props` file
shipped in this package — no MSBuild changes needed.

The shape of the file is **a mirror of the search sources you have already configured
in the Sitecore portals**. The generator does not invent anything; it lets you _project_
those sources into typed C# code. You only describe sources you actually want to consume
from your code — everything else is invisible to the generator.

#### Where to find the source values

| Backend | Portal | Path |
| - | - | - |
| **ExperienceSearch** | **SitecoreAI portal** — `https://portal.sitecorecloud.io` | **Content** menu → your search index. Each source appears with its **ID** (a GUID) and **Name**, and lists every **attribute** that is searchable / filterable / retrievable / etc. |
| **DiscoverSearch** | **Sitecore Discover portal** — reach it from the Sitecore Product Navigation switcher in any Sitecore portal | The widget administration screens list each widget's **ID**, **rfk_id**, and the **attributes** defined on the source. |

Open the relevant portal for each source you want to use, and copy the values verbatim
into `sitecore-search.json`. The names you use **must match** the portal's wire-format
attribute names exactly — the generator does not transform them. ExperienceSearch
attributes are typically PascalCase (`PageTitle`, `MetaDescription`); DiscoverSearch
attributes are typically snake_case (`product_id`, `in_stock`).

#### You only configure what you use

Both top-level arrays are optional. Most projects use only one backend, so leave the
other array empty (or omit it entirely):

- Using only ExperienceSearch? Provide `experienceSources` and omit `discoverSources`.
- Using only DiscoverSearch? Provide `discoverSources` and omit `experienceSources`.
- Using both side-by-side? Provide both — each leaf SDK only sees its own documents.

Within a backend, you also only declare the sources you want typed access to. A search
index may have ten sources defined in the portal, but if you only call into three of
them from code, only describe those three. The generator skips anything you didn't
declare; it never reads back from the portal.

#### Field flags — what each one controls

For each `field` you can set up to six boolean flags. They mirror the capabilities you
already toggled in the portal:

| Flag | Meaning in the generated code |
| - | - |
| `search`   | The field can appear in the keyphrase / free-text query. Appears under `Fields.Searchable`. |
| `filter`   | The field can be used in a `SearchFilter`. Appears under `Fields.Filterable`. |
| `retrieve` | The field is returned in the response body. Appears as a typed property on the record _and_ under `Fields.Retrievable`. |
| `facet`    | The field can be requested as a facet aggregation. Appears under `Fields.Facetable`. |
| `sort`     | The field can be used in `SearchSort`. Appears under `Fields.Sortable`. |
| `key`      | The field is the document key. Appears under `Fields.Key`. Use this for the field that uniquely identifies the document (typically the same as the document id). |

A field marked `retrieve: true` is the most common: it gets both a typed property on
the document record and an entry under `Fields.Retrievable`. A field marked only
`filter: true` (no `retrieve`) only gets a name constant — the SDK uses it to build
the filter clause, but the value never comes back in the response body.

#### Discover sources: reused by Ingestion

For **DiscoverSearch**, the generated document records are used by **both** the search
client _and_ the ingestion client. The same `ProductsDocument` you receive from
`ISitecoreDiscoverSearchSdk.Search<ProductsDocument>(...)` is the type you pass to
`ISitecoreDiscoverIngestionSdk.Upsert("products", id, document)`. So make sure your
field definitions include every attribute you push to Discover — not just the ones you
read back from search. (`retrieve: true` is what gets the typed property generated; the
JSON serializer ignores fields that aren't on the type.)

#### Example

```json
{
  "experienceSources": [
    {
      "id": "8c5e839b-997b-4fdd-8c5a-5e638a4f9bbe",
      "name": "Blogs",
      "description": "Rich-text search against the Blog library",
      "searchClientKey": "8ccdd99fc31ca889e6872a17782b358c",
      "searchSettings": {
        "fuzzySearch":     { "enabled": true },
        "semanticRanking": { "enabled": true }
      },
      "fields": [
        { "name": "Category",        "type": "string",   "search": true,  "filter": true,  "retrieve": true, "facet": true  },
        { "name": "Date",            "type": "datetime", "filter": true,  "retrieve": true, "facet": true                  },
        { "name": "PageContentText", "type": "string",   "search": true,  "retrieve": true                                 },
        { "name": "PageTitle",       "type": "string",   "search": true,  "retrieve": true                                 }
      ]
    }
  ],
  "discoverSources": [
    {
      "id": "abcd1234-aaaa-bbbb-cccc-1111deadbeef",
      "name": "Products",
      "rfkId": "rfkid_products_search",
      "fields": [
        { "name": "product_id", "type": "string", "retrieve": true, "key": true },
        { "name": "title",      "type": "string", "search": true,   "retrieve": true },
        { "name": "price",      "type": "number", "filter": true,   "sort": true, "retrieve": true }
      ]
    }
  ]
}
```

### Generated output

The generator emits one `partial record` per source, placed in `<RootNamespace>.Search`:

```csharp
// <auto-generated/>
[SitecoreSearchSource( "8c5e839b-...", SemanticSearch = true )]
public partial record BlogsDocument : ISitecoreSearchDocument
{
    public static class Fields
    {
        public static class Searchable
        {
            public const string Category        = "Category";
            public const string PageContentText = "PageContentText";
            public const string PageTitle       = "PageTitle";
        }
        public static class Filterable { public const string Category = "Category"; public const string Date = "Date"; }
        public static class Retrievable { /* ... */ }
        public static class Facetable { /* ... */ }
        public static class Sortable { /* ... */ }
    }

    public string? Category { get; init; }
    public DateTimeOffset? Date { get; init; }
    public string? PageContentText { get; init; }
    public string? PageTitle { get; init; }
}
```

Call into the leaf SDK with the strongly-typed document:

```csharp
var result = await search.Search<BlogsDocument>(
    "sitecore ai",
    new SearchQuery<BlogsDocument>
    {
        Limit = 10,
        Filters = [ new SearchFilter( BlogsDocument.Fields.Filterable.Category, "eq", "engineering" ) ],
        Facets = [ BlogsDocument.Fields.Facetable.Category ]
    } );

foreach ( var doc in result.Items )
    Console.WriteLine( $"{doc.PageTitle} — {doc.Date:yyyy-MM-dd}" );
```

### Diagnostics

The generator emits the following diagnostics. All are surfaced through the standard
compiler diagnostic pipeline (IDE squiggles + build output).

| ID | Severity | Meaning |
| - | - | - |
| `SCSEARCH001` | Warning | `sitecore-search.json` is present but contains no sources. |
| `SCSEARCH002` | Warning | A source has no fields defined — a skeleton-only record is emitted. |
| `SCSEARCH003` | Error   | A source is missing `id` or `name` — the source is skipped. |
| `SCSEARCH004` | Warning | A field has an unrecognised `type` — emitted as `object?`. |
| `SCSEARCH005` | Error   | A DiscoverSearch source is missing `rfkId` — Discover calls will fail. |
| `SCSEARCH900` | Error   | `sitecore-search.json` could not be parsed (malformed JSON). |
| `SCSEARCH901` | Error   | `sitecore-search.json` could not be read from disk. |

### Customising the generated namespace

The generator places generated types in `$(RootNamespace).Search`. To override, set
`SearchGenRootNamespace` in your csproj:

```xml
<PropertyGroup>
    <SearchGenRootNamespace>MyCompany.Site.Search</SearchGenRootNamespace>
</PropertyGroup>
```

## Schema reference

The full JSON Schema for `sitecore-search.json` is at:

> `https://dl.polyglot-app.com/schemas/sitecore-search.schema.json`

The schema also ships in the source repository alongside this README at
`PINGWorks.SitecoreSearch.Common/sitecore-search.schema.json`.

## Related libraries

- [`PINGWorks.SitecoreSearch.ExperienceSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.ExperienceSearch/) — leaf SDK for Sitecore ExperienceSearch
- [`PINGWorks.SitecoreSearch.DiscoverSearch`](https://www.nuget.org/packages/PINGWorks.SitecoreSearch.DiscoverSearch/) — leaf SDK for Sitecore Discover (Search + Events + Ingestion)
- `PINGWorks.SitecoreSearch.CodeGenerators` — the Roslyn source generator, shipped inside this package (not published separately)

## Known issues

None.
