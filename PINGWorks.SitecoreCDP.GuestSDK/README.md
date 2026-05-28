# PINGWorks.SitecoreCDP.GuestSDK

This is a .NET SDK for working with Sitecore CDP's Guest API.<br />
More information about Sitecore's API is at [Guest API Doc (official)](https://api-docs.sitecore.com/cdp/guest-rest-api)

## Features
- Typed, DI-friendly service for interacting with the Sitecore CDP Guest API v2.1
- Async-first API with full XML documentation comments
- NetStandard 2.1 for wide compatibility
- HTTP Basic Auth using Client Key and API Token from the CDP portal
- Focused sub-interfaces for guest management and data extensions

## Getting started
### Prerequisites
To work with the SDK you will need a **Client Key** and **API Token** from
**Settings &gt; API access** in the [Sitecore CDP portal](https://app.boxever.com/).

### Install
From your project directory:

```shell
dotnet add package PINGWorks.SitecoreCDP.GuestSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values.

```json
{
    /* The Sitecore CDP API region. Valid values: US, EU, AP, JP (default: US) */
    "Region": "US",

    /* The Client Key from Settings > API access in the Sitecore CDP portal */
    "ClientKey": "[Your Client Key]",

    /* The API Token from Settings > API access in the Sitecore CDP portal */
    "ApiToken": "[Your API Token]"
}
```

The regional endpoint URLs default to the standard Sitecore values. If you need to override a specific region's URL, set the corresponding property (`ServiceEndpointUs`, `ServiceEndpointEu`, `ServiceEndpointAp`, or `ServiceEndpointJp`).

We recommend the use of Visual Studio's User Secrets feature to store sensitive information such as `ClientKey` and `ApiToken` during development.

### Register services
Register the SDK in `Program.cs`, e.g. when using the minimal hosting model:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Register SDK services - set options through binding or manually
builder.Services.AddSitecoreCdpGuestSDK( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

### Guest type and status values

Use the enum and string constant types for type-safe API values:

```csharp
using PINGWorks.SitecoreCDP.GuestSDK.Enums;

var request = new CreateGuestRequest
{
    GuestType = GuestType.Customer,
    FirstName = "Jane",
    LastName = "Smith",
    Subscriptions =
    [
        new GuestSubscription
        {
            Channel = "EMAIL",
            Name = "newsletter",
            PointOfSale = "my-store",
            Status = SubscriptionStatus.Subscribed
        }
    ]
};
```

### Data extensions

Each guest can have up to six free-form key-value data extension slots. Use
`DataExtensionName` enum values for the slot name:

```csharp
using PINGWorks.SitecoreCDP.GuestSDK.Enums;

var ext = await sdk.CreateGuestDataExtension( guestRef, new CreateGuestDataExtensionRequest
{
    Name = DataExtensionName.Ext1,
    AdditionalData = new Dictionary<string, object>
    {
        ["vipMember"] = true,
        ["loyaltyTier"] = "gold",
        ["points"] = 1500
    }
});
```

## Available services

### Guests — `ISitecoreCdpGuestSdk`
| Method | Description |
| - | - |
| `CreateGuest(request)` | Creates a new guest record. |
| `RetrieveGuests(offset?, limit?, expand?, sort?)` | Lists guests with pagination. |
| `RetrieveGuest(guestRef, expand?)` | Retrieves a single guest by reference. |
| `ReplaceGuest(guestRef, request)` | Fully replaces a guest's writable fields (PUT). |
| `PatchGuest(guestRef, request)` | Partially updates a guest (PATCH). |
| `DeleteGuest(guestRef)` | Deletes a guest and all associated data. |

### Guest data extensions — `ISitecoreCdpGuestExtensionsSdk`
| Method | Description |
| - | - |
| `CreateGuestDataExtension(guestRef, request)` | Creates a data extension on a guest. |
| `RetrieveGuestDataExtensions(guestRef, offset?, limit?, expand?)` | Lists a guest's data extensions. |
| `RetrieveGuestDataExtension(guestRef, extensionName, expand?)` | Retrieves a single data extension. |
| `ReplaceGuestDataExtension(guestRef, extensionName, request)` | Fully replaces a data extension's data (PUT). |
| `PatchGuestDataExtension(guestRef, extensionName, request)` | Partially updates a data extension (PATCH). |
| `DeleteGuestDataExtension(guestRef, extensionName)` | Deletes an entire data extension. |
| `DeleteGuestDataExtensionFields(guestRef, extensionName, fieldNames?)` | Deletes specific fields from a data extension. |
