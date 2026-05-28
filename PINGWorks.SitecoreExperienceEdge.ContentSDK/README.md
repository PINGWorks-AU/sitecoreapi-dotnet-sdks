# PINGWorks.SitecoreExperienceEdge.ContentSDK

This is a .NET SDK for working with Sitecore's Experience Edge Delivery API.<br />
The API is still being actively developed so there is no official documentation yet.

This library has been named ContentSDK as it follows the purpose of Sitecore's Content SDK, even though technically it's the Delivery API
we are wrapping. The intent is to aid in discoverability, as the Content SDK is much more widely known than the Delivery API.


## Features
- Typed, DI-friendly services for interacting with Sitecore EE Delivery API
- Async-first APIs with `CancellationToken` support
- NetStandard 2.1 for wide compatibility

## Getting started
### Prerequisites
To work with the SDK you will require an Experience Edge Context ID, which can be found in the [SitecoreAI Deployment
Portal](https://deploy.sitecorecloud.io/projects) in the `Developer Settings` page for your environment.

This library has a dependency on [PolyglotDataStudio.GraphQL](https://www.nuget.org/packages/PolyglotDataStudio.GraphQL#readme-body-tab)
which allows for strongly-typed GraphQL queries to be created. There is a companion library called [PolyglotDataStudio.GraphQL.CLI](https://www.nuget.org/packages/PolyglotDataStudio.GraphQL.CLI#readme-body-tab)
which is a code-generator, and can create a complete set of types for use with Sitecore using GraphQL's introspection features.

Using strongly-typed GraphQL makes it much easier to create queries without having to memorize all the fields and parameters
in the Sitecore GraphQL schema... and there are A LOT. It also ensures you get the data types right and makes it simple
to access result properties.

Overloads are provided that also take GraphQL queries as strings, instead of Polyglot types, if that is your preference.

### Install
From your project directory:

```csharp
dotnet add package PINGWorks.SitecoreExperienceEdge.ContentSDK
```

### AppSettings

Properties are available to configure the SDK through configuration binding. The snippet below shows all options along with their default values. You **do not** need to include values where you wish to use the defaults.

```json
{
	/* The query string key to use when sending the ContextId to Edge's GraphQL endpoint.
	   Do not change this unless you know what you are doing. */
	"ContextIdQueryStringKey": "sitecoreContextId",

	/* Required. This is the Context ID of your SitecoreAI environment.
		Put this in your User Secrets file */
	"EdgeContextId": "6********************w",

	/* Edge service endpoint. Don't change this unless you know what you are doing. */
	"ServiceEndpoint": "https://edge-platform.sitecorecloud.io/v1/",

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
builder.Services.AddSitecoreEEContentSdk( opt => config.GetSection( "mySettings" ).Bind( opt ) );
```

> **HttpClient defaults.** This call (via `AddSitecoreEECommon`) applies `AddStandardResilienceHandler()`
> and a DEBUG-only console request/response logger to **every** HttpClient in your container —
> not just the SDK's own. See the [PINGWorks.SitecoreExperienceEdge.Common](https://www.nuget.org/packages/PINGWorks.SitecoreExperienceEdge.Common#readme-body-tab)
> README for how to customize the resilience options per-client.

## Available services

### Injectable interface `ISitecoreFormsSdk`

| Method | Description |
| - | - |
| GetForm( string ) | Retrieve the HTML associated with a Form defined in Sitecore. |

### Injectable interface `ISitecoreGraphQLSdk`

| Method | Description |
| - | - |
| GetLayout( ... ) | Get the page layout for a given Sitecore route and language. |
| GetEditingLayout( ... ) | Get the editing layout for a Sitecore item. Only metadata-mode editing is supported. |
| Execute&lt;T&gt;( string ) | Execute a GraphQL query against Sitecore and parse the response into a model. |
| Execute&lt;T&gt;( GraphQLRootOperation ) | Execute a GraphQL query against Sitecore and parse the response into a model. |
| Execute( GraphQLRootOperation ) | Execute a GraphQL query against Sitecore and set the response into the operation parameter. |

###  Models

Many models are supplied to represent the standard responses to layout APIs. These are largely contained in the namespace
`PINGWorks.SitecoreExperienceEdge.ContentSDK.Data`. In general, these are not needed directly.

Extensive models are supplied to provided strongly-typed access to Sitecore Fields. These can be found in the namespace
`PINGWorks.SitecoreExperienceEdge.ContentSDK.Fields` and `PINGWorks.SitecoreExperienceEdge.ContentSDK.Fields.Sitecore`. The
base namespeace contains Field primitives, while the `Sitecore` namespace contains direct representations of all supported
field types.

#### Namespace: PINGWorks.SitecoreExperienceEdge.ContentSDK.Fields

| Type | Description |
| - | - |
| IField | Base interface supporting all field types |
| ValueField | Base class representing a field with a Value property |
| EditableValueField | Base class representing a field that can be edited in Sitecore Pages UI |
| FieldMetaData | Metadata required to supported Sitecore Pages editing functions |
|  |  |
| BooleanField | A field containing a `boolean` value |
| DateTimeField | A field containing a `DateTime` value |
| DecimalField | A field containing a `decimal` value |
| HyperlinkField | A field containing one of several types of supported hyperlink<br />Types include: Internal, Media, External, Anchor, Mailto, Javascript |
| GuidField | A field containing a `Guid` value |
| IntegerField | A field containing a `integer` value |
| ItemLinkField | A field containing a link to another Sitecore item |
| ItemLinksField | A field containing a list of links to other Sitecore items |
| StringField | A field containing a `string` value |

#### Namespace: PINGWorks.SitecoreExperienceEdge.ContentSDK.Fields.Sitecore

The types below provide a convenient mapping to Sitecore's native field types. In many cases these are simple
wrappers for primitive field types, so they can be used interchangably. In other cases these provided extended
functionality for mapping additional properties or metadata.

| Type | Base primitive | Interchangable with base |
| - | - | - |
| CheckboxField | BooleanField | Yes |
| ChecklistField | ItemLinksField | Yes |
| DateField | DateTimeField | Yes |
| DroplistField | StringField | Yes |
| DropTreeField | ItemLinkField | Yes |
| FileField |  | No |
| GeneralLinkField | HyperlinkField | Yes |
| GeneralLinkSearchField | StringField | No |
| GraphQLField | StringField | Yes |
| GroupedDroplinkField | ItemLinkField | Yes |
| GroupedDroplistField | StringField | Yes |
| ImageField |  | No |
| NameValueListField | StringField | No |
| PasswordField | StringField | Yes |
| PluginField | StringField | No |
| RedirectMapField | NameValueListField | Yes |
| RichTextField | StringField | Yes, but this field automatically url-decodes values if it detects them. |
| SingleLineTextField | StringField | Yes |
| TagListField | GuidField | Yes |
| TreelistExField | ItemLinksField | Yes |
| TreelistField | ItemLinksField | Yes |

For those items marked as "No" above, be sure to use the correct Sitecore Field type in your projects to gain access
to the full set of properties needed for rendering.

### Model code generation for Sitecore using Polyglot.GraphQL.CLI

Models for use with general GraphQL requests should be generated manually using the 
[Polyglot.GraphQL.CLI](https://www.nuget.org/packages/PolyglotDataStudio.GraphQL.CLI#readme-body-tab) tool, as these are
project-specific.

The following PowerShell makes for a useful starting point when generating GraphQL models for your Sitecore project.

```powershell
# dotnet tool install --global polyglotdatastudio.graphql.cli
# dotnet tool update --global polyglotdatastudio.graphql.cli

# Preview URL https://xmc-xxxxxxxxxx.sitecorecloud.io/sitecore/api/graph/edge
# Live URL https://edge.sitecorecloud.io/api/graphql

& polyglotdatastudio.graphql.cli `
	-u "https://xmc-xxxxxxxxxx.sitecorecloud.io/sitecore/api/graph/edge" `
	-h "{ 'sc_apikey': '[`$prompt:sc_apikey]' }" `
	-n "MyProject.Web.GraphQL" `
	-o ".\apps\MyProject.Web.GraphQL\" `
	--force `
	--public
```

This will use the Preview (unpublished) Sitecore endpoint to retrieve all GraphQL types and will generate
supporting query and model classes for them.

- `-h` sets the header, the `$prompt` means the value will be prompted when the command is run
- `-n` sets the base namespace for the generated code
- `-o` sets the output directory
- `--force` overwrites existing files in the output directory
- `--public` sets the generated types to use **public** access modifiers rather than the default **internal** access modifiers

It is recommended that this script be saved into the root of your project, so that it's easy to run when updates
are made to Sitecore templates. Further, it is recommended that the output be added to a dependency library, rather
than being incorporated into the main library, as it is easier to version and control this way.
