# Review a model

Give the owner a short, prioritized list of what's wrong and why it matters. A review is read-only until the user approves changes.

**Uses:** [modeling-standards](./kb/modeling-standards.md), all sections for a full review or the matching section for a focused one; [dax](./kb/dax.md) for measure logic.

## Checklist

```
- [ ] 1. Understand the context
- [ ] 2. Take an inventory
- [ ] 3. Evaluate
- [ ] 4. Report
```

## 1. Understand the context

Find out, or infer and state, who consumes the model, how widely it's shared, and whether it's in development or production. Severity depends on it: a model that Copilot answers from across the company is held to a higher bar than a personal prototype.

## 2. Take an inventory

Get the big picture before the details:

- Tables with storage mode, partition source, row counts and size when available; parameters and shared expressions.
- Relationships with cardinality, direction and active state.
- Measure count per table, calculation groups, field parameters, roles, perspectives, cultures.
- Compatibility level and model settings (auto date/time, discourage implicit measures).

For large models, sample the details by table instead of reading every expression. When metadata tools are not available, but you can execute DAX queries, use the DAX `INFO.VIEW` functions;

Use this query to enumerate all currently exposed `INFO.*` functions in the active engine/version.

    ```dax
    EVALUATE
    SELECTCOLUMNS(
        FILTER(
            INFO.FUNCTIONS(),
            LEFT([FUNCTION_NAME], 5) = "INFO."
        ),
        [FUNCTION_NAME], [DESCRIPTION]
    )
    ```

## 3. Evaluate

Check against [modeling-standards](./kb/modeling-standards.md), in this order. When the user asks about one area (naming, performance, security), evaluate only that area and say what you didn't review. Always complete the correctness and security checks. Go lighter on the lower categories when critical findings already decide the next step.

1. **Correctness.** Sums over mixed currencies or units, averaged ratios, non-additive balances summed over time, measures in an error state, broken sort-by or references, invalid format strings, ambiguous filter paths, duplicate keys on the "one" side, inactive relationships nothing uses.
2. **Security and exposure.** Sensitive or personal columns visible without need, missing or fail-open RLS, perspectives or hidden flags used as security.
3. **Design.** Star-schema violations, unmarked or gappy date table, auto date/time on, bidirectional or many-to-many without a reason, calculated columns that belong upstream.
4. **Performance and size.** High-cardinality columns that aren't needed, unsplit DateTime, unused columns and tables, storage mode that doesn't fit the source.
5. **Usability and AI.** Missing descriptions and format strings, visible keys, names that break the model's convention, duplicate or overlapping measures (for AI consumers, go deeper with [optimize-for-ai](./optimize-for-ai.md)).

Not every standard fits every model. Explain why a finding matters for this model, and don't report rule violations that have no consequence.

## 4. Report

List the findings in this table, sorted by severity (high, medium, low). Name specific objects: "some columns are missing descriptions" is not actionable, a count with examples is.

```markdown
| # | Severity | Object | Problem | Why it matters here | Proposed fix |
| --- | --- | --- | --- | --- | --- |
| 1 | High | [Margin (%)] | Averages a ratio across rows | Totals are wrong in every visual | Divide summed margin by summed revenue |
```

Then say what you'd fix first and what you didn't review. A review changes nothing, so the Definition of done report is only needed once fixes are applied.

**Stop for approval:** ask which fixes to apply. Make the approved ones through [modify-model](./modify-model.md), which has its own testing and report.

## Gotchas

- **Do not use `INFO` DAX functions to retrieve role memberships.** `INFO.ROLEMEMBERSHIPS()` returns empty or incomplete results. Role members are assigned at the service level (Entra ID) after deployment, not in the model definition — so DAX `INFO` functions cannot reliably surface them.