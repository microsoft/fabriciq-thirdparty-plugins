# Optimize performance

Make it faster without changing what it returns. Speed claims need a measurement; equivalence claims need a comparison. Changes to what the user sees, or to the model and data, wait for approval.

**Uses:** [dax](./kb/dax.md#performance-patterns); [modeling-standards](./kb/modeling-standards.md), sections Storage mode, Columns and data types and Relationships, only for model changes.

## Checklist

```
- [ ] 1. Reproduce
- [ ] 2. Capture a baseline
- [ ] 3. Diagnose from the evidence
- [ ] 4. Change one thing at a time
```

## 1. Reproduce

- Get the exact slow query: from the report's Performance Analyzer, the user, or by writing the query a visual would generate (grouping columns, filters, measures).
- Resolve the full definitions involved: the measures, the measures they reference, calculation groups in scope, functions, relationships on the filter path, and the storage mode of the tables.

## 2. Capture a baseline

- Run the query with a server trace through the tools: total duration, formula engine versus storage engine time, number of storage engine queries, and their shape.
- Clear the cache for a cold run, then measure warm runs too. Use several runs; one timing is noise.
- Save the result set. Every candidate must reproduce it exactly: rows, subtotals, totals, blanks.

If you can't trace (no live model, no tools), still help, but say clearly that the gain isn't measured. Propose a rewrite only when it removes a cost you can name; restructuring an iterator without removing per-row work is not an improvement. Keep the original as a hidden `<name> (original)` measure, or leave it unchanged and add the candidate alongside, and give the user a DAX query that compares old and new by a relevant dimension and at the total. Don't launch Power BI Desktop or another engine to get a trace unless the user asked.

## 3. Diagnose from the evidence

Read the trace before you rewrite. Common signals:

| Signal | Usual cause |
| --- | --- |
| Callbacks from the storage engine, or high formula-engine time | Row-by-row work in iterators: context transition per row, `IF` or `DIVIDE` inside an iterator, iterating at a finer grain than needed. |
| Storage engine returns far more rows than the result | Filtering whole tables instead of columns, or materializing a large intermediate table. |
| Many similar storage engine queries | Sibling measures or branches that the engine can't combine into one scan. |
| Expensive distinct counts | Distinct count over a high-cardinality column. |
| Few, clean storage engine queries that are still slow | Data layout: cardinality, data types, relationship design, or for Direct Lake the Delta file layout. |

Treat any rewrite chosen without a trace as a hypothesis. [dax](./kb/dax.md#performance-patterns) lists the rewrites that usually match these signals.

## 4. Change one thing at a time

| Kind of change | Autonomy |
| --- | --- |
| Rewrite a measure or function with identical results | Apply, measure and compare. |
| Change the query shape (grain, filters, blank handling) | **Stop for approval.** It changes what the user sees. |
| Change the model or data (relationships, columns, types, aggregations, upstream layout) | **Stop for approval.** High impact: discuss the trade-offs and prefer testing on a copy. |

Keep the original definition so you can roll back. After each change, compare results with the baseline, then compare timings. A faster version with different numbers is a regression: revert it. If it's still too slow, return to step 3. Stop when the gain is negligible or the user's target is met.

When the model is a PBIP folder open in Desktop, trace and compare on the Desktop instance, but make each change in the folder and reload it before you measure (see [pbip](./kb/pbip.md#edit-a-project-open-in-desktop)). To try candidates without reloading, define them as query measures (`DEFINE MEASURE`) in the trace query.
