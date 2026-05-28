# PINGWorks.SitecoreExperienceEdge.AdminSDK

This is a .NET SDK for working with Sitecore's Experience Edge Admin API.<br />
More information about Sitecore's API is at [Admin API Doc (official)](https://doc.sitecore.com/sai/en/developers/sitecoreai/admin-api.html)

## Features
- Typed, DI-friendly services for interacting with Sitecore EE Admin API
- Async-first APIs with `CancellationToken` support
- NetStandard 2.1 for wide compatibility
- Automatic token maintenance

## Getting started
### Prerequisites
To work with the SDK you will require credentials which are created in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/credentials/environment).

Create an **Environment** credential of type **Edge Administration** for the project and environment
combination for which you wish to grant access. At this time the SDK only supports connecting to a
single environment.

### Install
From your project directory:

```csharp
dotnet add package PINGWorks.SitecoreExperienceEdge.AdminSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You do not need to include values where you wish to use the defaults.

```json
{
	/* The base URL of the Admin API */
	"ServiceEndpoint": "https://edge.sitecorecloud.io/api/admin/v1/",

	/* The URL of the Token Request API */
	"TokenEndpoint": "https://auth.sitecorecloud.io/oauth/token",

	/* The auth audience */
	"Audience": "https://api.sitecorecloud.io",

	/* The auth grant type */
	"GrantType": "client_credentials",

	/* The value of ClientID from the SitecoreAI Deploy Portal */
	"ClientID": "[Created in SitecoreAI Deploy portal's Credentials pane]",

	/* The value of ClientSecret from the SitecoreAI Deploy Portal */
	"ClientSecret": "[Created in SitecoreAI Deploy portal's Credentials pane]",
}
```

When creating credentials for use with the SDK, ensure that you select the `Edge Administration` credential type.

We recommend the use of Visual Studio's User Secrets feature to store sensitive information such as	`ClientId` and `ClientSecret` during development.

### Register services
Register the SDK in `Program.cs`, e.g. when using the minimal hosting model:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Register SDK services - set options through binding or manually
builder.Services.AddSitecoreEEAdminSdk( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

## Available services

### Injectable interface `ISitecoreAdminSdk`

| Method | Description |
| - | - |
| ClearCacheForTenant() | Clears the entire cache for a given tenant. |
| DeleteContent() | Removes tenant data from the data storage. |
| GetSettings() | Lists all available settings for a tenant. |
| UpdateSettings( SettingsRequest ) | Updates all available settings for a tenant. |
| PatchSettings( params PatchRequest<SettingsRequest>[] ) | Updates a setting for a tenant using one or more Patch operations. |
| CreateWebhook( WebhookRequest ) | Creates a new webhook. |
| UpdateWebhook( string, WebhookRequest ) | Updates an existing webhook. |
| DeleteWebhook( string ) | Deletes a specific webhook. |
| ListWebhooks() | Lists all webhooks for a tenant. |
| GetWebhookById( string ) | Gets a specific tenant webhook. |

## Upgrade notes
When upgrading from v1.x to v2.x you will need to change service references from `IAdminSdk` to `ISitecoreAdminSdk`. 
