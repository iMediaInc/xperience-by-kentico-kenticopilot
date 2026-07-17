---
name: migrate-non-kentico-controllers
description: >-
  Migrates a .NET Framework 4.8 / Kentico Xperience 13 business layer to .NET 10
  / Xperience by Kentico. Rewrites all CMS, System.Web, and KX13 API usage to
  xByK and ASP.NET Core equivalents while preserving Tessitura/iMedia SDK calls.
  Use when migrating business layer code, services, extensions, models, or event
  handlers from KX13 to xByK.
---

# Migrate Non-kentico Controllers (KX13 → xByK)

## Parameters

Before starting, collect these values from the user. Use them everywhere you see `{PARAM}` placeholders below.

| Parameter | Description | Example |
|-----------|-------------|---------|
| `{CLIENT}` | Client/project name used in **project/assembly** names | `Segerstrom` |
| `{CONTENT_TYPE_NAMESPACE}` | Entity ClassName / C# namespace from registered content types | `SCFTA` |
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

Resolve `{CONTENT_TYPE_NAMESPACE}` from Entities `CONTENT_TYPE_NAME` constants / migration plan / `CMS_Class` — not from `{CLIENT}`. See [content-type-namespaces.md](../_shared/references/content-type-namespaces.md).

## Prerequisites

- Source KX13 project at `{KX13_SOURCE}/`
- Target xByK web project at `{XBYK_WEB}/`
- `{CLIENT}.Entities` project with xByK-generated content types in the `{CONTENT_TYPE_NAMESPACE}` namespace

## Migration Steps

1. Read all documentation links mentioned above.
2. Identify any controller actions that are not directly related to kentico page or widget types.
3. Migrate those controllers and actions from the KX13 solution to the XbyK solution.
4. Iterate through all of the utility controllers and actions until everything has been ported over. Do not stub out any actions. do not skip any actions. Every action must have a completed, ported counterpart in XbyK.
5. Build in order and fix iteratively:

```bash
dotnet build {CLIENT}.Entities/{CLIENT}.Entities.csproj
dotnet build {CLIENT}.Business/{CLIENT}.Business.csproj
dotnet build {XBYK_WEB}/{XBYK_WEB}.csproj
```
Target: **0 errors** across all three projects. Iterate until clean.
6. When you are done, create a new file in the ./migration directory called controllers.md that details what controllers and actions were ported over, which were skipped because they are kentico-related, and if they were skipped, what skill in this repo needs to be used to migrate them.

Whenever unsure about anything, you can use Kentico Docs MCP to search for relevant information. 


## Common Error Patterns

| Error | Fix |
|-------|-----|
| `CS0120: object reference required for non-static member` | Service was `static` in KX13, now instance. Add `static Instance` property or pass via DI. |
| `CS0029: Cannot convert TessituraLogger to iLogger` | Make `TessituraLogger` implement `iMedia.Tess.Api.Interfaces.iLogger` |
| `CS0234: 'Entities' does not exist in namespace '{CLIENT}'` | Use `{CONTENT_TYPE_NAMESPACE}.TypeName` (from Entities), not `{CLIENT}.Entities.PageContentTypes...` and not `{CLIENT}.TypeName` unless ClassNames were remapped to `{CLIENT}` |
| `CS0436: type conflicts with imported type` | Delete the compatibility shim — real xByK type exists in `Kentico.Xperience.Core` |
| `CS1061: type missing a property from KX13` | Add property to partial class in `{CLIENT}.Entities` or as extension method |
| `CS1503: WCF constructor string→Binding` | Remove string-based `ClientBase<T>` constructors, keep `(Binding, EndpointAddress)` only |
| `CS0101: duplicate type definition` | Old KX13 file still compiled alongside new file — remove the `<Compile Include>` for KX13 source |
