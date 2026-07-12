# TOTS language quickstart

Tree of Tables (TOTS) is a spreadsheet-first schema language for data that
naturally belongs in a tree. The short version:

- The folder path tells you what kind of thing you are looking at and where it
belongs.  Folder paths are part of the specification for the type, allowing 
a well-designed data model to control not only format but also organization and
indexing.
- Dynamic names ("buckets") in the path capture changing values such as a year, 
field, station id, etc.  These are "indexes": things that derive from a property
of the underlying data.  By putting them in the path, you can find the data
you're looking for without having to look at all the data first.
- A spreadsheet-centric schema for the "leaves" of the tree where the
spreadsheets live tells the parser how spreadsheet sections are supposed to 
be formated and how they can reliably become in-memory data for software
(in JSON form).
- When reading ToTs data with the parsing library, the result always has 
`data` (as JSON), `meta` (information about data like units), and `tots` 
(info needed to recreate a spreadsheet).
- ToTs supports many forms: folders containg .xlsx files, or .csv files, or
zip files of each.  csv's can also be grouped as individual "worksheets" 
combined into a single overall ToTs workbook.


## The core mental model

A TOTS resource is a tree-like folder path that ends in a spreadsheet-shaped 
leaf. For example, the path `ag/weather/station-index/home/year-index/2026.xlsx`
could break down as:

| Path part       | Meaning                             |
| --------------- | ----------------------------------- |
| `ag`            | Agriculture namespace               |
| `weather`       | Weather domain                      |
| `station-index` | Dynamic index                       |
| `home`          | Concrete station value              |
| `year-index`    | Dynamic bucket marker               |
| `2026.schema`   | Concrete year value and schema leaf |

In a schema repository where groups of schemas can live, the 
logical leaf is stored as a `.schema/` directorso the schema workbook, 
examples, expected JSON, attachments, and human notes can live together.

## What lives in a `.schema` leaf

A leaf package usually contains:

| File               | Purpose                                       |
| ------------------ | --------------------------------------------- |
| `schema.xlsx`      | Spreadsheet-authored schema for the leaf      |
| `schema.data.json` | Derived JSON Schema for parsed `data`         |
| `example.1.xlsx`   | Example spreadsheet input                     |
| `example.1.json`   | Expected parser output                        |
| `attachments/`     | Optional files referenced from workbook cells |
| `README.md`        | Optional human notes for this leaf            |

The most important authoring file is `schema.xlsx`. It has two primary sheets:

- `sections` describes the meaningful areas of the spreadsheet.
- `fields` describes the properties inside those sections.

## Spreadsheets are understood as sections

TOTS starts with the idea that a workbook is not just cells. It is a set of
sections. For example, a section might be a normal table:

| grower            | farm      | field    | acres |
| ----------------- | --------- | -------- | ----: |
| Ault Family Farms | Home Farm | North 80 |  79.6 |
| Ault Family Farms | Home Farm | South 40 |  41.2 |

Or it might be a form-like object:

| Key       | Value    |
| --------- | -------- |
| field_id  | NORTH-80 |
| crop_year | 2025     |
| crop      | corn     |

In `schema.xlsx`, the `sections` sheet says how to find each section and what
shape it has. The examples are both required sections on the `Harvest Map`
sheet.  A "sentinel-row" locateMode means the section is expected to start
with a row that contains the startPattern.

| sectionId      | kind     | locateMode     | startPattern      |
| -------------- | -------- | -------------- | ----------------- |
| `summary`      | `object` | `sentinel-row` | `HARVEST SUMMARY` |
| `yield_points` | `table`  | `sentinel-row` | `YIELD POINTS`    |

The `fields` sheet says which fields belong to each section.

| sectionId      | name          | type       | required |
| -------------- | ------------- | ---------- | -------- |
| `summary`      | `field_id`    | `string`   | `true`   |
| `summary`      | `crop_year`   | `integer`  | `true`   |
| `yield_points` | `timestamp`   | `datetime` | `true`   |
| `yield_points` | `yield_bu_ac` | `number`   | `true`   |

## How sections become JSON

Section kind controls the JSON shape.

| Kind          | JSON shape       | Use when                              |
| ------------- | ---------------- | ------------------------------------- |
| `table`       | Array of objects | The section has repeated records      |
| `object`      | One object       | The section has one entity or summary |
| `comment`     | String           | The section is narrative text         |
| `non-tabular` | Object           | The data is not naturally tabular     |

Current parser behavior is strongest for `table` and `object`. 

The table above might parse into:

```json
{
  "data": {
    "summary": {
      "field_id": "NORTH-80",
      "crop_year": 2025
    },
    "yield_points": [
      {
        "timestamp": "2025-10-14T18:00:00Z",
        "yield_bu_ac": 214.2
      }
    ]
  },
  "meta": {},
  "tots": {}
}
```

Please see LANGUAGE_learning.md or LANGUAGE_reference.md for details
on how `meta` and `tots` are formatted.

## `data`, `meta`, and `tots`

Every parsed workbook returns:

| Key    | Meaning                                  |
| ------ | ---------------------------------------- |
| `data` | Clean domain data                        |
| `meta` | Semantic spreadsheet metadata            |
| `tots` | Provenance, diagnostics, and tool output |

Use `data` for application logic. Use `meta` when spreadsheet metadata matters.
Use `tots` for debugging, provenance, diagnostics, and tooling.


## Dynamic Indexing Buckets

Dynamic buckets let a schema path contain variable values without needing a
separate hard-coded schema for every value. `_defs.xlsx` currently defines
buckets such as:

| Bucket          | Example path          | Captured context               |
| --------------- | --------------------- | ------------------------------ |
| `year-index`    | `year-index/2026`     | `{ "year-index": "2026" }`     |
| `field-index`   | `field-index/north80` | `{ "field-index": "north80" }` |
| `station-index` | `station-index/home`  | `{ "station-index": "home" }`  |
| `field-name-index-inplace` | data path segment `north80` | `{ "field-name-index": "north80" }` |

The parser records captured values in `tots.pathContext`. Your spreadsheet may
also include the same values in `data` if they are part of the workbook's
semantic data.  In general, indexes like this typically reflect some property
of the underlying stored data.  For `year-index/2026`, for example, one would
expect to find data under that index containing timestamps or dates that fall
within the year 2026.  

There are two implemented matching modes. `segment-after-name` keeps the bucket
name and value as adjacent path segments, such as `year-index/2026`.
`segment-in-place` uses a schema placeholder ending in `-inplace`, such as
`field-name-index-inplace.schema`, while the data tree contains only the actual
value, such as `north80.xlsx`. Reusable base definitions like `name-index` let
schema authors create prefixed placeholders such as `field-name-index-inplace`
and `ration-name-index-inplace` without adding every possible name kind to
`_defs.xlsx`.

## Typed rows and units

Some spreadsheet rows are not data rows. A table can include a row-type column,
usually `_row_type`.

| _row_type | timestamp            | yield_bu_ac | moisture_pct |
| --------- | -------------------- | ----------: | -----------: |
| `UNITS`   |                      |       bu/ac |            % |
| `DATA`    | 2025-10-14T18:00:00Z |       214.2 |         18.4 |
| `COMMENT` | This is a comment.   | So is this. |              |

`UNITS` rows are captured in `meta.<section>.units`, not in `data`. `DATA` marks
an ordinary data row when a row-type column is present. 

## Field types

Use field types to tell TOTS how values should be interpreted and how JSON
Schema should be derived. Common choices:

- Use `string` for names, IDs, labels, notes, and text.
- Use `integer` for whole numbers such as crop years or intervals.
- Use `number` for measurements and decimals.
- Use `boolean` for yes/no values.
- Use `date` for calendar dates.
- Use `datetime` for timestamps.
- Use `geometry` for WKT or compact geometry text.
- Use `image` for relative paths to image attachments, with PNG as the
recommended default.
- Use `attachment` or `binary` for relative paths to non-image files.

## Default-first authoring

TOTS schemas should describe the spreadsheet people actually use. Do not
over-specify a normal spreadsheet. Useful defaults:

- Blank `sheet` means the first worksheet.
- Blank `kind` means `table`.
- Blank `locateMode` means ordered/default discovery.
- Blank section `required` means `true`.
- Blank field `required` means `false`.
- Blank field `type` means `string`.
- Blank `rowTypeColumn` means every non-empty table row is data except headers.

Add details only when the workbook layout needs them.

## A practical authoring workflow

1. Pick the tree path for the resource based on it's semantic relationship 
to the rest of the tree.
2. Identify dynamic index bucket segments in that path.
3. Open the real spreadsheet and determine its meaningful sections.
4. Decide whether each section is a `table`, `object`, or another section kind.
5. List fields for each section.
6. Add field types only where the default `string` is not enough.
7. Add typed rows if units or explicit data-row markers are useful.
8. Add example workbooks and expected parsed JSON.
9. Validate that the derived data schema and parser output match the examples.

## What to read next

- `LANGUAGE_learning.md` teaches the language step by step.
- `LANGUAGE_reference.md` lists the language surface, defaults, files, columns,

parser behavior, and current limitations.

- `schema/tots/README.md` explains this repository's schema root and fixture

package layout.
