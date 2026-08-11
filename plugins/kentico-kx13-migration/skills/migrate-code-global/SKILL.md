---
name: "migrate-code-global"
description: "Migrates global code from a Kentico Xperience 13 project to Xperience by Kentico. Sets up the target project structure, generates code files, and migrates shared code including localization, styles, business logic, and project startup configuration. Use when starting a KX13 to XbyK codebase migration."
compatibility: "Requires Kentico Docs MCP"
---

You are tasked with the process of migrating global code from a Kentico Xperience 13 project to an Xperience by Kentico project.

## Structure of the projects

You are currently located in the root folder, which contains two subfolders:

- `KX13/` - This folder contains the Kentico Xperience 13 project files. This is the legacy/source project.
- `XbyK/` - This folder contains the Xperience by Kentico project files. This is the new project.

## Content type namespaces (read first)

Before creating the Entities project or running codegen, read [content-type-namespaces.md](../_shared/references/content-type-namespaces.md).

Registered `CMS_Class.ClassName` values (from content migration) are the source of truth. Discover `{CONTENT_TYPE_NAMESPACE}` from, in order:

1. Migration plan **Content Type Namespace Convention** / target ClassNames (`migration-overview.md` / `migration-detail.md` if present)
2. Existing `CMS_Class.ClassName` values in the XbyK database
3. KX13 page type ClassName prefixes only if content has not been migrated yet

**Do not** assume `{CONTENT_TYPE_NAMESPACE}` equals the .NET `{ProjectName}` / client name. Example: project `Segerstrom.Entities` may contain types in namespace `SCFTA` when ClassNames are `SCFTA.*`.

## Migration Steps

1. Review the structure of both the legacy and new project.
2. Use Kentico Docs MCP to read the following page: <https://docs.kentico.com/guides/upgrade-to-xbyk/upgrade-walkthrough/adjust-global-code> (note that this guide is written for a sample project and that there will be some differences between the sample project and the project you are migrating)
3. Resolve `{CONTENT_TYPE_NAMESPACE}` per the section above. If multiple ClassName namespaces exist, note all of them.
4. Create a new project for generated code files (named `{ProjectName}.Entities`).
   1. Configure given project as described in the documentation.
   2. **CRITICAL:** Ensure the .csproj file contains the following (without this, content item reference fields will fail to populate):

      ```xml
      <ItemGroup>
        <AssemblyAttribute Include="CMS.AssemblyDiscoverableAttribute" />
      </ItemGroup>
      ```
5. Generate code files by running the `--kxp-codegen` command as described in the documentation. Always use `--skip-confirmation` flag to avoid interactive prompts.
   1. **Preferred for PageContentTypes / ReusableContentTypes / EmailContentTypes:** omit `--namespace` so each type uses its ClassName namespace (`{dataClassNamespace}`). Do **not** pass `--namespace {ProjectName}` unless `{ProjectName}` equals `{CONTENT_TYPE_NAMESPACE}` for every generated type.
   2. For `ReusableFieldSchemas`, `--namespace` is required — set it to the same C# namespace as the content type classes that implement those schemas (usually `{CONTENT_TYPE_NAMESPACE}`). If content types span multiple namespaces, generate schema interfaces for each namespace used.
   3. Example (page + reusable types without overriding namespace):
      ```bash
      dotnet run --no-build -- --kxp-codegen --type "PageContentTypes" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"
      dotnet run --no-build -- --kxp-codegen --type "ReusableContentTypes" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"
      dotnet run --no-build -- --kxp-codegen --type "ReusableFieldSchemas" --namespace "{CONTENT_TYPE_NAMESPACE}" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"
      ```
6. **Verify namespace alignment** after codegen (required):
   1. Spot-check generated files: `CONTENT_TYPE_NAME` must equal `CMS_Class.ClassName` for that type.
   2. C# `namespace` declaration must equal the ClassName namespace segment (e.g. `SCFTA` for `SCFTA.HomePage`).
   3. If any generated file uses the wrong namespace or wrong `CONTENT_TYPE_NAME`, delete the bad outputs, fix the codegen parameters, and regenerate. Do not hand-edit generated files to "fix" a wrong `--namespace`.
7. Copy relevant global code from source project to the new project.
   1. Localization
   2. Shared views
   3. Styles and scripts
   4. Identifiers
   5. Services registration
8. Configure the project to display content.
   1. Enable Content tree-based routing and Page Builder.
   2. Add future custom service registrations and localization.
9. Ensure that the new project builds successfully without errors and warnings. If it doesn't, some part of the documentation was not followed correctly. Fix the issues based on the docs page.

Notes relevant to the migration process:

- Do not change any other files or settings outside of the global code migration process (that are not mentioned in the docs page).
- When commenting out code, always add "TODO:" note to make it easier to find later.
- Later skills (migrate-page, migrate-business-logic, migrate-custom-tables) must reference entity types under the verified content type namespace, not invent `{ProjectName}.*` ClassNames.

## Output format

When done, provide the user with this exact output (without any additional text):

```markdown
# Migration Complete
Global code migration from the legacy project to the new one has been successfully completed.

**Next steps:**
- Update your channel configuration to include the port of the local XbyK instance (the one the project launches with).
    Follow these steps: https://docs.kentico.com/guides/upgrade-to-xbyk/upgrade-walkthrough/adjust-global-code#adjust-system-url
- Review the changes to ensure everything looks as expected.
- Continue with the migrate-code-page-widgets skill to migrate Page Builder widgets used by the specified page.
```
