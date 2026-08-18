# KX 13 -> XbyK upgrade path

## set up your project
1. create a new repo
2. move the legacy Kentico 13 code to the `/kx13` directory.
3. create a new `/xbyk` directory. Install xByK here. make sure that the version is supported by the content migration application. see the [README](./plugins/kentico-kx13-migration/README.md)
4. create an audit-results folder

5. install the kentico migration tool. see the deatils in the [migration readme](./plugins/kentico-kx13-migration/README.md)

your workspace should look like this:

```
<workspace-root>/
├── KX13/                            # KX13 source project 
├── XbyK/                            # XbyK target project 
├── audit-results/                   # Optional: kx13-content-audit JSON + report
├── kentico-migration-tool/
│   ├── Migration.Tool.CLI/          # appsettings.json is generated here
│   └── Migration.Tool.Extensions/   # Generated C# extensions are placed here
└── MigrationProtocol/               # Created by migrate-run; consumed by migrate-eval
```

### NOTE: USE A GOOD MODEL. 

## run the content audit
1. Add a connection string to the content Auditor cli: see directions [here:](./plugins/kentico-kx13-migration#set-up-the-auditor-source) 
2. build the content auditor solution
3. run `/migrate-content-audit` from the agent window. this will output the content audit data into the `/audit-results` folder.

## Run content migration
1. run the migration planning
``` 
/migrate-content-plan

Produce a migration plan from the JSON output in ./audit-results
```

2. validate the model. for example for SCFTA I needed to tell it:
```
For any pages that don't inherit from SCFTA.BasePage, lets assume that we want to migrate that to a new content type in xByK. Update the migration plans to do that.
```
you might need to iterate on this.

3. generate the migration tool's app settings.
```
/migrate-content-appsettings

The plan in ./migration-detail.md is ready. Generate the migration
tool's appsettings.json from it.
```

4. generate the IClassMappings
```
/migrate-content-classes

Generate the IClassMapping and ReusableSchemaBuilder C# extensions
for the page types and reusable field schemas described in
./migration-detail.md.
```

5. migrate the fields
```
/migrate-content-fields

Generate the IFieldMigration extensions for the cross-class field
transforms in ./migration-detail.md (HTML sanitization, URL rewrites,
and the legacy form-control conversions the plan flags).
```

6. migrate the widgets
```
/migrate-content-widgets

Generate the IWidgetMigration and IWidgetPropertyMigration extensions
for the custom widgets that ./migration-detail.md flags for transforms.
```

7. migrate the content items
```
/migrate-content-items

Generate the ContentItemDirectorBase extensions for the linked-page
strategies, child-as-reference linking, and page-to-widget conversions
in ./migration-detail.md.
```

8. loop on running and evaluating the content migration until everything works well.
*make sure you've got the media library in the kx13 directory or things will not go well*
```
/migrate-content-run

Migration.Tool.Extensions builds clean and the KX13 source app is
running. Execute the migration end-to-end against the configured
target database following ./migration-detail.md.
```
```
/migrate-content-eval

migrate-run finished. Compare the migrated XbyK database against
./migration-detail.md and produce the HTML report at ./migration-eval.html  so I know what to
fix and which sibling skill to re-run for each finding.
```
```
Based on the results in ./migration-eval.html, fix the critical issues in the migration.
```
... rerun eval, fix, repeat


## migrate custom tables
This will migrate custom tables into Kentico Content Types.
In the below prompt, substitute the prefix of the tables in the legacy and target (the target will be the namespace in XbyK)
```
For any of the custom tables that haven't been migrated yet, run the /migrate-custom-tables skill. The legacy prefix is {legacy prefix} and the ProjectName is {target prefix}
```
## migrate business layer
This will migrate the business layer from .net framework to .net core and upgrade the app to use XByK references. Here's the parameters:
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{CLIENT}` | Client/project name used in namespaces and project names | `Segerstrom` |
| `{KX13_SOURCE}` | Relative path to the KX13 business layer project | `kx13/Segerstrom.Business` |
| `{XBYK_WEB}` | Relative path to the xByK web project | `xbyk` |
| `{KX13_CODENAME_PREFIX}` | KX13 document type code name prefix (usually the site code) | `SCFTA` |
| `{TESS_API_VERSION}` | iMedia.Tess.Api NuGet package version | `16.0.16` |
| `{KENTICO_CORE_VERSION}` | Kentico.Xperience.Core NuGet package version | `31.4.3` |
| `{ALGOLIA_VERSION}` | Algolia.Search NuGet package version (omit if not used) | `6.10.2` |
| `{HAS_ALGOLIA}` | Whether project uses Algolia search | `true` / `false` |
| `{HAS_WCF_SOAP}` | Whether project has WCF/SOAP Connected Services | `true` / `false` |
| `{HAS_DYNAMIC_PDF}` | Whether project uses ceTe.DynamicPDF | `true` / `false` |
| `{EXTRA_PACKAGES}` | Additional NuGet packages the business layer needs | `log4net`, `Kount.Net.RisSDK`, etc. |



```
Migrate the business layer using the /migrate-business-logic skill. 
parameters: 
 - Client:{Client eg LAOpera}
 - ...
```

## Migrate pages
This is a multi-step process.

```
You are the migration orchestrator for a Kentico Xperience 13 to Xperience by Kentico page migration.

Your job is to plan and coordinate only. Do not migrate pages yourself unless explicitly asked.

Use the available KentiCopilot skills and repository context to:

1. Identify the source KX13 page templates, controllers, views, repositories, widgets, sections, and shared components.
2. Create a page inventory.
3. Group pages into four worker-safe batches.
4. Detect likely shared components and assign ownership so workers do not edit the same files independently.
5. Write all output to migration/_control/.
6. Create clear worker task files.
7. Create validation criteria for each page.

Rules:
- Do not edit page implementation files.
- Do not make broad refactors.
- Do not let two workers own the same shared component.
- Every page task must include source files, target files, dependencies, expected route, expected widgets, and validation notes.
- Prefer incremental page-by-page migration over large global changes.
```

when that's done, run the below to handle shared component migration:
```
You are the shared data worker orchestrator.
Create 4 subagents based on the identifiers made in the previous step. Each subagent should get this direction:

You are Worker {Subagent identifier} in a Kentico Xperience 13 to Xperience by Kentico migration.

Read these files first:
- migration/_control/migration-plan.md
- migration/_control/worker-assignments.md
- migration/_control/shared-component-ownership.md
- migration/_control/page-status/worker-{subagent identifier}.md

create a git worktree for yourself by running `git worktree add ./migration-worker-{subagent identifier} -b migration/worker-{subagent identifier}`

Only migrate pages assigned to Worker {subagent identifier}.

Use the KentiCopilot codebase migration skills where appropriate:
- migrate-page for page controller/view/repository/dependency migration
- migrate-page-widgets for page-specific Page Builder widgets and sections
- migrate-shared-component only for shared components explicitly assigned to Worker {subagent identifier}

Rules:
- Do not edit pages assigned to other workers.
- Do not edit shared components unless Worker {subagent identifier} owns them.
- Do not make unrelated formatting changes.
- Preserve behavior, routing, field mappings, SEO/meta behavior, localization, and widget behavior.
- After each page, update migration/_control/page-status/worker-a.md.
- Include source files changed, target files changed, assumptions, unresolved issues, and validation steps.
- Run build/tests if available and record the result.
```

now we bulk migrate pages.
substitute the `legacy site url` and `main page type` below.
```
you are the page migrator for a Kentico XPerience 13-> XbyK migration. 

1. Gather a list of all page urls from KX13 to migrate. store those in a file in migration/_control. 
Break that list into 10 worker processes. use the kentico MCP and the KxConnectionString from Migration.Tool.CLI/appsettings.json to help you get the page urls.
2. Start 10 subagents. have each worker process create their own git worktree. Each should take it's own list of urls from the list we generated in the first step. each subagent should run  /migrate-page-widgets  and /migrate-page  for that given url. The url for the legacy site is {legacy url}.  for now, focus on the "{main Page type}" page type pages.
3. when a worker is done with the migration of a page, use the /migrate-page-visual skill to take a snapshot of the page on both the xbyk site and the KX13 site and use the playwright-mcp to generate a differential. if more than 20% of the pixels are off, then flag that item in the url list we created above for further review.
4. When done with the first pass, let me know what pages have issues.
5. Run through the pages that have errors from the first pass. For each one use the /migrate-page-visual skill to take a snapshot of the page on both the xbyk site and the KX13 site and use the playwright-mcp to generate a differential. If more than 20% of the pixels are off, then flag that item in the url list we created above for further review. If there is still an issue, assign the page to it's particular worker. That worker should use /migrate-page-widgets and /migrate-page to fix the page. run /migrate-page-visual again when done, take the snapshot, and flag again if it still doesn't look right.
6. Generate another report with the final results after the 2nd pass.
```