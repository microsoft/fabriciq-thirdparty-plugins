# TMDL

Load this before writing or reviewing TMDL: in file mode, and when reviewing a PBIP diff. In file mode, the files are the model: every character you write is what Power BI loads, and nothing validates it for you until someone opens it.

## Contents

- [Before you edit](#before-you-edit)
- [Rules that break loading](#rules-that-break-loading)
- [Edit safely](#edit-safely)
- [Validate](#validate)
- [Complete model example](#complete-model-example)

## Before you edit

- **Check whether Power BI Desktop has the project open.** If it does, follow [Edit a project open in Desktop](./pbip.md#edit-a-project-open-in-desktop) before you edit and after you save; Desktop doesn't pick up file changes on its own.
- **Learn the project's own patterns.** Read `database.tmdl` (compatibility level), `model.tmdl` and one or two table files similar to what you'll write. Copy their indentation, quoting, property order and description style.
- **Use an existing object as the template** whenever the project has one of the same type. For an object type the project doesn't have yet, check the [TMDL documentation](https://learn.microsoft.com/analysis-services/tmdl/tmdl-overview) rather than writing from memory.

## Rules that break loading

These cause most load failures:

- **Indentation is structure.** Child objects and properties are indented one level deeper than their parent, consistently with tabs, as Power BI writes them. Multi-line expressions are indented deeper than the property that owns them, or wrapped in triple backticks.
- **Quote names** that contain spaces or special characters with single quotes, and double any single quote inside a name.
- **Descriptions** are `///` lines directly above the object. `//` comments are only allowed inside DAX or M expressions.
- **New objects don't need a lineage tag;** the engine assigns one. Never change or copy existing lineage tags.
- **Don't add internal annotations** (`PBI_*`) by hand, and keep the ones that exist.
- **Register what the project registers.** If `model.tmdl` lists `ref` entries for tables, roles, cultures or perspectives, add one for every new object of that kind and remove it when the object is deleted.

## Edit safely

- **Keep the project's file layout.** Power BI writes one file per table, role, culture and perspective, named after the object. TMDL also allows an object to be declared across several files, so find every declaration of an object before renaming or deleting it, and don't move objects between files.
- Edit only the lines for your change. Don't reformat files, reorder properties or normalize whitespace elsewhere.
- Renames are your job in file mode: update every reference in DAX expressions, sort-by columns, hierarchies, relationships, role filters, perspectives and translations, and search the report folders for the old name.
- Deleting an object means removing its definition, its `ref` entry and every reference to it.

## Validate

- If any parser is available (the modeling tools become available, a reload in Power BI Desktop, or a TOM-based script), load the folder and fix every error at the file and line it reports.
- Otherwise, review the full diff against the rules above and say clearly that the definition wasn't loaded. DAX can't be executed in file mode; hand the user the test queries from [test-model](../test-model.md).

## Complete model example

A minimal but complete Direct Lake model: three tables, a shared expression, relationships and measures. Use it to see how the files fit together, not as a template to copy.

`definition/database.tmdl`

```tmdl
database Sales
	compatibilityLevel: 1702
```

`definition/model.tmdl`

```tmdl
model Model
	culture: en-US
	defaultPowerBIDataSourceVersion: powerBI_V3
	discourageImplicitMeasures

ref table Date
ref table Product
ref table Sales
```

`definition/expressions.tmdl`

```tmdl
expression 'DirectLake - Sales' =
		let
		    Source = AzureStorage.DataLake("https://onelake.dfs.fabric.microsoft.com/<WorkspaceId>/<LakehouseId>", [HierarchicalNavigation=true])
		in
		    Source
```

`definition/tables/Date.tmdl`

```tmdl
/// Calendar at daily grain, one row per date.
table Date
	dataCategory: Time

	column Date
		dataType: dateTime
		isKey
		formatString: yyyy-mm-dd
		summarizeBy: none
		sourceColumn: calendar_date

	column 'Date Key'
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: date_key

	column Year
		dataType: int64
		formatString: 0
		summarizeBy: none
		sourceColumn: calendar_year

	column Month
		dataType: string
		summarizeBy: none
		sourceColumn: month_name
		sortByColumn: 'Month Number'

	column 'Month Number'
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: month_number

	hierarchy Calendar
		level Year
			column: Year
		level Month
			column: Month
		level Date
			column: Date

	partition Date = entity
		mode: directLake
		source
			entityName: dim_date
			schemaName: dbo
			expressionSource: 'DirectLake - Sales'
```

`definition/tables/Product.tmdl`

```tmdl
/// One row per product.
table Product

	column 'Product Key'
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: product_key

	/// Product name as shown to customers.
	column Product
		dataType: string
		summarizeBy: none
		isDefaultLabel
		sourceColumn: product_name

	partition Product = entity
		mode: directLake
		source
			entityName: dim_product
			schemaName: dbo
			expressionSource: 'DirectLake - Sales'
```

`definition/tables/Sales.tmdl`

```tmdl
/// Sales order lines: one row per product on an order.
table Sales

	/// Net revenue after discounts, excluding tax and freight.
	measure Revenue = SUM ( 'Sales'[Net Amount] )
		formatString: #,##0.00
		displayFolder: Revenue

	/// Cost of goods sold for the products on the order lines.
	measure Cost = SUM ( 'Sales'[Cost Amount] )
		formatString: #,##0.00
		displayFolder: Margin

	/// Revenue minus cost, before operating expenses.
	measure Margin = [Revenue] - [Cost]
		formatString: #,##0.00
		displayFolder: Margin

	/// Margin as a share of revenue. Blank when there is no revenue.
	measure 'Margin (%)' = DIVIDE ( [Margin], [Revenue] )
		formatString: 0.00%
		displayFolder: Margin

	/// Number of distinct orders. One order has one or more lines.
	measure Orders = DISTINCTCOUNT ( 'Sales'[Order Number] )
		formatString: #,##0
		displayFolder: Orders

	/// Average revenue per distinct order.
	measure 'Revenue per Order' =
			VAR _revenue = [Revenue]
			VAR _orders = [Orders]
			RETURN
			    DIVIDE ( _revenue, _orders )

		formatString: #,##0.00
		displayFolder: Revenue

	column 'Date Key'
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: order_date_key

	column 'Product Key'
		dataType: int64
		isHidden
		summarizeBy: none
		sourceColumn: product_key

	column 'Order Number'
		dataType: string
		summarizeBy: none
		sourceColumn: order_number

	column 'Net Amount'
		dataType: decimal
		isHidden
		formatString: #,##0.00
		sourceColumn: net_amount

	column 'Cost Amount'
		dataType: decimal
		isHidden
		formatString: #,##0.00
		sourceColumn: cost_amount

	column 'Discount Rate'
		dataType: double
		formatString: 0.0000%
		summarizeBy: none
		sourceColumn: discount_rate

	partition Sales = entity
		mode: directLake
		source
			entityName: fact_sales
			schemaName: dbo
			expressionSource: 'DirectLake - Sales'
```

`definition/relationships.tmdl`

```tmdl
relationship 'Sales to Date'
	fromColumn: Sales.'Date Key'
	toColumn: Date.'Date Key'

relationship 'Sales to Product'
	fromColumn: Sales.'Product Key'
	toColumn: Product.'Product Key'
```
