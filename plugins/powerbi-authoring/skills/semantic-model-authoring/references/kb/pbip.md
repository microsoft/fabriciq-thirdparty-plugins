# PBIP

A Power BI project (PBIP) is a folder that Power BI Desktop saves instead of a `.pbix`: the semantic model as TMDL and the report as PBIR JSON. Use this to understand a project, to edit one that Desktop has open, or to make a TMDL model openable in Desktop and deployable. Report content (pages, visuals) is report work, not this skill.

## Contents

- [Structure](#structure)
- [Edit a project open in Desktop](#edit-a-project-open-in-desktop)
- [Empty report for a model](#empty-report-for-a-model)
- [Deploy](#deploy)

## Structure

```text
Sales/
├── Sales.pbip                      # Optional entry point; Desktop can also open definition.pbir
├── Sales.SemanticModel/
│   ├── definition.pbism            # Required
│   ├── definition/                 # The model in TMDL (see tmdl)
│   ├── .platform                   # Fabric item identity (Git integration)
│   └── .pbi/                       # Per-user Desktop state, including cache.abf with data
└── Sales.Report/
    ├── definition.pbir             # Required: which model the report uses
    ├── definition/                 # The report in PBIR
    │   ├── version.json
    │   ├── report.json
    │   └── pages/
    │       ├── pages.json
    │       └── <page>/page.json
    └── .platform
```

**What to know:**

- **Desktop opens reports, not models.** A model folder alone doesn't open; it needs a report folder whose `definition.pbir` points to it. A project can hold several reports and models.
- **The report-to-model link** is in `definition.pbir`:
  - `byPath`: a relative path to a model folder in the same project, with forward slashes. Desktop opens the model for editing.
  - `byConnection`: a model already in a workspace. Desktop opens the report connected live, and the model can't be edited from it.
- **`.pbi/` is Desktop's own state.** Don't create or edit it. `localSettings.json` and `cache.abf` stay out of source control. A project without `cache.abf` opens with no data until it's refreshed. A pending `unappliedChanges.json` overrides Power Query edits made outside Desktop.
- **`.platform` holds the Fabric identity** (`logicalId`). Desktop and Git integration create it; you don't need it to open a project. Never change `logicalId` on an existing item. When copying a project to deploy as new items, give each copy a new GUID.
- **Write files as UTF-8 without a byte order mark,** and keep paths short: Windows' 260-character path limit applies to the deepest file.
- **Page folders** use only letters, digits, `_` or `-`. Desktop silently skips a page whose folder breaks this, without an error.
- **Desktop doesn't watch the folder.** Changes made while the project is open need a reload (see [below](#edit-a-project-open-in-desktop)).

## Edit a project open in Desktop

When you edit a PBIP or TMDL folder (offline tool mode or file mode), the changes exist only on disk. Power BI Desktop doesn't watch the folder: it only sees edits after a reload, and its next save overwrites them. A model in Fabric only sees them after a deploy (see [manage-model](../manage-model.md#deploy-to-a-fabric-workspace)). Details: [Edit project files outside Power BI Desktop](https://learn.microsoft.com/power-bi/developer/projects/projects-external-editing).

- **Before you edit.** Have the user save in Desktop: a reload discards unsaved Desktop changes.
- **After you save, reload.** Either the user selects **Apply external changes** in the banner Desktop shows, or you reload through the [Power BI Desktop Bridge](https://learn.microsoft.com/power-bi/developer/agentic/power-bi-desktop-bridge-overview) if it's available (for example through the `powerbi-desktop` CLI). With the Bridge, read its manifest before calling methods, check the open file and unsaved-change state first, and don't reload over unsaved Desktop changes without the user's agreement.
- **Then verify.** Desktop reloads the whole semantic model, not single files. Connect to the reloaded instance and check that your change is there before testing; use that connection read-only and keep editing the folder.

## Empty report for a model

The minimum to open a TMDL model in Power BI Desktop. It creates one blank page; add content with the report tools. Use the model folder's name in the report folder, the `.pbip` and the `byPath` path. Take newer schema versions from an existing project or the [json-schemas repository](https://github.com/microsoft/json-schemas/tree/main/fabric) when they matter; these are known to open.

`Sales.pbip`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/pbip/pbipProperties/1.0.0/schema.json",
  "version": "1.0",
  "artifacts": [
    {
      "report": {
        "path": "Sales.Report"
      }
    }
  ],
  "settings": {
    "enableAutoRecovery": true
  }
}
```

`Sales.SemanticModel/definition.pbism`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/definitionProperties/1.0.0/schema.json",
  "version": "4.2",
  "settings": {
    "qnaEnabled": true
  }
}
```

`Sales.Report/definition.pbir`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byPath": {
      "path": "../Sales.SemanticModel"
    }
  }
}
```

`Sales.Report/definition/version.json`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/versionMetadata/1.0.0/schema.json",
  "version": "2.0.0"
}
```

`Sales.Report/definition/report.json`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/report/3.0.0/schema.json",
  "themeCollection": {}
}
```

`Sales.Report/definition/pages/pages.json`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/pagesMetadata/1.0.0/schema.json",
  "pageOrder": ["page1"],
  "activePageName": "page1"
}
```

`Sales.Report/definition/pages/page1/page.json`

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/page/2.0.0/schema.json",
  "name": "page1",
  "displayName": "Page 1",
  "displayOption": "FitToPage",
  "height": 720,
  "width": 1280
}
```

## Deploy

Deploy the model first, then the report. A report deployed on its own needs its model in the workspace: if the deploy tool doesn't resolve `byPath`, set `byConnection` to the deployed model (`"connectionString": "semanticmodelid=<id>"`) in the copy you deploy, and keep `byPath` in the project. See [manage-model](../manage-model.md#deploy-to-a-fabric-workspace) for the tools.