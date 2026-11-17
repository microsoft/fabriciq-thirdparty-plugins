# Manage a model

These operations act on shared resources and might not be possible to undo. Each operation below follows the same pattern: check first, do it once, verify the result.

**Uses:** [pbip](./kb/pbip.md) for project files.

## Checklist

```
- [ ] 1. Check first
- [ ] 2. Do
- [ ] 3. Verify
- [ ] 4. Report
```

## Deploy to a Fabric workspace

**Check first.** If a model with that name already exists and the user asked for a new one, stop and ask whether to update it or use another name. Never delete a model to make room. If nothing can deploy, stop and tell the user (for example: publish from Power BI Desktop).

**Do.** Deploy once. On a timeout or an unclear result, check the workspace and the operation status before you retry.

- **Tool mode:** only when the connected tools explicitly support deployment from the chosen source. Hosted Authoring does not expose the local server's folder/deploy operations; don't invent one or replace a local project with live edits.
- **File mode,** following each tool's current documentation:
  - **[Fabric CLI](https://microsoft.github.io/fabric-cli/examples/item_examples/#import-item)** (`fab`), preferred when it's installed: import the `<Name>.SemanticModel` folder as an item, for example `fab import "Sales WS.Workspace/Sales.SemanticModel" -i ./Sales.SemanticModel`. Check `fab auth status` first and run `fab import --help` rather than guessing options.
  - Otherwise, the [Fabric REST APIs](https://learn.microsoft.com/rest/api/fabric/semanticmodel/items) or [fabric-cicd](https://microsoft.github.io/fabric-cicd/).

**Verify.** Connect to the deployed model and check its tables and measures, then refresh or frame it and run a query. A model that appears in the workspace list but has no data is not deployed successfully.

## Refresh

**Check first.** Refresh needs a live model in Desktop or Fabric; deploy local files first. Metadata-only changes (measures, descriptions, formats) don't need a data refresh; a new or changed column, table or partition does. Respect existing incremental refresh policies and schedules.

**Do.** Import refresh reloads data from the source. Direct Lake refresh is framing: it points to the latest Delta versions, and a new Direct Lake model needs it before it returns data.

**Verify.** Wait for the refresh to finish and check its status. On credential or gateway errors, stop: the user configures them in the Power BI service. Don't retry or work around them.

## Export to a PBIP project

**Check first.** Use the folder and name the user gave. Otherwise pick a new folder, say which, and never overwrite an existing project.

**Do.**

Check the actual export/save operations first. Hosted Authoring does not expose
the local server's TMDL-folder export. If it is unavailable, explain the limit
and use a supported local tool or user-driven Desktop export; do not claim a
complete project was exported from partial metadata.

1. Save the model's TMDL to `<Name>.SemanticModel/definition/` with the tools' export or save operation.
2. Create the missing project files. A TMDL folder alone doesn't open in Desktop: it needs a report that points to it. Copy the files from an existing project when there is one; otherwise use the empty report in [pbip](./kb/pbip.md#empty-report-for-a-model). If the report tools or report skills are available, they can create the report instead.

**Verify.** Run the checks listed in [pbip](./kb/pbip.md#empty-report-for-a-model).

## Change data sources or access

**Check first.** Read the current state before changing it. Match connections on type, server and database, not a partial name. Explain access implications before changing permissions or switching between single sign-on and fixed identities: access to the model doesn't grant access to its sources, and vice versa.

**Do.**

- **In the model** (through [modify-model](./modify-model.md)): source parameters, shared expressions, partition queries.
- **In the Power BI service** (only if a tool supports it, otherwise tell the user where): credentials, gateway and connection binding, workspace and model permissions, role membership, refresh schedules, endorsement and sensitivity labels.

**Verify.** After a source change, refresh and compare row counts with the previous refresh. After an access change, confirm with the user that the intended people can, and others can't, reach the model.

## Report

Use the report template in the Definition of done in SKILL.md. Its Persisted check and "Where it lives" line matter most here; report only the checks you actually ran.
