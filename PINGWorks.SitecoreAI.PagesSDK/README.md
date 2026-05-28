# PINGWorks.SitecoreAI.PagesSDK

This is a .NET SDK for working with SitecoreAI's Pages API.<br />
More information about Sitecore's API is at [Pages API Doc (official)](https://api-docs.sitecore.com/sai/pages-api)

## Features
- Typed, DI-friendly service for interacting with the SitecoreAI Pages API
- Async-first API with full XML documentation comments
- NetStandard 2.1 for wide compatibility
- Automatic token maintenance using client credentials flow
- Optional `environmentId` parameter on all operations for targeting specific environments

## Getting started
### Prerequisites
To work with the SDK you will require credentials which are created in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/credentials/environment).

Create an **Environment** credential of type **Automation** for the project and environment
combination for which you wish to grant access.

### Install
From your project directory:

```shell
dotnet add package PINGWorks.SitecoreAI.PagesSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You do not need to include values where you wish to use the defaults.

```json
{
    /* The base URL of the Pages API */
    "ServiceEndpoint": "https://xmapps-api.sitecorecloud.io/api/v1/",

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
builder.Services.AddSitecoreEEPagesSDK( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

### Using the environment ID

Most operations accept an optional `environmentId` parameter that targets a specific XM Cloud environment. When omitted, the default environment is used.

```csharp
// Target a specific environment
var page = await sdk.GetPage( pageId, site: "my-site", language: "en-US", environmentId: "main" );
```

## Available services

### Injectable interface `ISitecorePagesSdk`

#### Pages
| Method | Description |
| - | - |
| `CreatePage(request, environmentId?)` | Creates a new page. |
| `GetPage(pageId, site, language, version?, environmentId?)` | Retrieves a page's full details. |
| `UpdateFields(pageId, request, environmentId?)` | Updates field values on a page. |
| `DeletePage(pageId, request?, environmentId?)` | Deletes a page. |

#### State
| Method | Description |
| - | - |
| `GetPageState(pageId, site, language, version?, withWorkflow?, withVersions?, withLayout?, environmentId?)` | Retrieves lightweight page state. |
| `GetLivePage(pageId, language, environmentId?)` | Checks if a page is published to Edge. |
| `GetPageVariants(pageId, language, environmentId?)` | Lists active personalization variant identifiers. |

#### Search
| Method | Description |
| - | - |
| `SearchPages(searchText?, rootIds?, filter?, language?, pageSize?, pageNumber?, continuationToken?, environmentId?)` | Searches for pages and folders by name. |

#### Insert options
| Method | Description |
| - | - |
| `GetInsertOptions(pageId, site, language, insertOptionKind, environmentId?)` | Retrieves allowed child templates for a page. |

#### Versions
| Method | Description |
| - | - |
| `GetPageVersions(pageId, site, language?, environmentId?)` | Lists all versions of a page. |
| `AddPageVersion(pageId, request, environmentId?)` | Creates a new version of a page. |
| `DeletePageVersion(pageId, versionNumber, language, environmentId?)` | Deletes a specific page version. |

#### Layout & fields
| Method | Description |
| - | - |
| `SaveLayout(pageId, request, environmentId?)` | Updates the layout of a page. |
| `SaveFields(pageId, request, environmentId?)` | Updates the field values of a page. |

#### Operations
| Method | Description |
| - | - |
| `CreatePageFromBlueprint(request, environmentId?)` | Creates a page from a blueprint (async job). |
| `DuplicatePage(pageId, request, environmentId?)` | Duplicates a page. |
| `RenamePage(pageId, request, environmentId?)` | Renames a page. |
| `TranslatePage(pageId, request, siteId?, environmentId?)` | Translates a page using AI (async job). |
