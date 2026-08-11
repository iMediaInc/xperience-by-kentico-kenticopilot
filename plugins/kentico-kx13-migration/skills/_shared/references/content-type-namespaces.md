# Content Type Namespace Contract (Content ↔ Code)

Page types and reusable content types have a **registered code name** (`CMS_Class.ClassName`) of the form `{ContentTypeNamespace}.{TypeName}` (e.g. `SCFTA.HomePage`). Generated C# model classes must stay aligned with that registration at all times.

## Source of truth

| Concept | Authority | Example |
| --- | --- | --- |
| Content type code name | `CMS_Class.ClassName` (and migration plan target ClassName) | `SCFTA.HomePage` |
| Content type namespace | Segment of ClassName before the last `.` | `SCFTA` |
| Type name | Segment of ClassName after the last `.` | `HomePage` |
| C# model namespace | Must equal content type namespace (default codegen) | `namespace SCFTA` |
| `CONTENT_TYPE_NAME` / `RegisterContentTypeMapping` | Must equal full ClassName exactly | `"SCFTA.HomePage"` |
| Project / assembly name | Independent — may differ from content type namespace | `Segerstrom.Entities` |

**Never equate project name (`{CLIENT}`, `{ProjectName}`, `.csproj` root) with content type namespace** unless the migration plan explicitly remaps ClassNames to that project name.

## Default convention (preserve KX13)

Unless the user or target model explicitly remaps namespaces:

1. Target ClassName **preserves** the KX13 ClassName namespace (`SCFTA.Article` → `SCFTA.Article`, not `Segerstrom.Article`).
2. `IClassMapping` sets `target.ClassName` / `ClassTableName` from that target ClassName (`SCFTA_Article`).
3. `--kxp-codegen` for page/reusable content types **omits** `--namespace`, or uses `--namespace "{dataClassNamespace}"`, so the C# namespace is taken from each ClassName.
4. Business / page code references entities as `{ContentTypeNamespace}.{TypeName}` (e.g. `SCFTA.HomePage`), never `{CLIENT}.{TypeName}` when those differ.
5. Custom tables / manual content types created during code migration use the **same** `{ContentTypeNamespace}` as already-migrated page types.

## When remapping namespaces

If the plan renames namespaces (e.g. `SCFTA.*` → `Segerstrom.*`):

1. Document every source ClassName → target ClassName in the migration plan.
2. Every `IClassMapping` must use the **target** ClassName.
3. Codegen `--namespace` (if used) must match the **target** content type namespace, not an unrelated project name.
4. All later code skills must use the target namespace for entity types and `CONTENT_TYPE_NAME` strings.

Partial remaps (some types keep `SCFTA`, others become `Segerstrom`) are allowed only when listed explicitly; codegen must not force a single wrong `--namespace` over mixed ClassNames — prefer default `{dataClassNamespace}` per type.

## Verification checklist

After content migration and after codegen, confirm for each content type:

1. `CMS_Class.ClassName` == plan target ClassName
2. Generated `CONTENT_TYPE_NAME` == `CMS_Class.ClassName`
3. Generated C# `namespace` == ClassName namespace segment
4. Business/page code `using` / type references use that same C# namespace
5. Queries use `TypeName.CONTENT_TYPE_NAME` (or the identical string), never a guessed `{CLIENT}.{TypeName}`

## Anti-patterns

| Anti-pattern | Why it breaks |
| --- | --- |
| `--kxp-codegen --namespace {CLIENT}` when ClassNames are still `{KX13Prefix}.*` | C# types live under the wrong namespace; content retrieval / mapping still keys off ClassName |
| Hand-written entities with `CONTENT_TYPE_NAME = "{ProjectName}.{Type}"` while DB has `{KX13Prefix}.{Type}` | `RegisterContentTypeMapping` does not match registered types |
| Business skill maps `CMS.DocumentEngine.Types.SCFTA.*` → `{CLIENT}.*` without checking Entities | Compile-time or runtime type mismatches |
| Custom-table migration invents a new ClassName namespace different from migrated page types | Split model; inconsistent codegen and queries |
| Plan lists target types as `Namespace.Type` but class mappings / SQL use a different namespace | Content and code diverge permanently |
