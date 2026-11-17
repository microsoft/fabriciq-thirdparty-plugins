# Modify a model

Change what was asked, fit it into the model as if it had always been there, and prove that nothing else moved. Anything high-impact waits for the user's agreement.

**Uses:** [modeling-standards](./kb/modeling-standards.md), only the sections for the objects you touch; [dax](./kb/dax.md) for any DAX; [tmdl](./kb/tmdl.md) in file mode.

## Checklist

```
- [ ] 1. Locate the target
- [ ] 2. Learn the neighborhood
- [ ] 3. Check for existing work
- [ ] 4. Capture a baseline
- [ ] 5. Plan the change set
- [ ] 6. Apply
- [ ] 7. Test
```

## 1. Locate the target

Connect to the right model, or open the right PBIP folder. If several models or tables could match, confirm before writing. If the model is a PBIP folder, edit the folder even when Desktop has it open; Desktop only sees the changes after a reload (see [pbip](./kb/pbip.md#edit-a-project-open-in-desktop)).

## 2. Learn the neighborhood

Look at the objects you'll touch, what they reference and what references them, the storage mode and compatibility level, and the conventions around them: naming, measure placement, folders, format strings and description style.

## 3. Check for existing work

Look for a measure, column or field parameter that already does this. Reuse or extend it instead of adding a near-duplicate.

## 4. Capture a baseline

When the model is live, run the measures you'll change and their closest dependents by a relevant dimension, and keep the results (see [test-model](./test-model.md)). Without a live model, write the baseline queries so they can be run later.

## 5. Plan the change set

Order the changes by dependency: partitions and columns, relationships, measures, then metadata. Check the plan against the [Gotchas](#gotchas). Performance-motivated rewrites go through [optimize-performance](./optimize-performance.md) instead, so speed and equivalence are measured, not assumed.

**Stop for approval** on the changes listed under "Confirm irreversible or far-reaching changes" in SKILL.md, and on any business definition the model doesn't have yet.

## 6. Apply

Apply the change set in one batch where the tools allow it, sending complete definitions for operations that replace the whole object. After saving, check that no measure or calculated object is in an error state.

## 7. Test

Test the new behavior, then rerun the baseline to show that unrelated results didn't change. Rerun it even when you only touched a relationship: a different filter path can silently change totals. If anything differs that the request doesn't explain, find the cause, fix it and return to the start of this step.

## Gotchas

- **Renames and deletes.** Inside the model, find every dependent measure, calculated column, calculation item, relationship, sort-by column, hierarchy, role filter, field parameter and perspective, and update or report them. Outside the model you usually can't see what uses the name: say so, and in a PBIP project search the report folders for the old name.
- **Storage mode.** Check the source supports the target mode first (DirectQuery needs a foldable source: not files, web sources or non-foldable steps) and explain the impact. If it can't be done, leave the model unchanged and list the alternatives: scheduled or incremental refresh, moving the data to a DirectQuery-capable source, or Direct Lake on Fabric.
- **Data types.** Changing a type can change results (rounding, truncation) and break relationships. Check the source precision and every relationship and measure that uses the column.
- **Calculation groups.** A new item or a precedence change affects every measure in scope. Test a representative set of measures, including ones that shouldn't be transformed, and check format strings.
- **Field parameters.** An edit rewrites the whole field list, so build the complete new ordered set and keep the parameter's columns and metadata unchanged (see [modeling-standards](./kb/modeling-standards.md#reusable-logic)). A renamed parameter breaks report slicers bound to it.
- **Security roles.** Changing a filter changes who sees what. Test each affected role with impersonation, including a user who should see nothing. Assigning members is done in the Power BI service, not in the model.
