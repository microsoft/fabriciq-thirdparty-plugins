# DAX

DAX that runs is not the same as DAX that is right, or fast on a large model. Load this before you write or review any DAX: measures, calculated columns, calculation items, RLS filters, functions and queries. On an existing model, its DAX style wins (see the consistency principle in the skill); these are the defaults and the traps.

## Contents

- [Standards](#standards)
- [Context: where wrong numbers come from](#context-where-wrong-numbers-come-from)
- [Performance patterns](#performance-patterns)
- [DAX Queries](#dax-queries)
- [DAX Functions](#dax-functions)

## Standards

- **References:** columns always as `'Table'[Column]`, with the table name in single quotes even when it doesn't need them; measures always as `[Measure]` without a table. Moving a measure then never breaks an expression, and a reader can tell measures from columns.
- **Variables split the logic.** Use clearly named variables for meaningful intermediate results or repeated expressions. A variable captures its value in the context where it's defined; it is not a reusable expression. Prefix variables with `_` unless the model uses another style.
- **Build on base measures.** Reuse `[Revenue]` rather than repeating `SUM ( 'Sales'[Net Amount] )` with a filter. One definition per business rule.
- **Ratios with `DIVIDE`.** For a complement that should stay blank when the denominator is zero or blank, use `DIVIDE ( y - x, y )`, not `1 - DIVIDE ( x, y )`: the latter returns 1 when the division returns BLANK. Use `/` when the denominator is known to be nonzero and nonblank; benchmark performance-sensitive rewrites.
- **BLANK is a feature.** Don't add `+ 0`, `COALESCE ( ..., 0 )` or `IF ( ISBLANK ( ... ), 0 )` unless the business definition requires zero: they can make visuals show members with no data. Defaults must express a business rule.
- **Filter columns, not tables.** In `CALCULATE`, prefer column predicates and separate arguments for independent conditions. Compound predicates with `&&` or `||` are valid when needed. Use `KEEPFILTERS` to intersect with an existing filter rather than replace it. Use `FILTER` over the required entity grain when the condition needs a measure evaluated per entity.
- **Prefer functions that state the intent.** Use `REMOVEFILTERS` to clear filters and keep `ALL` for when a table is needed, so a reader knows which of the two `ALL` would mean. Use `TREATAS` to apply a filter across tables without a relationship, rather than `INTERSECT` or `FILTER` with `CONTAINS`; it is clearer and usually faster. Use `SELECTEDVALUE` rather than `IF ( HASONEVALUE ( ... ), VALUES ( ... ) )`.
- **No errors as control flow.** Avoid `IFERROR` and `ISERROR`; guard expected invalid inputs with `DIVIDE` or explicit checks using `IF`. `COALESCE` handles BLANK, not errors. Never leave `EVALUATEANDLOG` in a model.
- **Keep numeric measures numeric.** Use format strings or dynamic format strings instead of `FORMAT` unless the requested output is text.
- **Format for reading.** One function argument per line when an expression doesn't fit on one line, and consistent indentation. Comments only for business rules the code can't show.

The standards together, in one measure: meaningful variables, base measures reused, a column filter that respects the user's selection, `DIVIDE` keeping BLANK when there is no revenue, and a comment only for the business rule.

```dax
High Discount Revenue (%) =
VAR _revenue = [Revenue]
VAR _highDiscountRevenue =
    CALCULATE (
        [Revenue],
        KEEPFILTERS ( 'Sales'[Discount Rate] >= 0.2 ) // 20% or more needs manager approval
    )
RETURN
    DIVIDE ( _highDiscountRevenue, _revenue )
```

## Context: where wrong numbers come from

- **Context transition.** A measure reference, or `CALCULATE`, inside an iterator or a calculated column turns row context into filter context. It filters by column values, not row identity, so duplicate rows can cause overcounting. Check the iteration grain and the cost of repeated transitions.
- **Variables do not reevaluate under new filters.** `VAR _sales = [Sales] RETURN CALCULATE ( _sales, ... )` returns the captured value, not `[Sales]` under the new filters. Put the measure expression inside `CALCULATE` when it must reevaluate.
- **Calculated columns** start with row context, not a filter on the current row. Without context transition, `SUM ( 'Sales'[Amount] )` returns the grand total on every row.
- **Totals are evaluated, not summed.** A total evaluates the measure in its own filter context. Ratios, distinct counts and conditional measures need not add up from the displayed rows. Only when the business definition requires a sum of contributions, iterate that grain explicitly: `SUMX ( VALUES ( 'Product'[Category] ), [Measure] )`.
- **Grouping is not selection.** Use `ISINSCOPE` to detect the grouping level in a visual. `HASONEVALUE` tests cardinality and can be true at a total when a slicer selects one member.
- **BLANK is not zero.** `=` can treat BLANK as zero or an empty string. Use `ISBLANK` or strict equality (`==`) when those states have different meanings.
- **Filter the dimension, not the fact.** For shared attributes, filter `'Date'[Date]` or `'Product'[Category]`, not a copy on the fact or its foreign key. In a typical single-direction star schema, a fact filter does not reach other facts. Attributes that exist only on the fact are an exception.
- **Whole-table filters have wider effects.** Clearing or applying filters to a fact table in `CALCULATE` can affect related dimensions through expanded-table semantics. Target the intended columns or dimensions unless that broader effect is deliberate.
- **Sort-by columns can retain filters.** Visuals may group by both a displayed column and its sort-by column. When clearing the displayed grouping, check whether its sort-by filter must also be removed.
- **Time intelligence** must match the model's classic or calendar-based approach and its date-table requirements. Verify the anchor date, window boundaries and partial-period treatment; the correct offset depends on the function and the intended endpoints. In a query without date grouping, provide the intended date context explicitly, not an assumed `TODAY()`.
- **`ALLSELECTED`** depends on the visual's shape and on outer iterators. Use it only for "share of what's visible" and test it in the target visual.

## Performance patterns

Rewrites are hypotheses. Apply one, then measure and compare results per [optimize-performance](../optimize-performance.md). A pattern can help, do nothing or hurt, depending on the data and the model.

**Column predicates instead of table filters.** A whole-table filter has a wider scope than a column predicate. Preserve intersection with `KEEPFILTERS`, and verify equivalence if the original filter also affected related tables.

```dax
// Avoid
CALCULATE ( [Orders], FILTER ( 'Date', 'Date'[Year] = 2025 ) )
// Prefer
CALCULATE ( [Orders], KEEPFILTERS ( 'Date'[Year] = 2025 ) )
```

**Variables for repeated or row-independent work.** Hoist a measure only when its value in the outer context is the same as for every iterated row. Here, `[Target Markup]` must not vary by product, customer or date within that context; otherwise iterate the grain where it varies.

```dax
// Avoid: [Target Markup] is evaluated for every sales row
SUMX ( 'Sales', 'Sales'[Cost Amount] * ( 1 + [Target Markup] ) )
// Prefer
VAR _markupFactor = 1 + [Target Markup]
RETURN SUMX ( 'Sales', 'Sales'[Cost Amount] * _markupFactor )
```

**Iterate the smallest grain that gives the right answer.** Here, `[Revenue]` must be additive across products so products with the same commission rate can be combined. This is not a general rewrite for distinct counts, ratios or conditional measures. Preserve unknown-member behavior too: `VALUES` can include an extra blank member for unmatched fact keys.

```dax
// Avoid: one context transition per product
SUMX ( 'Product', [Revenue] * 'Product'[Commission Rate] )
// Prefer: one per distinct commission rate, usually a handful
SUMX ( VALUES ( 'Product'[Commission Rate] ), [Revenue] * 'Product'[Commission Rate] )
```

**Build a reusable grouped result.** `SUMMARIZECOLUMNS` is an option, not an automatic upgrade from `ADDCOLUMNS`. Check target-engine support inside measures, grouping and filter semantics, blank-row elimination, and measured performance. A table variable does not guarantee a particular physical materialization or number of scans.

```dax
// Peak month versus the average of months with nonblank orders
VAR _monthly =
    SUMMARIZECOLUMNS ( 'Date'[Year], 'Date'[Month], "@Orders", [Orders] )
RETURN
    DIVIDE ( MAXX ( _monthly, [@Orders] ), AVERAGEX ( _monthly, [@Orders] ) )
```

**Keep row-level logic simple inside iterators.** Per-row conditions and measure evaluations can be expensive; use traces to establish the cost. A column condition may move to a filter when the base aggregation is equivalent. Here, `[Revenue]` must be `SUM ( 'Sales'[Net Amount] )`, and `KEEPFILTERS` preserves any existing discount selection.

```dax
// Avoid: IF evaluated on every sales row
SUMX ( 'Sales', IF ( 'Sales'[Discount Rate] >= 0.2, 'Sales'[Net Amount] ) )
// Prefer: the condition becomes a filter on the base measure
CALCULATE ( [Revenue], KEEPFILTERS ( 'Sales'[Discount Rate] >= 0.2 ) )
```

**Apply shared filters once when equivalent.** Here, each LY measure must be exactly its base measure under the same `SAMEPERIODLASTYEAR` filter, and `[Revenue per Order]` must be `DIVIDE ( [Revenue], [Orders] )`. Applying the filter to the combination may help, but does not guarantee a single scan.

```dax
// Avoid: each with its own SAMEPERIODLASTYEAR
DIVIDE ( [Revenue (LY)], [Orders (LY)] )
// Prefer
CALCULATE ( [Revenue per Order], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
```

**Other signals worth testing:**

- `DISTINCTCOUNT` on a high-cardinality column: test `COUNTROWS ( VALUES ( ... ) )` or `SUMX ( DISTINCT ( ... ), 1 )`, or `COUNTROWS` on the table when the column is its unique key. Preserve blank-member and empty-result behavior; these expressions are not unconditional substitutes.
- `SWITCH` over measure branches (for example a measure selector): keep each branch a simple aggregation and read the column the slicer filters directly.
- Bidirectional or many-to-many paths: test `TREATAS` or `CROSSFILTER` inside the measure before changing the model.
- Text searches (`SEARCH`, `CONTAINSSTRING`) and complex logic in RLS filters run for every query: precompute a column upstream.

## DAX Queries

```dax
// Compare year-month results and the grand total over two years
DEFINE
    MEASURE 'Sales'[Margin Candidate] = [Revenue] - SUM ( 'Sales'[Cost Amount] )
    VAR _years = TREATAS ( { 2024, 2025 }, 'Date'[Year] )
    VAR _comparison =
        SUMMARIZECOLUMNS (
            ROLLUPADDISSUBTOTAL (
                ROLLUPGROUP ( 'Date'[Year], 'Date'[Month Number], 'Date'[Month] ),
                "IsTotal"
            ),
            _years,
            "Current", [Margin],
            "Candidate", [Margin Candidate]
        )
EVALUATE
ADDCOLUMNS (
    _comparison,
    "Difference", [Candidate] - [Current],
    "SameBlankStatus", ISBLANK ( [Candidate] ) == ISBLANK ( [Current] ),
    "ExactMatch", [Candidate] == [Current]
)
ORDER BY [IsTotal] DESC, 'Date'[Year], 'Date'[Month Number]
```

The inner query suppresses rows where both results are BLANK before comparison flags are added. Strict equality distinguishes BLANK from zero. For floating-point results, use a justified absolute or relative tolerance for the numeric comparison, while still requiring the same BLANK status. If expected rows are part of the requirement, check their presence separately.

- One `DEFINE` block at the top. Define a query measure on its host table (`MEASURE 'Sales'[Name] = ...`) and refer to it as `[Name]`.
- `SUMMARIZECOLUMNS` is the default for grouped results with measures: group-by columns, then filters, then name and expression pairs. It drops rows where every measure is blank, which is usually what a visual does too.
- Filter with `TREATAS`, a table variable or an outer `CALCULATETABLE`, not with a boolean expression inside `SUMMARIZECOLUMNS`.
- `SUMMARIZE` only for distinct combinations of columns, never to add calculated columns. `SELECTCOLUMNS` to rename or keep duplicate rows; later expressions use the new names.
- Boolean filters in `CALCULATE` can't reference a measure directly. A variable can supply a fixed scalar threshold; for a condition reevaluated per entity, use `FILTER ( VALUES ( 'Product'[Product Key] ), [Revenue] > 1000 )` or the required entity grain instead.
- Add `ORDER BY` when a query returns several rows. Use `TOPN` or a filter to keep results small; don't pull fact tables into the conversation.
- `ROW` for a single row of values, wrapped in `CALCULATETABLE` when it needs filters.

## DAX Functions

DAX user-defined functions remove repeated logic. Read the model's level first and see [modeling-standards](./modeling-standards.md#reusable-logic) before using them.
