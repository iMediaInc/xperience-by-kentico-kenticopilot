---
name: migrate-business-layer
description: >-
  Migrates a .NET Framework 4.8 / Kentico Xperience 13 business layer to .NET 10
  / Xperience by Kentico. Rewrites all CMS, System.Web, and KX13 API usage to
  xByK and ASP.NET Core equivalents while preserving Tessitura/iMedia SDK calls.
  Use when migrating business layer code, services, extensions, models, or event
  handlers from KX13 to xByK.
---

# Migrate Business Layer (KX13 → xByK)

## Parameters

Before starting, collect these values from the user. Use them everywhere you see `{PARAM}` placeholders below.

| Parameter | Description | Example |
|-----------|-------------|---------|
| `{CLIENT}` | Client/project name used in **project/assembly** names (not necessarily content type ClassNames) | `Segerstrom` |
| `{CONTENT_TYPE_NAMESPACE}` | C# / ClassName namespace of generated entity types — must match registered `CMS_Class.ClassName` prefixes | `SCFTA` |
| `{KX13_SOURCE}` | Relative path to the KX13 business layer project | `kx13/Segerstrom.Business` |
| `{XBYK_WEB}` | Relative path to the xByK web project | `xbky` |
| `{KX13_CODENAME_PREFIX}` | KX13 document type code name prefix (usually the site code) | `SCFTA` |
| `{TESS_API_VERSION}` | iMedia.Tess.Api NuGet package version | `16.0.16` |
| `{KENTICO_CORE_VERSION}` | Kentico.Xperience.Core NuGet package version | `31.4.3` |
| `{ALGOLIA_VERSION}` | Algolia.Search NuGet package version (omit if not used) | `6.10.2` |
| `{HAS_ALGOLIA}` | Whether project uses Algolia search | `true` / `false` |
| `{HAS_WCF_SOAP}` | Whether project has WCF/SOAP Connected Services | `true` / `false` |
| `{HAS_DYNAMIC_PDF}` | Whether project uses ceTe.DynamicPDF | `true` / `false` |
| `{EXTRA_PACKAGES}` | Additional NuGet packages the business layer needs | `log4net`, `Kount.Net.RisSDK`, etc. |

If the user does not specify values, inspect the KX13 `.csproj` file at `{KX13_SOURCE}` to discover them: read the `<RootNamespace>`, package references, and connected services.

**Resolve `{CONTENT_TYPE_NAMESPACE}` before migrating** (see [content-type-namespaces.md](../_shared/references/content-type-namespaces.md)):

1. Migration plan Content Type Namespace Convention / target ClassNames
2. `CONTENT_TYPE_NAME` constants in `{CLIENT}.Entities` (authoritative after codegen)
3. `CMS_Class.ClassName` in the XbyK database

Default when content migration preserved KX13 names: `{CONTENT_TYPE_NAMESPACE}` = `{KX13_CODENAME_PREFIX}`. **Never default `{CONTENT_TYPE_NAMESPACE}` to `{CLIENT}`** unless the plan remapped ClassNames to `{CLIENT}` and Entities were generated that way.

## Prerequisites

- Source KX13 project at `{KX13_SOURCE}/`
- Target xByK web project at `{XBYK_WEB}/`
- `{CLIENT}.Entities` project with xByK-generated content types whose C# namespace and `CONTENT_TYPE_NAME` match registered ClassNames (`{CONTENT_TYPE_NAMESPACE}.*`)

## Phase 1: Scaffold the .NET 10 Project

Create `{CLIENT}.Business/{CLIENT}.Business.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>{CLIENT}.Business</RootNamespace>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\{CLIENT}.Entities\{CLIENT}.Entities.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Kentico.Xperience.Core" Version="{KENTICO_CORE_VERSION}" />
    <PackageReference Include="iMedia.Tess.Api" Version="{TESS_API_VERSION}" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Abstractions" Version="9.0.4" />
    <PackageReference Include="System.Configuration.ConfigurationManager" Version="9.0.4" />
    <!-- If {HAS_ALGOLIA}: -->
    <PackageReference Include="Algolia.Search" Version="{ALGOLIA_VERSION}" />
    <!-- If {HAS_WCF_SOAP}: -->
    <PackageReference Include="System.ServiceModel.Http" Version="8.1.2" />
    <PackageReference Include="System.ServiceModel.Primitives" Version="8.1.2" />
    <!-- Add {EXTRA_PACKAGES} here -->
  </ItemGroup>
</Project>
```

Do NOT include KX13 source files via `<Compile Include>`. All files are rewritten into the new project directory.

## Phase 2: Discovery

Before migrating, catalog the KX13 source. For each file in `{KX13_SOURCE}/`, record:
- Class name, namespace
- Which `CMS.*` namespaces and types it uses
- Which `System.Web` types it uses
- Which Tessitura/iMedia APIs it uses
- Dependencies on other files in the project

Group files into these categories:
1. Generated document types (`Models/Generated/Classes/`) — `TreeNode` subclasses + providers
2. Generated custom tables (`Models/Generated/CustomTables/`) — `CustomTableItem` subclasses
3. Generated partials (`Models/Generated/Partials/`) — business logic on generated types
4. Generated CMSModules — custom module `AbstractInfo` classes
5. Config/constants — `ConfigurationManager.AppSettings` readers
6. Extensions — static extension method classes
7. Models (Tessitura, Algolia, BuyButtons, etc.) — POCOs and DTOs
8. Services — business and integration services
9. Event handlers — `DocumentEvents` hooks
10. Scheduled tasks — `ITask` implementations
11. Connected Services — WCF/SOAP proxies
12. Utility — JSON helpers, factories, loggers

## Phase 3: Migration Order

Migrate in dependency order — each phase builds on the last:

1. **Compatibility shims** — minimal stubs for `System.Web`, `CMS.Scheduler`, and optionally `ceTe.DynamicPDF`
2. **ConfigSettings + Constants** — foundation, no deps
3. **Tessitura models** — POCOs, minor deps
4. **Custom table POCOs** — replace KX13 `CustomTableItem` subclasses
5. **ContentQueryService** — replaces all KX13 generated providers
6. **Content type extensions** — replace generated partials with extension methods on `{CONTENT_TYPE_NAMESPACE}.*` entity types
7. **Extensions** — depend on models, config, Tessitura SDK
8. **Algolia models + repositories** (if `{HAS_ALGOLIA}`)
9. **Core services** — depend on everything above
10. **Integration services** — third-party integrations
11. **EventHandlers + Tasks** — highest level
12. **Connected Services** (if `{HAS_WCF_SOAP}`)
13. **DI wiring + build verification**

## Phase 4: Replacement Patterns

### System.Web → ASP.NET Core

| KX13 | xByK / .NET 10 |
|------|-----------------|
| `System.Web.HttpContext.Current` | Inject `IHttpContextAccessor` |
| `System.Web.HttpContextBase` | `Microsoft.AspNetCore.Http.HttpContext` |
| `HttpContext.Current.Session["key"]` | `httpContext.Session.GetString("key")` / `SetString` |
| `System.Web.HttpCookie` | `Request.Cookies` / `Response.Cookies.Append()` |
| `System.Web.HtmlString` | `Microsoft.AspNetCore.Html.HtmlString` |
| `System.Web.HttpUtility` | `System.Net.WebUtility` or `System.Web.HttpUtility` (still available) |
| `System.Web.Hosting.HostingEnvironment.MapPath` | `IWebHostEnvironment.ContentRootPath` + `Path.Combine` |
| `System.Web.VirtualPathUtility.ToAbsolute` | Simple path resolution |
| `System.Net.WebClient` | `HttpClient` via `IHttpClientFactory` |

### CMS / Kentico APIs

| KX13 | xByK |
|------|------|
| `ConfigurationManager.AppSettings["key"]` | `IConfiguration["key"]` with static `Init()` pattern |
| `CMS.Helpers.CacheHelper` | `Microsoft.Extensions.Caching.Memory.IMemoryCache` |
| `CMS.SiteProvider.SiteContext.CurrentSiteName` | `ConfigSettings.Kentico.Sitename` |
| `CMS.Localization.LocalizationContext` | `CultureInfo.CurrentCulture` |
| `CMS.Localization.ResHelper` / `ResourceStringInfoProvider` | `IStringLocalizer` or simple service |
| `CMS.EventLog.EventLogProvider` | `Microsoft.Extensions.Logging.ILogger<T>` |
| `CMS.Helpers.URLHelper.ResolveUrl` | Simple path formatting |

### Content Queries (biggest change)

| KX13 | xByK |
|------|------|
| `CMS.DocumentEngine.Types.{KX13_CODENAME_PREFIX}.*` | `{CONTENT_TYPE_NAMESPACE}.*` (from `{CLIENT}.Entities` — often same as `{KX13_CODENAME_PREFIX}`, not `{CLIENT}`) |
| `DocumentQuery<T>` / `*Provider.Get*()` | `ContentQueryService` wrapping `IContentQueryExecutor` |
| `TreeNode` props (`NodeID`, `NodeAliasPath`, `NodeGUID`) | `IWebPageFieldsSource.SystemFields` (`WebPageItemID`, `WebPageItemTreePath`, `WebPageItemGUID`) |
| `CustomTableItem` / `CustomTableItemProvider` | POCO models in `Models/CustomTables/` |
| `DocumentHelper.GetDocuments<T>()` | `ContentItemQueryBuilder` + `IContentQueryExecutor.GetMappedWebPageResult<T>()` |
| `TreeProvider` / `TreeNode` insert/update | `IContentItemManager` |

### Static → Instance Classes

KX13 services were often `static` classes. Convert to instance classes with constructor injection:

```csharp
// KX13
public static class MyService {
    public static string DoThing() => CacheHelper.GetItem(...);
}

// xByK
public class MyService {
    private readonly IMemoryCache _cache;
    public MyService(IMemoryCache cache) => _cache = cache;
    public string DoThing() => _cache.GetOrCreate(...);
}
```

For extension methods that called static services, add a static `Instance` property on the service for backward compatibility.

### Tessitura / iMedia SDK

**Keep ALL Tessitura SDK calls exactly as-is.** The `iMedia.Tess.Api` and `Tessitura.Service.Contract` namespaces are .NET Standard and work unchanged on .NET 10.

Only changes needed:
- `TessituraLogger` must implement `iMedia.Tess.Api.Interfaces.iLogger`
- `TessituraCache` must implement `iMedia.Tess.Api.Interfaces.iCache` using `IMemoryCache`

## Phase 5: ContentQueryService Pattern

Create a single service replacing all KX13 generated providers:

```csharp
using CMS.ContentEngine;
using CMS.Websites;
using CMS.Websites.Routing;

namespace {CLIENT}.Business.Services;

public class ContentQueryService
{
    private readonly IContentQueryExecutor _executor;
    private readonly IWebsiteChannelContext _channelContext;

    public ContentQueryService(IContentQueryExecutor executor, IWebsiteChannelContext channelContext)
    {
        _executor = executor;
        _channelContext = channelContext;
    }

    public async Task<IEnumerable<T>> GetContentItemsAsync<T>(
        Action<ContentTypeQueryParameters>? filter = null) where T : IWebPageFieldsSource
    {
        var builder = new ContentItemQueryBuilder()
            .ForContentType(GetContentTypeName<T>(), q => {
                q.ForWebsite(_channelContext.WebsiteChannelName);
                filter?.Invoke(q);
            });
        return await _executor.GetMappedWebPageResult<T>(builder);
    }
}
```

For each content type in `{CLIENT}.Entities`, add typed convenience methods like `GetProductionDetailsAsync()`, `GetVenuesAsync()`, etc. Map every KX13 provider method to a new async method here. Use `SomeType.CONTENT_TYPE_NAME` for query type names — never hardcode `"{CLIENT}.SomeType"` when ClassName is `"{CONTENT_TYPE_NAMESPACE}.SomeType"`.

## Phase 6: EventHandlers

Replace KX13 `DocumentEvents` with xByK `ContentItemEvents`:

```csharp
using CMS;
using CMS.ContentEngine;
using CMS.DataEngine;

namespace {CLIENT}.Business.EventHandlers;

[assembly: RegisterModule(typeof(ContentEventHandlerModule))]

public class ContentEventHandlerModule : Module
{
    public ContentEventHandlerModule() : base(nameof(ContentEventHandlerModule)) { }

    protected override void OnInit(ModuleInitParameters parameters)
    {
        base.OnInit(parameters);
        ContentItemEvents.Publish.Execute += OnPublish;
        ContentItemEvents.Delete.Execute += OnDelete;
        ContentItemEvents.Unpublish.Execute += OnUnpublish;
    }
}
```

## Phase 7: DI Wiring

In `{XBYK_WEB}/Helpers/ServiceCollectionExtensions.cs`:

```csharp
public static void Add{CLIENT}Services(this IServiceCollection services)
{
    services.AddMemoryCache();
    services.AddHttpContextAccessor();
    services.AddHttpClient();
    services.AddSession();

    services.AddSingleton<iCache, TessituraCache>();
    services.AddSingleton(_ => TessituraLogger.GetInstance());
    services.AddScoped<ContentQueryService>();
    // Register all other migrated services as Scoped
}
```

In `{XBYK_WEB}/Program.cs`, initialize configuration before Kentico:

```csharp
var app = builder.Build();
ConfigSettings.Init(app.Configuration);
ServiceFactory.Init(app.Services, app.Environment.EnvironmentName, app.Environment.ContentRootPath);
app.InitKentico();
```

## Phase 8: Build Verification

Build in order and fix iteratively:

```bash
dotnet build {CLIENT}.Entities/{CLIENT}.Entities.csproj
dotnet build {CLIENT}.Business/{CLIENT}.Business.csproj
dotnet build {XBYK_WEB}/{XBYK_WEB}.csproj
```

Target: **0 errors** across all three projects. Iterate until clean.

## Common Error Patterns

| Error | Fix |
|-------|-----|
| `CS0120: object reference required for non-static member` | Service was `static` in KX13, now instance. Add `static Instance` property or pass via DI. |
| `CS0029: Cannot convert TessituraLogger to iLogger` | Make `TessituraLogger` implement `iMedia.Tess.Api.Interfaces.iLogger` |
| `CS0234: 'Entities' does not exist in namespace '{CLIENT}'` | Entity types live under `{CONTENT_TYPE_NAMESPACE}` (e.g. `SCFTA.HomePage`), not `{CLIENT}.Entities.PageContentTypes...` and not necessarily `{CLIENT}.HomePage` |
| `CS0246: type or namespace '{CLIENT}' not found` (for a page type) | Wrong content type namespace — check generated `CONTENT_TYPE_NAME` / `namespace` in Entities and use `{CONTENT_TYPE_NAMESPACE}` |
| `CS0436: type conflicts with imported type` | Delete the compatibility shim — real xByK type exists in `Kentico.Xperience.Core` |
| `CS1061: type missing a property from KX13` | Add property to partial class in `{CLIENT}.Entities` or as extension method |
| `CS1503: WCF constructor string→Binding` | Remove string-based `ClientBase<T>` constructors, keep `(Binding, EndpointAddress)` only |
| `CS0101: duplicate type definition` | Old KX13 file still compiled alongside new file — remove the `<Compile Include>` for KX13 source |
