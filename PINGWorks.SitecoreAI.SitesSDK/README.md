# PINGWorks.SitecoreAI.SitesSDK

This is a .NET SDK for working with SitecoreAI's Sites API.<br />
More information about Sitecore's API is at [Sites API Doc (official)](https://api-docs.sitecore.com/sai/sites-api)

## Features
- Typed, DI-friendly service for interacting with the SitecoreAI Sites API
- Async-first API with `CancellationToken` support and full XML documentation comments
- NetStandard 2.1 for wide compatibility
- Automatic token maintenance using client credentials flow
- Optional `environmentId` parameter on all operations for targeting specific environments
- Individual focused interfaces for each API group for narrower dependency injection

## Getting started
### Prerequisites
To work with the SDK you will require credentials which are created in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/credentials/environment).

Create an **Environment** credential of type **Automation** for the project and environment
combination for which you wish to grant access.

### Install
From your project directory:

```shell
dotnet add package PINGWorks.SitecoreAI.SitesSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You do not need to include values where you wish to use the defaults.

```json
{
    /* The base URL of the Sites API */
    "ServiceEndpoint": "https://xmapps-api.sitecorecloud.io/",

    /* The URL of the Token Request API */
    "TokenEndpoint": "https://auth.sitecorecloud.io/oauth/token",

    /* The auth audience */
    "Audience": "https://api.sitecorecloud.io",

    /* The auth grant type */
    "GrantType": "client_credentials",

    /* The value of ClientID from the SitecoreAI Deploy Portal */
    "ClientID": "[Created in SitecoreAI Deploy portal's Credentials pane]",

    /* The value of ClientSecret from the SitecoreAI Deploy Portal */
    "ClientSecret": "[Created in SitecoreAI Deploy portal's Credentials pane]"
}
```

We recommend the use of Visual Studio's User Secrets feature to store sensitive information such as `ClientId` and `ClientSecret` during development.

### Register services
Register the SDK in `Program.cs`, e.g. when using the minimal hosting model:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Register SDK services - set options through binding or manually
builder.Services.AddSitecoreEESitesSDK( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

### Using the environment ID

Most operations accept an optional `environmentId` parameter that targets a specific SAI environment. When omitted, the default environment is used.

```csharp
// Target a specific environment
var sites = await sdk.ListSites( environmentId: "main" );
```

## Available services

Inject `ISitecoreSitesSdk` for full API access, or inject any of the individual group interfaces for narrower, more focused dependencies.

### Jobs — `ISitecoreSitesJobsSdk`
| Method | Description |
| - | - |
| `ListJobs()` | Lists all background jobs. |
| `GetJobStatus(jobHandle)` | Retrieves the current status of a background job. |

### Languages — `ISitecoreSitesLanguagesSdk`
| Method | Description |
| - | - |
| `ListLanguages(environmentId?)` | Lists all languages in the environment. |
| `CreateLanguage(request, environmentId?)` | Adds a new language to the environment. |
| `ListSupportedLanguages(environmentId?)` | Lists languages available for use as site languages. |
| `UpdateLanguage(isoCode, request, environmentId?)` | Updates a language's properties. |
| `DeleteLanguage(isoCode, environmentId?)` | Deletes a language from the environment. |

### Collections — `ISitecoreSitesCollectionsSdk`
| Method | Description |
| - | - |
| `ListCollections(environmentId?)` | Lists all site collections. |
| `CreateCollection(request, environmentId?)` | Creates a new site collection (async job). |
| `GetCollection(collectionId, environmentId?)` | Retrieves a site collection by ID. |
| `UpdateCollection(collectionId, request, environmentId?)` | Updates a site collection's display name, description, or sort order. |
| `DeleteCollection(collectionId, environmentId?)` | Deletes a site collection (async job). |
| `ListCollectionSites(collectionId, environmentId?)` | Lists the sites in a collection. |
| `RenameCollection(collectionId, request, environmentId?)` | Renames a collection's system name (async job). |
| `SortCollections(request)` | Sets the sort order for multiple collections. |
| `ValidateCollectionName(request, environmentId?)` | Validates a proposed collection name. |

### Favorites — `ISitecoreSitesFavoritesSdk`
| Method | Description |
| - | - |
| `ListFavoriteSites(environmentId?)` | Lists the current user's favorite sites. |
| `AddFavoriteSite(request, environmentId?)` | Adds a site to favorites. |
| `RemoveFavoriteSite(siteId, environmentId?)` | Removes a site from favorites. |
| `ListFavoriteSiteTemplates(environmentId?)` | Lists the current user's favorite site templates. |
| `AddFavoriteSiteTemplate(request, environmentId?)` | Adds a site template to favorites. |
| `RemoveFavoriteSiteTemplate(siteTemplateId)` | Removes a site template from favorites. |

### Editor Profiles — `ISitecoreSitesEditorProfilesSdk`
| Method | Description |
| - | - |
| `ListEditorProfiles()` | Lists all editor toolbar profiles. |
| `CreateEditorProfile(request)` | Creates a new editor toolbar profile. |
| `GetEditorProfile(id)` | Retrieves an editor profile by ID. |
| `UpdateEditorProfile(id, request)` | Updates an editor toolbar profile. |
| `DeleteEditorProfile(id)` | Deletes an editor toolbar profile. |

### Aggregation — `ISitecoreSitesAggregationSdk`
| Method | Description |
| - | - |
| `AggregateLivePageVariants(pages)` | Retrieves live variant identifiers for a set of pages. |
| `AggregatePageData(pages)` | Retrieves aggregated component data for a set of pages. |

### Sites — `ISitecoreSitesSitesSdk`
#### Sites
| Method | Description |
| - | - |
| `ListSites(environmentId?)` | Lists all sites in the environment. |
| `CreateSite(request, environmentId?)` | Creates a new site (async job). |
| `GetSite(siteId, environmentId?)` | Retrieves a site by ID. |
| `UpdateSite(siteId, request, environmentId?)` | Updates a site's properties. |
| `DeleteSite(siteId, force?, environmentId?)` | Deletes a site (async job). |
| `CopySite(siteId, request, environmentId?)` | Copies a site with a new name (async job). |
| `RenameSite(siteId, request, environmentId?)` | Renames a site's system name (async job). |
| `SortSites(request, environmentId?)` | Sets the sort order for multiple sites. |
| `ValidateSiteName(request, environmentId?)` | Validates a proposed site name. |
| `ListSiteTemplates(environmentId?)` | Lists available site templates. |

#### Analytics identifiers
| Method | Description |
| - | - |
| `ListTrackedSites(analyticsIdentifier, environmentId?)` | Lists sites using a given analytics identifier. |
| `DetachAnalyticsIdentifier(analyticsIdentifier, request, environmentId?)` | Detaches an analytics identifier from sites. |

#### Translation
| Method | Description |
| - | - |
| `TranslateSite(siteId, request, environmentId?)` | Translates a site to a target language (async job). |

#### Hierarchy
| Method | Description |
| - | - |
| `GetSiteHierarchy(siteId, language?, environmentId?)` | Retrieves the full page hierarchy for a site. |
| `GetPageHierarchy(siteId, pageId, language?, environmentId?)` | Retrieves hierarchy context for a page. |
| `ListPageAncestors(siteId, pageId, language?, environmentId?)` | Lists ancestors of a page. |
| `ListPageChildren(siteId, pageId, language?, environmentId?)` | Lists direct children of a page. |

#### Hosts
| Method | Description |
| - | - |
| `ListHosts(siteId, environmentId?)` | Lists all hosts for a site. |
| `CreateHost(siteId, request, environmentId?)` | Creates a new host for a site. |
| `GetHost(siteId, hostId, environmentId?)` | Retrieves a host by ID. |
| `UpdateHost(siteId, hostId, request, environmentId?)` | Updates a host. |
| `DeleteHost(siteId, hostId, force?, environmentId?)` | Deletes a host. |
| `ListRenderingHosts(siteId, environmentId?)` | Lists rendering hosts for a site. |
| `ListEditingHosts(siteId, environmentId?)` | Lists editing hosts for a site. |

#### Sitemap
| Method | Description |
| - | - |
| `GetSitemapConfiguration(siteId, environmentId?)` | Retrieves the sitemap configuration for a site. |
| `UpdateSitemapConfiguration(siteId, request, environmentId?)` | Updates the sitemap configuration for a site. |

#### Statistics
| Method | Description |
| - | - |
| `GetLocalizationStatistics(siteId, environmentId?)` | Retrieves localization statistics for a site. |
| `GetWorkflowStatistics(siteId, environmentId?)` | Retrieves workflow statistics for a site. |
