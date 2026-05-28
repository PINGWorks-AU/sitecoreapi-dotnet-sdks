# PINGWorks.SitecoreExperienceEdge.EventsSDK

This is a .NET SDK for working with Sitecore's Experience Edge Events API.<br />
More information about Sitecore's API is at [CloudSDK Events Doc (official)](https://doc.sitecore.com/sdk/en/developers/latest/cloud-sdk/cloud-sdk-events.html)

## Features
- Typed, DI-friendly services for interacting with Sitecore EE Events API
- Async-first APIs with `CancellationToken` support
- NetStandard 2.1 for wide compatibility

## Getting started
### Prerequisites
To work with the SDK you will require an Experience Edge Context ID, which can be found in the [SitecoreAI Deployment Portal](https://deploy.sitecorecloud.io/projects) in the `Developer Settings` page for your environment.


### Install
From your project directory:

```csharp
dotnet add package PINGWorks.SitecoreExperienceEdge.EventsSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You **do not** need to include values where you wish to use the defaults.

```json
{
	/* The base URL of the Events API */
	"ServiceEndpoint": "https://edge-platform.sitecorecloud.io/v1/events/v1.2/",

	/* Required. This is the Context ID of your SitecoreAI environment.
		Put this in your User Secrets file */
	"EdgeContextId": "6********************w",

	/* The name of the site in the SitecoreAI Deploy portal.
	   Get this from SitecoreAI Deploy > Projects > Environments > Developer Settings.
	   Sitecore Deploy -> Projects -> Environments -> Site.Name */
	"SiteName": "my_website"
}
```

We recommend the use of Visual Studio's User Secrets feature to store sensitive information such as	`EdgeContextId` during development.

### Register services
Register the SDK in `Program.cs`, e.g. when using the minimal hosting model:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Register SDK services - set options through binding or manually
builder.Services.AddSitecoreEEEventsSdk( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

## Available services

### Injectable interface `ISitecoreEventsSdk`

| Method | Description |
| - | - |
| GetAnalyticsContextIds( HttpContext ) | Create new browser and guest identity tokens in the Sitecore Events API. |
| PageView( PageViewEventRequest, string, string ) | Record a page view event. |
| Identity( IdentityEventRequest, string, string ) | Associate an identity with the current visitor. |
| Form( FormEventRequest, string, string ) | Record form events. |
| Custom( CustomEventRequest, string, string ) | Send a custom event. |
