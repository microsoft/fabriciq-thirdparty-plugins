# Optimize a model for AI

Copilot, data agents and other natural-language consumers use model metadata (names, descriptions, measures and relationships) to pick the right fields for a question. This workflow makes the model unambiguous for them. AI instructions help, but can't rescue an ambiguous model.

**Uses:** [modeling-standards](./kb/modeling-standards.md), sections Columns and data types, Measures and Naming. Platform details change often: check [Optimize your semantic model for Copilot in Power BI](https://learn.microsoft.com/power-bi/create-reports/copilot-evaluate-data) before quoting limits.

## Checklist

```
- [ ] 1. Scope it
- [ ] 2. Assess, in order
- [ ] 3. Route the findings
- [ ] 4. Apply
- [ ] 5. Test
```

## 1. Scope it

- Ask how the model will be consumed: reports only, Copilot or agents, or both. Report-only models need the standard quality bar, not this whole workflow.
- Collect business context; don't invent it: key metrics and their competing definitions, the questions people ask, the vocabulary different teams use, the main date field to use, and the direction of good (is a lower value better?). Existing reports on the model show which questions matter.

## 2. Assess, in order

Each layer depends on the one before it. Don't polish descriptions on a model with a broken foundation.

1. **Foundation.** Star schema, correct data types, explicit measures for every metric people will ask about, no duplicate or overlapping measures, numeric attributes not summarized, a default label column on main dimensions, unused objects removed. See [modeling-standards](./kb/modeling-standards.md).
2. **Names.** Business terms in the language users ask in, unambiguous across tables (several `Name` columns across tables, or `Amount` and `Amt`, confuse the model).
3. **Descriptions.** On every visible table, column and measure. Only the start of a description is used as AI context (about 200 characters), so front-load the disambiguation: what it is, the unit, when to use it versus similar fields. Describe calculation groups on their column and list their items, because calculation items aren't exposed as metadata.
4. **Scope of what AI sees.** Keep out what AI shouldn't use (keys, helper columns, technical tables and measures) by hiding it in the model or leaving it out of the AI data schema. Both are guidance, not security (see Gotchas in SKILL.md).
5. **AI instructions.** Business rules, metric routing for ambiguous terms, date and fiscal defaults, polarity. They're guidance, not enforcement, and have a size limit.
6. **Verified answers.** A curated visual for the most common, high-stakes questions.

## 3. Route the findings

Tag each finding with who applies it:

| Item | Route |
| --- | --- |
| Names, descriptions, hidden flags, summarization, default label, data categories, measures, relationships | **Agent-editable** through [modify-model](./modify-model.md). |
| AI data schema, synonyms, AI instructions, verified answers | **User-configured** in the Power BI "Prep data for AI" experience ([Prepare your data for AI](https://learn.microsoft.com/power-bi/create-reports/copilot-prepare-data-ai)). The modeling tools can't edit this metadata. Draft recommendations for the user to apply themselves, and don't write the stored representation directly. Check the docs for the current options before drafting. |

**Browser editing (user-authorized only).** If you have browser tools, offer to apply the user-configured changes through the web modeling experience in the Power BI service; do it only if the user explicitly authorizes it. Confirm the exact changes with the user before saving, and follow the docs for the current UI steps. Saving has no undo, so first keep a copy of the current AI instructions and tell the user where it is, then add to them instead of replacing them.

**Stop for approval** of the agent-editable changes.

## 4. Apply

- Apply the approved agent-editable changes. Keep readiness changes separate from other fixes: new measures, format-string or visibility changes you noticed are proposed separately.
- Write descriptions from the business context you collected, not by restating the object name or guessing from the DAX. Show the drafts for review when they encode business meaning.

## 5. Test

Descriptions don't prove readiness; answers to real questions do. Give the user a set of representative questions, including ambiguous ones, with the expected answer and the field that should be used, so they can test in Copilot. If answers pick the wrong field, return to step 2 for that area.

## Gotchas

- **Existing Prep data for AI configuration.** Preserve it during any other change; if an operation could drop it, stop before writing.
- **Q&A.** Q&A is being retired in favor of Copilot, so don't invest in Q&A phrasings or linguistic tuning. Keep Q&A enabled while the AI data schema depends on it (for a PBIP project, check the Q&A setting in `definition.pbism`).
