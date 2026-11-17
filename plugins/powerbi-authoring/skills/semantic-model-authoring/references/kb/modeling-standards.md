# Modeling standards

The standards new and changed objects should meet, and the checklist a review compares a model against. These are defaults, not laws: on an existing model, the consistency principle in SKILL.md applies. Explain why a standard applies before recommending a change. For a focused edit, read only the sections for the objects you're touching.

## Contents

- [Shape](#shape)
- [Storage mode](#storage-mode)
- [Columns and data types](#columns-and-data-types)
- [Relationships](#relationships)
- [Date table](#date-table)
- [Measures](#measures)
- [Reusable logic](#reusable-logic)
- [Security](#security)
- [Naming](#naming)
- [Data source parameters](#data-source-parameters)

## Shape

- **Star schema by default.** Avoid snowflakes, fact-to-fact relationships and wide flat tables unless a concrete requirement needs them.
- **Declare the grain.** Every fact's description states one row = what. Measures and relationships follow from it.
- **Conform dimensions.** Facts that should slice together share dimensions with compatible keys and grain. Use bridges for genuine many-to-many associations. For higher-grain facts, design relationships and measures around that grain; don't imply detail-level attribution or allocation the data doesn't contain.
- **Lean.** Only the tables and columns the scenario needs. Unnecessary columns add storage, refresh or query costs depending on the mode, and every visible field is one more choice for users and AI.

## Storage mode

Decide before building; it's hard to change later.

| Mode | Default when |
| --- | --- |
| Direct Lake | The data is Delta tables in Fabric OneLake, suitable Fabric capacity is available, and the variant supports the required features and security model. |
| Import | Anything else. |
| DirectQuery | Only with a stated reason (freshness, volume or source-side security), a supported source and foldable transformations. Explain the cost: potentially slower visuals, DAX and Power Query limits, and limited relationships across source groups. Per-user source security requires an appropriate identity configuration, such as SSO; DirectQuery alone doesn't provide it. |

**Import refresh design.** Choose refresh frequency and history retention from freshness and volume requirements. Consider incremental refresh for large facts and verify that partition filters are efficiently applied at the source. It isn't automatic.

**Direct Lake behaves differently from Import:**

- **No Power Query for Direct Lake tables.** They use entity partitions pointing to OneLake through shared expressions, not transformation queries. Prepare their data upstream. A mixed model can also contain Import tables with Power Query where supported.
- **Two variants.** One reads OneLake directly (an `AzureStorage.DataLake` expression); the other goes through the SQL analytics endpoint (`Sql.Database`) and can fall back to DirectQuery for views or SQL security. Keep the variant an existing model uses.
- **Refresh is framing.** It updates metadata references to Delta versions rather than performing a full data import; columns still load into memory for queries. Frame new or changed tables before testing: unframed tables error or, on the SQL endpoint variant, can fall back to DirectQuery. A query that returns results doesn't prove it ran in Direct Lake mode.
- **Schema from current metadata.** Take names, types and schemas from the current lakehouse metadata (SQL endpoint or catalog), not old Delta logs or one Parquet file. Leave out unsupported types such as binary and say why.
- **Keys fail late.** A non-unique "one" side key fails at query time, not at refresh. Check uniqueness before you relate.
- **Limits change.** Check current limits on learn.microsoft.com (calculated columns, guardrails, fallback) before promising a feature.

## Columns and data types

- **Keep the source meaning.** Preserve key identity, range and required precision. Use `Int64` for integer surrogate keys and whole numbers; keep textual identifiers, including leading zeros, as text. Use fixed `Decimal` for money when its four-decimal scale and range fit; `Double` is floating point, not an exact substitute for higher-precision decimals. If the source needs greater exact precision, agree on the representation rather than silently rounding or converting. Use `DateTime` for dates.
- **Hide what users shouldn't pick.** Keys, sort helpers and technical columns. Hide raw numeric columns intended only for aggregation through measures, but retain columns needed for legitimate detail analysis, grouping or filtering.
- **Numeric attributes don't sum.** Set summarization to none on IDs, years, month numbers and codes.
- **Sort and categorize.** Month and day names sort by a number column. Each displayed value must map to exactly one sort value; sort year-month labels by a chronological year-month key, not month number alone. Set the data category on geography, URL and image columns.
- **Default label.** On main dimensions, set the column that identifies a row to users (for example the product name) as the default label where consuming clients support it. Don't assume every visual or AI client uses this metadata.
- **Describe the meaning.** Describe table purpose and grain, and clarify visible columns whose meaning, units or codes aren't obvious from their names.
- **Watch cardinality.** Remove time or precision that analysis doesn't need; split date and time only when both are needed separately. Preserve the required business meaning. Question GUIDs, free text and transaction IDs that aren't needed for analysis.
- **Calculated columns are a last resort.** Prefer the source or Power Query. Never use a calculated column for logic that depends on the filter context; that's a measure.

## Relationships

- **One-to-many, single direction by default.** The "one" side key is unique and non-blank (prove it with a query, not a sample); both keys have the same data type.
- **No workarounds.** Don't add bidirectional filtering or many-to-many just to make a visual show rows or to hide duplicate keys. A deliberate bidirectional path, such as for a bridge, needs a stated purpose and checks for ambiguity, totals and security propagation. Query filter direction and bidirectional security propagation are separate settings.
- **Design for each role.** Prefer separate, actively related role-playing dimensions when users must filter roles independently or simultaneously, such as Order Date and Ship Date. Use inactive relationships for intentional measure-specific alternatives with `USERELATIONSHIP`; this doesn't change the stored relationship. RLS never propagates through an inactive relationship, even when a measure uses `USERELATIONSHIP`.
- **Fix keys at the source.** Report missing or duplicate keys (referential integrity) and fix them at the source; don't mask them.

## Date table

- **Source first.** Prefer a shared date table from the source, then Power Query. A local DAX date table is acceptable when the target model supports it and no suitable shared table is available. Cover the relevant fact dates and required reporting periods; for classic time intelligence, use unique, nonblank, contiguous dates spanning full calendar or fiscal years.
- **Match the time-intelligence approach.** For classic time intelligence, mark the date table and check the saved definition: the table has the `Time` data category and the date column is marked as the key. Tools don't always set both. For calendar-based time intelligence, validate the calendar definitions and required category assignments; marking alone isn't a substitute. See [date table guidance](https://learn.microsoft.com/power-bi/transform-model/desktop-date-tables).
- **Join at date grain.** Use a date-only value or matching date key, not a timestamp containing a time component. When deriving dates from timestamps, establish the business timezone before taking the date.
- **No auto date/time.** Turn it off when the model has a proper date table.
- **Business calendar.** Respect the fiscal year start and week definitions. They're business definitions: ask, don't assume.

## Measures

- **Explicit measures.** Every metric that users or AI should use has its own measure. Hide aggregation-only source columns, not columns still needed for detail analysis or filtering. Discourage implicit measures in new models; assess report-authoring impact and obtain agreement before changing that setting on an existing model.
- **Aggregation fits the grain.** See the business meaning principle in the skill. Use `DIVIDE` for ratios and keep BLANK behavior intentional: don't turn blanks into zeros unless the business definition requires it, because it can fill visuals with members that have no data.
- **Reuse before you create.** Search for a measure that already computes the same thing; build new ones on existing base measures.
- **Metadata in the same pass.** Format string, description (business rule, unit, when to use it versus similar measures), display folder if the model uses them.
- **Placement follows the model.** Put new measures where the model puts similar ones (home fact table or a measure table). Don't introduce a new scheme.
- **DAX follows the standards.** See [dax](./dax.md), starting with references: columns always as `'Table'[Column]`, measures always as `[Measure]` without a table, so moving a measure never breaks an expression.

## Reusable logic

- **Calculation groups** fit when the same transformation (time intelligence, currency, scenario) applies to many measures. They change every measure in their scope: check precedence with other groups, dynamic format strings, and the effect on measures that shouldn't be transformed. For a handful of measures, plain measures are simpler.
- **DAX user-defined functions** need compatibility level 1702 or higher and target-engine support. Check both before using them; a compatibility upgrade is irreversible and needs agreement. Read [DAX user-defined functions](https://learn.microsoft.com/dax/best-practices/dax-user-defined-functions) for syntax and guidance.
- **Field parameters** are calculated tables. A table is a field parameter when one of its columns carries `ParameterMetadata` with `"kind": 2`. Check for an existing one before creating another.
  - Create them with the tools dedicated field-parameter operation when there is one; it builds the metadata correctly.
  - Editing the field list means rewriting the whole calculated partition expression with the complete new ordered set; there's no merge.
  - After any change, check that the label, fields and order columns, their sort-by and group-by settings and the `ParameterMetadata` are intact. In file mode, copy a working field parameter or follow the current documentation; don't write one from memory.
  - The report slicer and visual are report work.

## Security

- **Simple RLS with explicit paths.** Prefer dimension-based RLS with tested propagation to the facts. Use security mapping tables when needed, keep filters simple and push complex string logic upstream.
- **Fail closed.** An unmatched user sees nothing, not everything.
- **Roles add access.** Multiple RLS roles grant the union of their effective allowed rows, not their intersection. A restrictive role doesn't override a permissive one.
- **Consumer permissions matter.** RLS doesn't constrain users with model write permissions, including workspace Admins, Members and Contributors. Validate actual consumer permissions, not just role expressions.
- **Test every role and relevant combinations.** Use impersonation (see [test-model](../test-model.md)), including an unmapped user. Role membership is assigned in the Power BI service, not in the model.
- **Hiding isn't security.** Object-level security hides tables or columns from a role. Perspectives and hidden flags don't: anyone with model access can still query those objects.

## Naming

Consumers, users and AI, rely on names being predictable. Use these defaults for new models and new subject areas unless the user gives their own convention. On an existing model, its convention wins (see the consistency principle in SKILL.md).

**Scope:** these conventions cover what users see when they explore the model: tables, columns, measures, hierarchies and levels, display folders, calculation groups and items, and field parameters. Applying them to an existing model doesn't extend to partitions, shared expressions and Power Query queries, parameters, DAX functions, roles or relationships. Leave those names as they are unless the user names them explicitly; scripts, refresh and deployment tools can depend on them.

- **Business language.** Use the terms consumers ask questions in, in Title Case with spaces: `Order Date`, not `OrderDate`, `order_date` or `ORDER_DATE`. No technical prefixes on visible objects (`Dim`, `Fact`, `DIM_`, `vw_`). Spell words out; keep only abbreviations the business uses (YTD, KPI) and define them in the description.
- **Tables.** Facts are plural or named after the process (`Sales`, `Orders`, `Invoices`); dimensions are singular (`Product`, `Customer`, `Date`).
- **Columns.** A dimension's main label column takes the table's name (`'Product'[Product]`, not `Product Name`). Keys end in `Key` (`Product Key`) and are hidden. Don't otherwise repeat the table name (`'Customer'[City]`, not `Customer City`).
- **Measures.** Name the metric, not the calculation: `Revenue`, `Margin`, `Orders`, not `Sum of Amount` or `Revenue Measure`. Counts are plain nouns (`Orders`, `Customers`).
- **Measure variants.** The base name, then the variant in parentheses with uppercase abbreviations: `Revenue (YTD)`, `Revenue (LY)`, `Margin (%)`. Put the period before the unit when combining them: `Margin (LY) (%)`. Use the same abbreviations everywhere in the model.
- **Display folders.** One folder per business subject (`Revenue`, `Margin`, `Orders`), with variants next to their base measure. Add subfolders only when a subject has many measures (`Revenue\Time Intelligence`).

## Data source parameters

Centralize each source location (server, database, workspace, lakehouse), so changing environments doesn't require editing every table's query. Different sources can need different parameters or shared expressions.

- **Parameters for Import and DirectQuery.** Define each source's location as Power Query parameters (named expressions marked as parameters) and reuse them in its partition queries. Name them after what they hold (`Server`, `Database`), distinguishing sources when needed.
- **Shared expressions for Direct Lake.** Reuse the shared expression for each source rather than repeating its location across tables. Don't assume the whole model has only one source.
- **Locations, not credentials.** Never put passwords, tokens or other secrets in parameters or model text. Configure credentials through the supported connection or credential management mechanism.
- **Deployment sets the values.** Deployment pipelines, `fabric-cicd` or the model's settings in the service can change parameter values per environment without editing the model.
- **Don't refactor existing sources unasked.** Use parameters or shared expressions for new models and new tables; reuse existing ones for the same source. Don't convert unrelated existing partition queries as part of a focused change.

See the example of parameter use

```tmdl
expression Server = "sql-prod.contoso.com" meta [IsParameterQuery = true, Type = "Text", IsParameterQueryRequired = true]

expression Database = "SalesDW" meta [IsParameterQuery = true, Type = "Text", IsParameterQueryRequired = true]
```

```tmdl
partition Sales = m
  mode: import
  source =
      let
          Source = Sql.Database(Server, Database),
          Sales = Source{[Schema = "dbo", Item = "fact_sales"]}[Data]
      in
          Sales
```