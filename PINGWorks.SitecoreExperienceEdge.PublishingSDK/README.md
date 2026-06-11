# PINGWorks.SitecoreExperienceEdge.PublishingSDK

This is a .NET SDK for working with Sitecore's Experience Edge Publishing API.<br />
More information about Sitecore's API is at [Publishing API Doc (official)](https://api-docs.sitecore.com/sai/publishing-api)

## Features
- Typed, DI-friendly service for interacting with the Sitecore Experience Edge Publishing API
- Async-first API with `CancellationToken` support and full XML documentation comments
- NetStandard 2.1 for wide compatibility
- Automatic token maintenance using client credentials flow
- Offset and checkpoint pagination for listing publishing jobs

## Getting started
### Prerequisites
To work with the SDK you will require credentials which are created in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/credentials/environment).

Create an **Environment** credential of type **Automation** for the project and environment
combination for which you wish to grant access.

### Install
From your project directory:

```shell
dotnet add package PINGWorks.SitecoreExperienceEdge.PublishingSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You do not need to include values where you wish to use the defaults.

```json
{
    /* The base URL of the Publishing API */
    "ServiceEndpoint": "https://edge-platform.sitecorecloud.io/authoring/publishing/v1/",

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
builder.Services.AddSitecoreEEPublishingSDK( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

This library uses Json serialization through `System.Text.Json`.

## Available services

### Injectable interface `ISitecorePublishingSdk`

#### Jobs
| Method | Description |
| - | - |
| `GetJob(id)` | Retrieves a specific publishing job by ID. |
| `CreateJob(request)` | Creates a new publishing job. |
| `ListJobs(pageNumber?, pageSize?, source?, createdById?, createdByName?, status?, queuedTime?, startTime?, finishTime?, sortBy?)` | Lists publishing jobs using offset pagination (max 1000 records). |
| `ListJobsFromCheckpoint(continuationToken?, pageSize?, source?, createdById?, createdByName?, status?, queuedTime?, startTime?, finishTime?, sortBy?)` | Lists publishing jobs using checkpoint pagination (no record limit). |
| `CancelJob(jobId)` | Cancels a publishing job with `Queued` or `Running` status. |

#### Filters & summary
| Method | Description |
| - | - |
| `GetJobFilters(source?, createdById?, createdByName?, status?, queuedTime?, startTime?, finishTime?)` | Retrieves distinct filter values for dynamically populating filter UI. |
| `GetJobsSummary(queuedTime?)` | Retrieves job counts grouped by status. |

#### Permissions
| Method | Description |
| - | - |
| `GetPermissions()` | Retrieves the permissions of the currently authenticated user or application. |

## Pagination

Both list methods share the same filter and sort parameters; they differ only in how they page through results.

**Offset pagination** (`ListJobs`) — use `pageNumber` and `pageSize`. Capped at 1000 total records.

```csharp
var page1 = await sdk.ListJobs( pageNumber: 1, pageSize: 50 );
var page2 = await sdk.ListJobs( pageNumber: 2, pageSize: 50 );
```

**Checkpoint pagination** (`ListJobsFromCheckpoint`) — no record limit. Pass `null` for the first
call, then pass the `Next` token from each response until `Next` is `null`.

```csharp
string? token = null;
do
{
    var result = await sdk.ListJobsFromCheckpoint( continuationToken: token, pageSize: 100 );
    // process result.Result?.Data
    token = result.Result?.Next;
}
while ( token != null );
```

## Filtering

### Source and status — `[Flags]` enums

`source` accepts a `PublishJobSources` flags enum; `status` accepts a `PublishJobStatus` flags enum.
Combine multiple values with the bitwise OR operator:

```csharp
var jobs = await sdk.ListJobs(
    source: PublishJobSources.Pages | PublishJobSources.Sites,
    status: PublishJobStatus.Completed | PublishJobStatus.Failed
);
```

### Date ranges — `DateRangeFilter`

`queuedTime`, `startTime`, and `finishTime` each accept a `DateRangeFilter`. Set `From` for a
lower bound, `To` for an upper bound, or both for a closed range:

```csharp
var jobs = await sdk.ListJobs(
    queuedTime: new DateRangeFilter
    {
        From = new DateTime( 2024, 10, 14 ),
        To   = new DateTime( 2024, 12, 31 )
    }
);
```

A one-sided range is also valid:

```csharp
// All jobs queued since 1 Jan 2025
var jobs = await sdk.ListJobs(
    queuedTime: new DateRangeFilter { From = new DateTime( 2025, 1, 1 ) }
);
```

### Creator filters — `IEnumerable<string>`

`createdById` and `createdByName` accept any number of string values; jobs matching any of the
supplied values are returned:

```csharp
var jobs = await sdk.ListJobs(
    createdByName: [ "Jane Smith", "John Doe" ]
);
```

## Sorting

Pass a `PublishJobSortBy` value to either list method. Each member encodes both the sort field and
its direction:

```csharp
// Most recently queued jobs first
var jobs = await sdk.ListJobs(
    sortBy: PublishJobSortBy.QueuedTimeDescending
);

// Alphabetical by name
var jobs = await sdk.ListJobsFromCheckpoint(
    sortBy: PublishJobSortBy.NameAscending
);
```

| Value | API field | Direction |
| - | - | - |
| `NameAscending` / `NameDescending` | `name` | A→Z / Z→A |
| `SourceAscending` / `SourceDescending` | `source` | A→Z / Z→A |
| `StatusAscending` / `StatusDescending` | `system.status` | asc / desc |
| `QueuedTimeAscending` / `QueuedTimeDescending` | `system.queuedTime` | oldest / newest |
| `StartTimeAscending` / `StartTimeDescending` | `system.startTime` | oldest / newest |
| `FinishTimeAscending` / `FinishTimeDescending` | `system.finishTime` | oldest / newest |

## Dynamic filter data

`GetJobFilters` returns the distinct values currently present across matching jobs, scoped by any
filters you supply. Use it to drive filter dropdowns that only show relevant options:

```csharp
// What sources and statuses exist for jobs that failed?
var filters = await sdk.GetJobFilters( status: PublishJobStatus.Failed );
var sources  = filters.Result?.Sources?.Select( s => s.Data?.Source );
var statuses = filters.Result?.Statuses?.Select( s => s.Data?.Status );
```

## Job summary

`GetJobsSummary` returns per-status counts, optionally restricted to a queued-time window:

```csharp
var summary = await sdk.GetJobsSummary(
    queuedTime: new DateRangeFilter { From = new DateTime( 2025, 1, 1 ) }
);
Console.WriteLine( $"Running: {summary.Result?.Counts?.Running}" );
Console.WriteLine( $"Failed:  {summary.Result?.Counts?.Failed}" );
```
