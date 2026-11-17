# Test a model

A model is tested when its numbers have been checked against something independent. Saving, deploying and "the query ran" are not tests. Use this workflow on its own ("test this model", "check this measure") and as the last step of every change.

**Uses:** [dax](./kb/dax.md#dax-queries) for test queries.

## Checklist

```
- [ ] 1. Decide what you can test
- [ ] 2. Pick the test layers
- [ ] 3. Get a reference
- [ ] 4. Write and run the queries
- [ ] 5. Fix failures and rerun
```

## 1. Decide what you can test

| Situation | Possible tests |
| --- | --- |
| Tool mode on a live model (Desktop or Fabric) | Every layer. |
| Tool mode on an offline PBIP or TMDL folder | Structure only: the definition loads and serializes, references resolve. No data, so no DAX execution, refresh, traces or impersonation. Once the changes are reloaded in Desktop or deployed, connect to that live model to run the rest. |
| File mode | Parse the definition if any parser is available; otherwise review the diff. Write the DAX test queries and hand them to the user, or run them once the changes are live and tools can connect. |

## 2. Pick the test layers

Run the layers that apply to the change, cheapest first. Use aggregate queries; don't pull large row sets.

1. **Structure.** The model loads. References, relationships, sort-by columns and partition sources resolve. No measure, calculated column or calculation item is in an error state (check the object state or error message through the tools or INFO functions after saving).
2. **Data.** Every table has rows after refresh or framing. Relationship keys behave: the "one" side is unique and non-blank, and few or no fact rows fail to match a dimension (unmatched rows show up as a blank member in visuals). Date tables are contiguous and cover the facts.
3. **Results.** Query each new or changed measure:
   - by a relevant dimension and at the total, so non-additive logic and total behavior show up;
   - against an independent calculation: a simpler DAX expression, a source SQL query, or a figure the user trusts;
   - at edges: blanks, an empty selection, a single member, the first and last period for time intelligence.
4. **Regression.** Compare the baseline captured before the change with the same queries after it. Every difference must be explained by the request.
5. **Security.** For each affected role, run the same query with role or user impersonation and check the expected rows are visible, the rest are not, and an unmatched user sees nothing.
6. **Usability.** Visible measures have format strings and descriptions, keys and helper columns are hidden, sort orders work.

## 3. Get a reference

Results are only checked when they're compared with something: the baseline captured before a change, a source SQL query, a simpler DAX expression, or a figure the user trusts. When this workflow ends another one, reuse that workflow's baseline. A single number is weak evidence; a breakdown that adds up to an independently checked total is strong evidence.

## 4. Write and run the queries

Follow the query rules in [dax](./kb/dax.md#dax-queries).

- Use `EVALUATE` with `SUMMARIZECOLUMNS` for breakdowns and `ROW` for single values. Put the old and new versions of a measure side by side, plus a difference column, to make comparisons obvious.
- Define temporary measures in the query (`DEFINE MEASURE`) to try a definition before you write it into the model.
- Use dependency information (for example `INFO.CALCDEPENDENCY`) to find what a change can affect, and include those measures in the regression set.
- Keep queries small, named and reusable. If the project keeps saved DAX queries, offer to add the tests there.

## 5. Fix failures and rerun

Find the cause before fixing: a wrong definition, a wrong relationship or filter path, bad keys in the data, or a wrong expected value. Don't adjust a measure until it matches a number you haven't verified. Data problems (duplicate keys, orphan rows) are reported and fixed at the source, not masked in DAX. After a fix, return to step 4.
