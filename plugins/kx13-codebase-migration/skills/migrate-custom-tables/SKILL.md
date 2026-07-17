---
name: migrate-custom-table
description: >-
  Migrates a KX13 custom table to an xByK content hub content type. Creates the
  CMS_Class registration, backing SQL table, C# entity class, content hub folder,
  and migrates all data rows. Use when the user asks to migrate a custom table,
  create a content type from a custom table, or move a custom table data to xByK.
compatibility: Requires Kentico Docs MCP
---

# Migrate KX13 Custom Table to xByK Content Hub Content Type

## Overview

This skill migrates a single KX13 custom table (e.g. `SCFTA.ConfigSettings`) into a fully
functional xByK **Reusable** content hub content type. It produces:

1. A `CMS_Class` row (content type registration)
2. A backing SQL table (`{CONTENT_TYPE_NAMESPACE}_{TypeName}`)
3. A C# entity class in `{ProjectName}.Entities/ReusableContentTypes/{TypeName}/`
4. A content hub workspace (existing or newly created) and folder for the items
5. Migrated data rows as published content items

## Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `{ProjectName}` | .NET Entities **project** name (assembly/folder) | `Segerstrom` |
| `{CONTENT_TYPE_NAMESPACE}` | ClassName / C# namespace — must match migrated page types | `SCFTA` |
| `{TypeName}` | Content type name segment | `AppConfigSetting` |

Resolve `{CONTENT_TYPE_NAMESPACE}` from the migration plan, existing `CONTENT_TYPE_NAME` constants in `{ProjectName}.Entities`, or `CMS_Class.ClassName` for already-migrated page types. **Do not default it to `{ProjectName}`** when page types use a different namespace. See [content-type-namespaces.md](../_shared/references/content-type-namespaces.md).

## Structure of the projects

You are currently located in the root folder, which contains two subfolders:

- `KX13/` - This folder contains the Kentico Xperience 13 project files. This is the legacy/source project.
- `XbyK/` - This folder contains the Xperience by Kentico project files. This is the new project.


## Prerequisites

- **xByK database**: get the connection string from the XbyK application
- **KX13 database**: get the connection string from the KX13 application
- **KX13 custom tables** live in the xByK database as `{LegacyPrefix}_*` tables (already migrated by the migration tool)
- **Workspace**: resolve via Step 1 (do **not** hardcode `KenticoDefault` / ID `1`)
- **Default language ID**: query `CMS_ContentLanguage WHERE ContentLanguageIsDefault = 1`
- **Default user**: query `CMS_User WHERE UserName = 'administrator'`
- **Entity project**: `{ProjectName}.Entities/{ProjectName}.Entities.csproj` (targets `net10.0`, uses `CMS.AssemblyDiscoverableAttribute`)
- **Namespace alignment**: new types use `{CONTENT_TYPE_NAMESPACE}.{TypeName}` so they match page-type ClassNames already in the database

## Step-by-step Workflow

### Step 1: Resolve Target Workspace

**STOP and prompt the user** before creating folders or migrating data. Workspaces own content hub folders and content items; every migrated item must reference the chosen `@WorkspaceID`.

1. List existing workspaces from the xByK database:

```sql
SELECT WorkspaceID, WorkspaceName, WorkspaceDisplayName, WorkspaceGUID
FROM CMS_Workspace
ORDER BY WorkspaceID;
```

2. Ask the user which option they want:
   - **Use an existing workspace** — they pick one from the list (by `WorkspaceName` or `WorkspaceID`)
   - **Create a new workspace** — ask for a code name (`WorkspaceName`, e.g. `Segerstrom`) and display name (`WorkspaceDisplayName`)

3. Resolve `@WorkspaceID` from `CMS_Workspace`. If creating (or the requested name is missing), insert the row only when it does not already exist:

```sql
DECLARE @WorkspaceName NVARCHAR(100) = N'{WorkspaceName}';
DECLARE @WorkspaceDisplayName NVARCHAR(200) = N'{WorkspaceDisplayName}';
DECLARE @WorkspaceID INT;

SELECT @WorkspaceID = WorkspaceID
FROM CMS_Workspace
WHERE WorkspaceName = @WorkspaceName;

IF @WorkspaceID IS NULL
BEGIN
    INSERT INTO CMS_Workspace (
        WorkspaceDisplayName,
        WorkspaceName,
        WorkspaceGUID
    ) VALUES (
        @WorkspaceDisplayName,
        @WorkspaceName,
        NEWID()
    );

    SET @WorkspaceID = SCOPE_IDENTITY();
END

SELECT @WorkspaceID AS WorkspaceID, @WorkspaceName AS WorkspaceName;
```

`CMS_Workspace` columns used here:

| Column | Notes |
|--------|-------|
| `WorkspaceID` | Identity PK — store as `@WorkspaceID` for later steps |
| `WorkspaceName` | Code name (unique) |
| `WorkspaceDisplayName` | Admin UI label |
| `WorkspaceGUID` | `NEWID()` |

4. Ensure the workspace has a **root content folder** (required for Content hub). Query and create if missing:

```sql
DECLARE @RootFolderID INT;

SELECT @RootFolderID = ContentFolderID
FROM CMS_ContentFolder
WHERE ContentFolderWorkspaceID = @WorkspaceID
  AND ContentFolderParentFolderID IS NULL;

IF @RootFolderID IS NULL
BEGIN
    INSERT INTO CMS_ContentFolder (
        ContentFolderName, ContentFolderDisplayName, ContentFolderGUID,
        ContentFolderCreatedByUserID, ContentFolderCreatedWhen, ContentFolderModifiedWhen,
        ContentFolderTreePath, ContentFolderParentFolderID, ContentFolderWorkspaceID
    ) VALUES (
        N'{WorkspaceName}', N'{WorkspaceDisplayName}', NEWID(),
        @DefaultUserId, GETUTCDATE(), GETUTCDATE(),
        N'/', NULL, @WorkspaceID
    );

    SET @RootFolderID = SCOPE_IDENTITY();
END

-- Store @RootFolderID for Step 7 (folder parent)
```

5. Carry `@WorkspaceID` and `@RootFolderID` through all later SQL. Never fall back to hardcoding `1`.

### Step 2: Examine the KX13 Custom Table

Query the source table schema and data:

```sql
-- Get column definitions (skip system columns: ItemID, ItemCreatedBy, ItemCreatedWhen,
-- ItemModifiedBy, ItemModifiedWhen, ItemOrder, ItemGUID)
SELECT c.name, t.name AS type_name, c.max_length, c.is_nullable
FROM sys.columns c
JOIN sys.types t ON t.system_type_id = c.system_type_id AND t.user_type_id = c.user_type_id
JOIN sys.tables tb ON tb.object_id = c.object_id
WHERE tb.name = '{SCFTA_TableName}'
ORDER BY c.column_id;

-- Preview data
SELECT * FROM {SCFTA_TableName};
```

Identify the **user columns** (skip the `Item*` system columns). These become content type fields.

### Step 3: Choose the xByK Type Name

- Pick a ClassName like `{CONTENT_TYPE_NAMESPACE}.{TypeName}` (e.g. `SCFTA.AppConfigSetting` when page types are `SCFTA.*` — **not** `Segerstrom.AppConfigSetting` unless the plan remapped namespaces to `Segerstrom`)
- The C# class will be `{TypeName}` in namespace `{CONTENT_TYPE_NAMESPACE}`
- **CRITICAL**: Check for name collisions with existing classes in `{ProjectName}.Business` and
  the Tessitura SDK. Search the codebase: `class {TypeName}`. If there's a collision, pick a
  different name (prefix with `App`, `Site`, etc.)

### Step 4: Create the Content Type (SQL)

Write a SQL script to `scripts/create-{typename}.sql`. Use `sqlcmd -i` to execute (avoids
PowerShell quoting issues with XML).

The script must create three things in this order:

#### 4a. CMS_Class row

Required columns and their values:

| Column | Value |
|--------|-------|
| `ClassName` | `{CONTENT_TYPE_NAMESPACE}.{TypeName}` |
| `ClassDisplayName` | Human-readable name |
| `ClassTableName` | `{CONTENT_TYPE_NAMESPACE}_{TypeName}` |
| `ClassXmlSchema` | See template below |
| `ClassFormDefinition` | See template below |
| `ClassType` | `Content` |
| `ClassContentTypeType` | `Reusable` |
| `ClassGUID` | `NEWID()` |
| `ClassResourceID` | `NULL` |
| `ClassHasUnmanagedDbSchema` | `0` |
| `ClassWebPageHasUrl` | `0` |
| `ClassLastModified` | `GETUTCDATE()` |
| `ClassShortName` | `{CONTENT_TYPE_NAMESPACE}{TypeName}` |

#### ClassXmlSchema template

```
<?xml version="1.0" encoding="utf-8"?>
<xs:schema id="NewDataSet" xmlns="" xmlns:xs="http://www.w3.org/2001/XMLSchema"
  xmlns:msdata="urn:schemas-microsoft-com:xml-msdata">
  <xs:element name="NewDataSet" msdata:IsDataSet="true" msdata:UseCurrentLocale="true">
    <xs:complexType><xs:choice minOccurs="0" maxOccurs="unbounded">
      <xs:element name="{TableName}"><xs:complexType><xs:sequence>
        <xs:element name="ContentItemDataID" msdata:ReadOnly="true"
          msdata:AutoIncrement="true" type="xs:int" />
        <xs:element name="ContentItemDataCommonDataID" type="xs:int" />
        <xs:element name="ContentItemDataGUID"
          msdata:DataType="System.Guid, System.Private.CoreLib, Version=8.0.0.0,
          Culture=neutral, PublicKeyToken=7cec85d7bea7798e" type="xs:string" />
        <!-- Add one xs:element per user field -->
      </xs:sequence></xs:complexType></xs:element>
    </xs:choice></xs:complexType>
    <xs:unique name="Constraint1" msdata:PrimaryKey="true">
      <xs:selector xpath=".//{TableName}" />
      <xs:field xpath="ContentItemDataID" />
    </xs:unique>
  </xs:element>
</xs:schema>
```

XSD type mapping:

| SQL type | XSD element |
|----------|-------------|
| `nvarchar(N)` | `<xs:element name="Col" minOccurs="0"><xs:simpleType><xs:restriction base="xs:string"><xs:maxLength value="N" /></xs:restriction></xs:simpleType></xs:element>` |
| `nvarchar(max)` | Same but `maxLength value="2147483647"` |
| `int` | `<xs:element name="Col" type="xs:int" />` |
| `bit` | `<xs:element name="Col" type="xs:boolean" />` |
| `datetime2` | `<xs:element name="Col" type="xs:dateTime" />` |
| `bigint` / `long` | `<xs:element name="Col" type="xs:long" />` |

Add `minOccurs="0"` for nullable columns.

#### ClassFormDefinition template

Every form starts with three system fields, then user fields:

```xml
<form>
  <field column="ContentItemDataID" columntype="integer" enabled="true"
    guid="{NEWGUID}" isPK="true" />
  <field column="ContentItemDataCommonDataID" columntype="integer" enabled="true"
    guid="{NEWGUID}" refobjtype="cms.contentitemcommondata" reftype="Required" system="true" />
  <field column="ContentItemDataGUID" columntype="guid" enabled="true"
    guid="{NEWGUID}" isunique="true" system="true" />
  <!-- User fields follow -->
</form>
```

User field template:

```xml
<field allowempty="{true|false}" column="{ColumnName}" columnsize="{N}"
  columntype="{type}" enabled="true" guid="{NEWGUID}" visible="true">
  <properties><fieldcaption>{Display Name}</fieldcaption></properties>
  <settings><controlname>{AdminControl}</controlname></settings>
</field>
```

Form field type mapping:

| SQL type | `columntype` | `columnsize` | `controlname` |
|----------|-------------|-------------|--------------|
| `nvarchar(N)` | `text` | `N` | `Kentico.Administration.TextInput` |
| `nvarchar(max)` | `longtext` | omit | `Kentico.Administration.TextArea` |
| `int` | `integer` | omit | `Kentico.Administration.NumberInput` |
| `bit` | `boolean` | omit | `Kentico.Administration.Checkbox` |
| `datetime2` | `datetime` | omit | `Kentico.Administration.DateTimeInput` |
| `bigint` | `longinteger` | omit | `Kentico.Administration.NumberInput` |

- Set `allowempty="true"` for nullable columns
- Omit `allowempty` (or set `false`) for required columns
- For `boolean` fields, add `<properties><defaultvalue>false</defaultvalue>...</properties>`

#### 4b. Backing SQL table

```sql
CREATE TABLE {CONTENT_TYPE_NAMESPACE}_{TypeName} (
    ContentItemDataID           INT              NOT NULL IDENTITY(1,1) PRIMARY KEY,
    ContentItemDataCommonDataID INT              NOT NULL,
    ContentItemDataGUID         UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),
    -- user columns here (use [brackets] for SQL reserved words like [Key], [Value], [Order]) --
    CONSTRAINT FK_{CONTENT_TYPE_NAMESPACE}_{TypeName}_CommonData
        FOREIGN KEY (ContentItemDataCommonDataID)
        REFERENCES CMS_ContentItemCommonData (ContentItemCommonDataID)
);

CREATE UNIQUE NONCLUSTERED INDEX UQ_{CONTENT_TYPE_NAMESPACE}_{TypeName}_GUID
    ON {CONTENT_TYPE_NAMESPACE}_{TypeName} (ContentItemDataGUID);
CREATE NONCLUSTERED INDEX IX_{CONTENT_TYPE_NAMESPACE}_{TypeName}_CommonDataID
    ON {CONTENT_TYPE_NAMESPACE}_{TypeName} (ContentItemDataCommonDataID);
```

### Step 5: Create the C# Entity Class

Create `{ProjectName}.Entities/ReusableContentTypes/{TypeName}/{TypeName}.generated.cs`:

```csharp
using System;
using System.Collections.Generic;
using CMS.ContentEngine;

namespace {CONTENT_TYPE_NAMESPACE}
{
    [RegisterContentTypeMapping(CONTENT_TYPE_NAME)]
    public partial class {TypeName} : IContentItemFieldsSource
    {
        public const string CONTENT_TYPE_NAME = "{CONTENT_TYPE_NAMESPACE}.{TypeName}";

        [SystemField]
        public ContentItemFields SystemFields { get; set; }

        // One property per user field
        // string for text/longtext, int for integer, bool for boolean,
        // DateTime for datetime, long for longinteger
    }
}
```

### Step 6: Build and Verify

```bash
dotnet build {ProjectName}.Entities/{ProjectName}.Entities.csproj --no-restore
```

If there's a name collision error, rename the type (go back to Step 3). Then build the full app:

```bash
dotnet build xbky/xbky.csproj
```

### Step 7: Create Content Hub Folder

Create the type folder under the workspace root from Step 1 (`@RootFolderID` / `@WorkspaceID`):

```sql
INSERT INTO CMS_ContentFolder (
    ContentFolderName, ContentFolderDisplayName, ContentFolderGUID,
    ContentFolderCreatedByUserID, ContentFolderCreatedWhen, ContentFolderModifiedWhen,
    ContentFolderTreePath, ContentFolderParentFolderID, ContentFolderWorkspaceID
) VALUES (
    N'{TypeName}', N'{Display Name}', NEWID(),
    @DefaultUserId, GETUTCDATE(), GETUTCDATE(),
    N'/{TypeName}', @RootFolderID, @WorkspaceID
);
```

### Step 8: Migrate Data

Write a SQL script to `scripts/migrate-{typename}-data.sql`. For each source row, insert into
four tables in this order:

1. **CMS_ContentItem** — use `ItemGUID` as `ContentItemGUID` for traceability

    | Column | Value |
    |--------|-------|
    | `ContentItemGUID` | Source `ItemGUID` |
    | `ContentItemName` | Slug derived from a display field |
    | `ContentItemContentTypeID` | `@ClassID` |
    | `ContentItemChannelID` | `NULL` |
    | `ContentItemIsSecured` | `0` |
    | `ContentItemIsReusable` | `1` |
    | `ContentItemContentFolderID` | `@FolderID` |
    | `ContentItemWorkspaceID` | `@WorkspaceID` (from Step 1 — never hardcode `1`) |

2. **CMS_ContentItemCommonData**

    | Column | Value |
    |--------|-------|
    | `ContentItemCommonDataGUID` | `NEWID()` |
    | `ContentItemCommonDataContentItemID` | `SCOPE_IDENTITY()` from step 1 |
    | `ContentItemCommonDataContentLanguageID` | `@DefaultLanguageId` |
    | `ContentItemCommonDataVersionStatus` | `2` (Published) |
    | `ContentItemCommonDataIsLatest` | `1` |
    | `ContentItemCommonDataFirstPublishedWhen` | `GETUTCDATE()` |
    | `ContentItemCommonDataLastPublishedWhen` | `GETUTCDATE()` |

3. **CMS_ContentItemLanguageMetadata**

    | Column | Value |
    |--------|-------|
    | `ContentItemLanguageMetadataContentItemID` | ContentItemID from step 1 |
    | `ContentItemLanguageMetadataDisplayName` | Human-readable name from source data |
    | `ContentItemLanguageMetadataLatestVersionStatus` | `2` |
    | `ContentItemLanguageMetadataGUID` | `NEWID()` |
    | `ContentItemLanguageMetadataCreatedWhen` | `GETUTCDATE()` |
    | `ContentItemLanguageMetadataCreatedByUserID` | `@DefaultUserId` |
    | `ContentItemLanguageMetadataModifiedWhen` | `GETUTCDATE()` |
    | `ContentItemLanguageMetadataModifiedByUserID` | `@DefaultUserId` |
    | `ContentItemLanguageMetadataHasImageAsset` | `0` |
    | `ContentItemLanguageMetadataContentLanguageID` | `@DefaultLanguageId` |

4. **{CONTENT_TYPE_NAMESPACE}_{TypeName}** — the type-specific data table

    | Column | Value |
    |--------|-------|
    | `ContentItemDataCommonDataID` | CommonDataID from step 2 |
    | `ContentItemDataGUID` | `NEWID()` |
    | All user columns | Source values |

Use a cursor to iterate source rows. Wrap in `BEGIN TRANSACTION / COMMIT`.
Skip already-migrated rows by checking `ContentItemGUID` existence.

### Step 9: Restart and Verify

1. Kill any running `dotnet run` process on port 20830
2. Start fresh: `dotnet run` from `xbky/`
3. Items should appear in the content hub folder and be editable

## Critical Gotchas

- **`ClassWebPageHasUrl`** must be `0` for reusable types — if NULL, items won't appear in admin
- **`ContentItemWorkspaceID`** must be the resolved `@WorkspaceID` on every `CMS_ContentItem` row — if NULL, items are invisible in the content hub. Do not hardcode `1` unless the user explicitly chose that workspace
- **New workspaces need a root folder**: inserting only into `CMS_Workspace` is not enough — create the root `CMS_ContentFolder` (`ParentFolderID IS NULL`, `TreePath = '/'`) for that workspace before adding type folders
- **`ContentItemCommonDataFirstPublishedWhen`** should be set to `GETUTCDATE()` for published items
- **Namespace must match page types**: `ClassName`, SQL table name, C# `namespace`, and `CONTENT_TYPE_NAME` all use `{CONTENT_TYPE_NAMESPACE}`, not `{ProjectName}`, unless the migration plan remapped everything to `{ProjectName}`
- **Name collisions**: The C# class lives in namespace `{CONTENT_TYPE_NAMESPACE}`. Any class name that matches an existing type in the Tessitura SDK or business layer will cause build errors. Always search first.
- **SQL reserved words**: Columns like `Key`, `Value`, `Order`, `Index` need `[brackets]` in SQL
- **`SET QUOTED_IDENTIFIER ON`**: Required at the top of every SQL script for xByK
- **Form field GUIDs**: Each `<field>` needs a unique GUID — generate deterministic ones or use `NEWID()` style
- **System fields have no `visible` attribute**: The three system fields (`ContentItemDataID`, `ContentItemDataCommonDataID`, `ContentItemDataGUID`) must NOT have `visible="true"` — this causes `ArgumentNullException` in the admin
- **App restart required**: Kentico caches content type definitions in memory. Always restart after creating a new type.
