# Content Type Namespace Contract (Code ↔ Content)

Generated entity classes must match the **registered** content type code names in the XbyK database (`CMS_Class.ClassName`). Those ClassNames are created by content migration (or manual SQL) and are the source of truth for code.

## Rules

1. **ClassName is authoritative.** Format: `{ContentTypeNamespace}.{TypeName}` (e.g. `SCFTA.HomePage`).
2. **C# namespace = ClassName namespace.** Default `--kxp-codegen` uses `{dataClassNamespace}` from each ClassName. Do **not** pass `--namespace {CLIENT}` / `--namespace {ProjectName}` unless that string equals the registered ClassName namespace for every type being generated.
3. **`CONTENT_TYPE_NAME` = full ClassName.** Never invent a different prefix for registration or queries.
4. **Project name ≠ content type namespace.** `{CLIENT}.Entities` may host types in namespace `SCFTA` (or whatever is registered). Assembly/project naming is independent.
5. **Discover before inventing.** Resolve `{CONTENT_TYPE_NAMESPACE}` from, in order: (a) migration plan target ClassNames, (b) existing generated `CONTENT_TYPE_NAME` constants in `*.Entities`, (c) `CMS_Class.ClassName` in the XbyK database. Do not default to `{CLIENT}`.
6. **Preserve KX13 namespaces by default.** If content migration kept `SCFTA.*`, all code skills use `SCFTA`, not `{CLIENT}`.
7. **Custom / manual types follow the same namespace** as already-migrated page types.

## Verification (required after codegen and after hand-written entities)

For each content type used in code:

- Generated `CONTENT_TYPE_NAME` == `CMS_Class.ClassName`
- C# `namespace` == ClassName segment before the last `.`
- Business/page references use that namespace (e.g. `SCFTA.HomePage`, not `Segerstrom.HomePage` when ClassName is `SCFTA.HomePage`)

## Codegen command pattern

```bash
# Preferred: let each ClassName drive its C# namespace
dotnet run --no-build -- --kxp-codegen --type "PageContentTypes" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"
dotnet run --no-build -- --kxp-codegen --type "ReusableContentTypes" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"

# Reusable field schemas MUST use the same C# namespace as the content type classes
dotnet run --no-build -- --kxp-codegen --type "ReusableFieldSchemas" --namespace "{CONTENT_TYPE_NAMESPACE}" --skip-confirmation --location "../{ProjectName}.Entities/{type}/{name}"
```

If multiple ClassName namespaces exist, generate without a single overriding `--namespace` for content types (default `{dataClassNamespace}`), and generate schema interfaces once per namespace used by those types.
