# Create a model

Design before you build. The most expensive mistakes in a model (wrong grain, wrong storage mode, wrong keys) are made before the first table exists. Nothing is built until the user agrees with the design.

**Uses:** [modeling-standards](./kb/modeling-standards.md), all sections; [dax](./kb/dax.md) for measures; [tmdl](./kb/tmdl.md) in file mode; [pbip](./kb/pbip.md) for a new project.

## Checklist

```
- [ ] 1. Frame the model
- [ ] 2. Discover the sources
- [ ] 3. Propose the design
- [ ] 4. Build in dependency order
- [ ] 5. Persist and load data
- [ ] 6. Test
```

## 1. Frame the model

Collect, reusing what the user already said:

- **Purpose and consumers.** Which decisions it supports; reports, self-service analysts, Copilot or agents (AI consumers raise the bar on names and descriptions, see [optimize-for-ai](./optimize-for-ai.md)).
- **Business process and grain.** What one fact row is (an order line, a daily balance, a ticket event).
- **Priority metrics.** Their definitions, and the dimensions people slice by.
- **Sources.** Their location, freshness and volume.
- **Security.** Who may see which rows or columns.
- **Target.** Desktop, Fabric workspace or PBIP folder, and the name.

Never fabricate a source or a business definition. When the user is still deciding what to build, do the read-only discovery in step 2 first, then ask one focused round of questions.

For a new subject area in an existing model, design only the new fact and its dimensions: reuse the existing dimensions and follow the model's conventions instead of the naming defaults.

## 2. Discover the sources

Reach the source with tools available that let you query the datasource schema and get necessary metadata for model development: Fabric MCP server, fabric-cli, Data Source MCP server or a command-line client. Read-only queries only. If nothing can reach the source, ask the user for the schema or a sample query result rather than guessing.

- Inspect schemas, data types with precision and scale, and keys through the source's metadata (SQL endpoint, catalog, file headers). Except for file sources (e.g. CSV), don't download data to learn its shape.
- Use aggregate queries to check key uniqueness, null keys, row counts, date ranges and orphan foreign keys. A sample can't prove a key is unique.
- Look for an existing date table, existing views that already model the business, and calculations already done upstream.

## 3. Propose the design

Write a short design brief:

- Tables with their role (fact or dimension), grain, source and storage mode.
- Relationships with keys and cardinality.
- Measures with their exact definitions.
- Security roles, if any.
- Open questions and the assumptions you'd make otherwise.

If you're blocked (no source, a missing business definition), still deliver the brief with options and a numbered list of exact questions. "Blocked" alone is not an answer.

**Stop for approval** before building, unless the request was already explicit and complete.

## 4. Build in dependency order

1. Source parameters or the Direct Lake shared expression.
2. Tables with partitions and columns (types, hidden flags, summarization, sort-by, data categories). Declare every column yourself, mapped to the query output or the lakehouse column (see Gotchas in SKILL.md).
3. Relationships.
4. Measures with format strings, descriptions and folders.
5. Security roles, hierarchies, calculation groups, functions.
6. Model-level settings: discourage implicit measures, turn off auto date/time.

Apply [modeling-standards](./kb/modeling-standards.md) throughout, including its naming defaults. Use a current compatibility level. Build the whole model before deploying; avoid deploy-fix-redeploy cycles.

## 5. Persist and load data

Save, deploy or export as the user asked (see [manage-model](./manage-model.md)). Refresh before testing data.

## 6. Test

Run the full [test-model](./test-model.md) pass. A new model passes only when every table has rows, every relationship has matching keys and every priority metric returns the expected value at the total and by at least one dimension. If a check fails, fix the cause and return to step 5.