# PINGWorks.SitecoreExperienceEdge.AgentSDK

This is a .NET SDK for working with Sitecore's Experience Edge Agent API.<br />
More information about Sitecore's API is at [Agent API Doc (official)](https://api-docs.sitecore.com/sai/agent-api)

## Features
- Typed, DI-friendly services for interacting with the Sitecore Agent API
- Async-first APIs with `CancellationToken` support and full XML documentation comments
- NetStandard 2.1 for wide compatibility
- Automatic token maintenance using client credentials flow
- `x-sc-job-id` header support on all operations for tracing and reverting agent actions

## Getting started
### Prerequisites
To work with the SDK you will require credentials which are created in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/credentials/environment).

Create an **Environment** credential of type **Automation** for the project and environment
combination for which you wish to grant access.

### Install
From your project directory:

```csharp
dotnet add package PINGWorks.SitecoreExperienceEdge.AgentSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You do not need to include values where you wish to use the defaults.

```json
{
	/* The base URL of the Agent API */
	"ServiceEndpoint": "https://edge-platform.sitecorecloud.io/stream/ai-agent-api/",

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
builder.Services.AddSitecoreEEAgentApi( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

### Using job IDs

The Agent API supports a `x-sc-job-id` header on all operations. When provided, the job ID is used by Sitecore to group operations so they can be traced, audited, and reverted as a unit.

```csharp
var jobId = $"job-{Guid.NewGuid()}";

// All operations with the same jobId can be reverted together
await sdk.CreatePage( request, jobId );
await sdk.AddComponentToPage( pageId, componentRequest, jobId );

// Undo everything in the job
await sdk.RevertJob( jobId );
```

## Available services

The SDK exposes a composite interface `ISitecoreAgentSdk` as well as 10 focused resource-group interfaces. Inject the composite for full API access, or inject individual interfaces for narrower dependencies.

### Composite interface `ISitecoreAgentSdk`
Extends all 10 resource interfaces. Use when a single component needs access to multiple API groups.

### `ISitecoreAgentSitesSdk`
| Method | Description |
| - | - |
| `ListSites(jobId?)` | Retrieves a list of available sites. |
| `GetSiteDetails(siteId, jobId?)` | Retrieves the details of a specific site. |
| `GetPagesBySite(siteName, language?, jobId?)` | Lists pages of a site, optionally filtered by language. |
| `GetSiteIdFromItem(itemId, jobId?)` | Retrieves a site ID from an item ID. |

### `ISitecoreAgentPagesSdk`
| Method | Description |
| - | - |
| `CreatePage(request, jobId?)` | Creates a new page. |
| `GetPageDetails(pageId, language?, jobId?)` | Retrieves the details of a page. |
| `SearchPages(searchQuery, siteName, language?, jobId?)` | Searches for pages on a site. |
| `GetPagePathByUrl(liveUrl, jobId?)` | Retrieves the page path by live URL. |
| `GetPageTemplateById(templateId, jobId?)` | Retrieves a page template by ID. |
| `AddLanguageToPage(pageId, request, jobId?)` | Adds a language version to a page. |
| `AddComponentToPage(pageId, request, jobId?)` | Adds a component to a page. |
| `GetPageComponents(pageId, language?, version?, jobId?)` | Retrieves components on a page. |
| `SetComponentDatasource(pageId, componentId, request, jobId?)` | Sets the component datasource. |
| `GetPageHtml(pageId, language?, version?, jobId?)` | Retrieves the HTML of a page. |
| `GetPagePreviewUrl(pageId, language?, version?, jobId?)` | Retrieves the page preview URL. |
| `GetPageScreenshot(pageId, language?, version?, width?, height?, jobId?)` | Retrieves a screenshot of a page. |
| `GetAllowedComponents(pageId, placeholderName, language?, jobId?)` | Lists allowed components for a placeholder. |

### `ISitecoreAgentContentSdk`
| Method | Description |
| - | - |
| `CreateContent(request, jobId?)` | Creates a content item. |
| `GetContentById(itemId, language?, jobId?)` | Retrieves a content item by ID. |
| `GetContentByPath(itemPath, failOnNotFound?, language?, jobId?)` | Retrieves a content item by path. |
| `UpdateContent(itemId, request, jobId?)` | Updates a content item. |
| `DeleteContent(itemId, language?, jobId?)` | Deletes a content item. |
| `GetContentInsertOptions(itemId, language?, jobId?)` | Lists insert options for a content item. |

### `ISitecoreAgentComponentsSdk`
| Method | Description |
| - | - |
| `ListComponents(siteName, jobId?)` | Lists all components available for a site. |
| `GetComponentDetails(componentId, jobId?)` | Retrieves the details of a component. |
| `CreateComponentDatasource(componentId, request, jobId?)` | Creates a component datasource. |
| `SearchComponentDatasources(componentId, term, jobId?)` | Searches for component datasources. |

### `ISitecoreAgentAssetsSdk`
| Method | Description |
| - | - |
| `UploadAsset(fileContent, fileName, jobId?)` | Uploads an asset. |
| `SearchAssets(query, language?, type?, jobId?)` | Searches for assets. |
| `GetAssetDetails(assetId, jobId?)` | Retrieves the asset details. |
| `UpdateAsset(assetId, request, jobId?)` | Updates an asset. |

### `ISitecoreAgentEnvironmentsSdk`
| Method | Description |
| - | - |
| `ListLanguages(jobId?)` | Lists languages available in the environment. |

### `ISitecoreAgentPersonalizationSdk`
| Method | Description |
| - | - |
| `CreatePersonalizationVariant(pageId, request, jobId?)` | Creates a personalization variant. |
| `GetPersonalizationByPage(pageId, language?, jobId?)` | Lists personalization variants by page. |
| `ListConditionTemplates(jobId?)` | Lists personalization condition templates. |
| `GetConditionTemplate(templateId, jobId?)` | Retrieves a specific condition template. |

### `ISitecoreAgentJobsSdk`
| Method | Description |
| - | - |
| `GetJobDetails(jobId)` | Retrieves job details. |
| `ListJobOperations(jobId)` | Lists operations in a job. |
| `RevertJob(jobId)` | Reverts all operations in a job. |

### `ISitecoreAgentBrandKitsSdk`
| Method | Description |
| - | - |
| `ListBrandKits(jobId?)` | Lists available brand kits. |
| `GetBrandKit(brandKitId, jobId?)` | Retrieves a brand kit by ID. |

### `ISitecoreAgentBriefsSdk`
| Method | Description |
| - | - |
| `ListBriefTypes(jobId?)` | Lists available brief types. |
| `CreateBrief(request, jobId?)` | Creates a brief and saves it as a draft. |
| `GenerateBrief(request, jobId?)` | Generates a brief using AI. |
