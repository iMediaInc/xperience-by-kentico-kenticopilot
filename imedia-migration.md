# KX 13 -> XbyK upgrade path

## set up your project
1. create a new repo
2. move the legacy Kentico 13 code to the `/kx13` directory.
3. create a new `/xbyk` directory. Install xByK here. make sure that the version is supported by the content migration application. see the [README](./plugins/kx13-content-migration/README.md)
4. create an audit-results folder
5. install the kentico migration tool. see the deatils in the [migration readme](./plugins/kx13-content-migration/README.md)

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
1. Add a connection string to the content Auditor cli: see directions [here:](./plugins/kx13-content-audit#set-up-the-auditor-source) 
2. build the content auditor solution
3. run `/kx13-content-audit` from the agent window. this will output the content audit data into the `/audit-results` folder.

## Run content migration
1. run the migration planning
``` 
/migrate-plan

Produce a migration plan from the JSON output in ./audit-results
```

2. validate the model. for example for SCFTA I needed to tell it:
```
For any pages that don't inherit from SCFTA.BasePage, lets assume that we want to migrate that to a new content type in xByK. Update the migration plans to do that.
```
you might need to iterate on this.

3. generate the migration tool's app settings.
```
/migrate-appsettings

The plan in ./migration-detail.md is ready. Generate the migration
tool's appsettings.json from it.
```

4. generate the IClassMappings
```
/migrate-classes

Generate the IClassMapping and ReusableSchemaBuilder C# extensions
for the page types and reusable field schemas described in
./migration-detail.md.
```

5. migrate the fields
```
/migrate-fields

Generate the IFieldMigration extensions for the cross-class field
transforms in ./migration-detail.md (HTML sanitization, URL rewrites,
and the legacy form-control conversions the plan flags).
```

6. migrate the widgets
```
/migrate-widgets

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
```
/migrate-run

Migration.Tool.Extensions builds clean and the KX13 source app is
running. Execute the migration end-to-end against the configured
target database following ./migration-detail.md.
```
```
/migrate-eval

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

