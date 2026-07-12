# Learning the TOTS language

Tree of Tables (ToTs) is built for a common problem: real data often lives in
spreadsheets.  Humans use spreadsheets when left to their own devices.
The flexibility of spreadsheets provides most non-developer people with
a tool that they can adequately apply to most of their data needs.  However,
this flexibility is the bane of software developers and data managers, and
even in some cases the original humans themselves as critical context can
be lost from origination to later re-use, especially as data needs scale.

So despite the software and data world treating spreadsheets as second-class,
real data in the real world stubbornly refuses to escape the spreadsheet.
Rather than taking the traditional approach of lamenting foolish lay-person
users while imagining a future utopia where spreadsheets have been relegated
to the wastebin of history, ToTs inverts that model.  ToTs embraces the
spreadsheet as first-class, and provides the tooling, schema language, and 
logical geography definitions necessary to capture context, provide validation,
and handle traditional data management best practices.  

This guide teaches the ToTs language in the following order:

1. Tree schemas
2. Dynamic index buckets
3. Leaf packages
4. Parsing results
5. Spreadsheet "sections"
6. Section kinds
7. Fields and field types
8. Typed rows and metadata
9. Attachments
10. Schema discovery and catalogs

## 1. Tree schemas

A TOTS schema is not only a workbook schema. It is a folder tree of 
possible resources.  For example, this path says more than “there 
is a spreadsheet here”:

```text
ag/harvest/yield-moisture/year-index/2026/field-index/north80.xlsx
```

The path tells us:

| Segment          | What it contributes               |
| ---------------- | --------------------------------- |
| `ag`             | Domain namespace                  |
| `harvest`        | Subject area                      |
| `yield-moisture` | Resource family                   |
| `year-index`     | The next segment is a year value  |
| `2026`           | A concrete year in the path       |
| `field-index`    | The next segment is a field value |
| `north80.xlsx`   | The concrete field workbook       |

A typical person stores their spreadsheets in a structure like this,
and it is from this structure that they derive much-needed context to 
know what to expect from the spreadsheet.  This "_logical geography_"
is included in ToTs as the first-class means to instruct users and
parsers alike about the "_ambient context_" of the data: what kind
of thing is it, where can the schema be found, etc.

The workbook at the leaf then explains the spreadsheet layout for that resource.
This is the first major idea: TOTS combines route-like context with
spreadsheet-like structure.

## 2. Dynamic Indexing Buckets

A dynamic index bucket is a named path segment that captures a changing 
value from the next segment. The language-level definitions live in 
`_defs.xlsx` on the `dynamic-bucket-defs` sheet. Current fixtures define:

| Bucket          | Value type | Regex          | Example   |  matchMode           |
| --------------- | ---------- | -------------- | --------- | -------------------- |
| `field-index`   | `string`   | `^[a-z0-9-]+$` | `north80` | `segment-after-name` |
| `id-index`      | `string`   | `^[a-z0-9-]+$` | `north80` | `segment-after-name` |
| `name-index`    | `string`   | `^[a-z0-9-]+$` | `north80` | `segment-after-name` |
| `station-index` | `string`   | `^[a-z0-9-]+$` | `home`    | `segment-after-name` |
| `year-index`    | `integer`  | `^\d{4}$`      | `2026`    | `segment-after-name` |

The common `matchMode` is `segment-after-name`. That means this path:

```text
ag/weather/station-index/home/year-index/2026.schema
```

captures:

```json
{
  "station-index": "home",
  "year-index": "2026"
}
```

Those values appear in `tots.pathContext`. Dynamic bucket values are provenance
by default. If a workbook also needs the value as domain data, model it as a
field in `schema.xlsx`. It is recommended to include the values in the underlying
data where possible as this is typically what a human author of a spreadsheet 
would do.

Schemas can also use `segment-in-place` when the data path should contain only
the value. For example, the schema leaf
`ag/planting/as-planted/year-index/2026/field-name-index-inplace.schema` matches
the data workbook `ag/planting/as-planted/year-index/2026/north80.xlsx` and
captures:

```json
{
  "year-index": "2026",
  "field-name-index": "north80"
}
```

The `field-name-index-inplace` segment is a schema placeholder; it should not be
created literally in a data tree. The reusable `name-index` definition supplies
the regex for both `field-name-index-inplace` and
`ration-name-index-inplace`.

Use dynamic buckets when:

- The same schema shape repeats for many years, fields, stations, growers,
accounts, or similar values.
- The path value is important context.
- You do not want to duplicate identical schemas for every concrete value.

Avoid dynamic buckets when:
- The path segment is a fixed semantic category, not a changing value.
- The value belongs only inside workbook data and not in the resource path.

## 3. Leaf packages

In a real data tree, a leaf might be a workbook such as `field_boundaries.xlsx`.
A repository of schemas (as opposed to a real ToTs tree containing actual data),
mirrors the structure of the data tree but replaces the spreadsheet leaves with
a `.schema/` directory.  This directory contains files such as:

| File               | Role                                           |
| ------------------ | ---------------------------------------------- |
| `schema.xlsx`      | The source schema workbook                     |
| `schema.data.json` | Derived JSON Schema for `data`                 |
| `example.N.xlsx`   | Example workbook input                         |
| `example.N.json`   | Expected parser result                         |
| `attachments/`     | Binary files referenced by examples            |
| `schema.meta.json` | Optional leaf-level metadata schema override   |
| `schema.tots.json` | Optional leaf-level provenance schema override |
| `README.md`        | Optional human explanation for the leaf        |

The `.schema` directory is an authoring and test package. It is not saying
production data must be stored as directories.  In the production data,
the `.schema` directory becomes a `.xlsx` file with the same name prefix 
as the `.schema` directory.

## 4. Parse results

Every parsed workbook returns the same outer shape:

```json
{
  "data": {},
  "meta": {},
  "tots": {}
}
```

### `data`

`data` is the clean domain data. It should be safe for applications to use
without understanding spreadsheet layout. Table sections become arrays. Object
sections become objects.  "Sections" and their types will be explained below.

Every ToTs schema contains enough information to automatically derive a JSON
schema that defines the shape of the `data` object.

### `meta`

`meta` holds semantic spreadsheet metadata. Units are the currently implemented
as part of `meta`.  Units can be specified dynamcally in a spreadsheet or 
universally via the schema, and therefore they consistently get returned in the
`meta`.  For example:

```json
{
  "data": {},
  "meta": {
    "yield_points": {
      "units": {
        "yield_bu_ac": "bu/ac",
        "moisture_pct": "%"
      }
    },
  },
  "tots": {}
}
```

The language-level metadata schema also allows comments and settings.

### `tots`

`tots` holds parser and tooling information:

- `version`
- `leafPath`
- `pathContext`
- `encoding`
- `sheets`
- `sections`
- `input`
- `schema`
- `sources`
- `diagnostics`

Use it for provenance, debugging, validation warnings, and future
reconstruction.

## 5. Workbooks are made of sections

The central spreadsheet concept in TOTS is the "_section_". A section is a
meaningful area of a workbook. It can be an entire worksheet or a block inside a
worksheet. For example, a single worksheet can contain an object-style section
("HARVEST_SUMMARY" below), followed by a table section:

| A               | B                    | C           | D            |
| --------------- | -------------------- | ----------- | ------------ |
| HARVEST SUMMARY |                      |             |              |
| field_id        | NORTH-80             |             |              |
| crop_year       | 2025                 |             |              |
|                 |                      |             |              |
| YIELD POINTS    |                      |             |              |
| `_row_type`     | timestamp            | yield_bu_ac | moisture_pct |
| UNITS           |                      | bu/ac       | %            |
| DATA            | 2025-10-14T18:00:00Z | 214.2       | 18.4         |

The matching `sections` worksheet of the schema (`schema.xlsx`) could 
contain the rows below. Both sections are on the `Harvest Map` worksheet 
within the spreadsheet itself, and `yield_points` also declares `_row_type` 
as its `rowTypeColumn`.

| sectionId      | kind     | locateMode     | startPattern      |
| -------------- | -------- | -------------- | ----------------- |
| `summary`      | `object` | `sentinel-row` | `HARVEST SUMMARY` |
| `yield_points` | `table`  | `sentinel-row` | `YIELD POINTS`    |

The matching `fields` sheet in the schema definition says which fields are recognized:

| sectionId      | name           | type       | required |
| -------------- | -------------- | ---------- | -------- |
| `summary`      | `field_id`     | `string`   | `true`   |
| `summary`      | `crop_year`    | `integer`  | `true`   |
| `yield_points` | `timestamp`    | `datetime` | `true`   |
| `yield_points` | `yield_bu_ac`  | `number`   | `true`   |
| `yield_points` | `moisture_pct` | `number`   | `true`   |

## 6. Section kinds

### `table`

Use `table` for repeated rows with a header row. Spreadsheet shape:

| field    | acres | crop     |
| -------- | ----- | -------- |
| North 80 |  79.6 | corn     |
| South 40 |  41.2 | soybeans |

Parsed shape:

```json
{
  "fields": [
    { "field": "North 80", "acres": 79.6, "crop": "corn" },
    { "field": "South 40", "acres": 41.2, "crop": "soybeans" }
  ]
}
```

Choose `table` when:

- There can be zero, one, or many records.
- Each row has the same field names.
- A human would describe the area as a list, log, inventory, map points, line
items, readings, lookup sheet, etc.

### `object`

Use `object` for one "thing" represented as key/value rows. Spreadsheet shape:

| Key       | Value    |
| --------- | -------- |
| field_id  | NORTH-80 |
| crop_year | 2025     |
| crop      | corn     |

Parsed shape:

```json
{
  "summary": {
    "field_id": "NORTH-80",
    "crop_year": 2025,
    "crop": "corn"
  }
}
```

Choose `object` when:

- The section describes one entity, configuration, report summary,
application event, bin, station, ration, etc.
- The spreadsheet is naturally form-like.

### `comment`

`comment` is for raw or lightly structured notes.   They are parsed
and returned, but unstructured, and generally should be ignored.


### `non-tabular`

`non-tabular` is for small payloads that are not ordinary tables or key/value
objects. It is part of the normalized language surface, but parser behavior is
reserved for future expansion. Use it only when you are deliberately modeling a
non-table layout and are ready to define parser behavior for it.

## 7. Finding sections in worksheets

TOTS should support ordinary spreadsheets without forcing people to redesign
them. That is why section location is explicit but default-first.

### Ordered default

If `locateMode` is blank, the parser uses the ordered/default behavior. Today
that means it starts at the first non-blank row for the selected sheet and reads
until the next blank row. Use this for simple one-section worksheets.

### `sentinel-row`

`sentinel-row` means a row marks the start of a section.  In the example below,
"REPORT" is the sentinel.  Example:

| A          | B          |
| ---------- | ---------- |
| REPORT     |            |
| scout_date | 2025-07-15 |
| crop       | corn       |

Use this when multiple sections share a worksheet or a section needs a
human-readable title.

### `fixed-range`

`fixed-range` means the section lives in a known cell range, described by fields
such as `startCell` and `endCell`. This is part of the language vocabulary, but
it is generally not very sustainable to define a spreadsheet with hard-coded 
cells.

### Future expansion

It is inteded that additional section types and addressing will be added in the
future.


## 8. Fields and field types

Fields are the properties inside a section. The `fields` sheet has one row per
field.

| sectionId | name              | type     | required | notes             |
| --------- | ----------------- | -------- | -------- | ----------------- |
| `images`  | `image_date`      | `date`   | `true`   |                   |
| `images`  | `ndvi_image_path` | `image`  | `true`   | Relative PNG path |
| `images`  | `mean_ndvi`       | `number` | `true`   |                   |

Use field types as semantic hints and JSON Schema derivation inputs.

| Type         | Use for                      |
| ------------ | ---------------------------- |
| `string`     | Text, names, IDs, labels     |
| `integer`    | Whole numbers                |
| `number`     | Measurements and decimals    |
| `boolean`    | Yes/no values                |
| `date`       | Calendar dates               |
| `datetime`   | Timestamps                   |
| `geometry`   | WKT or compact geometry text |
| `image`      | Relative path to an image    |
| `attachment` | Relative path to a file      |
| `binary`     | Relative path to binary data |

Derived behavior is covered exhaustively in `LANGUAGE_reference.md`. In short,
numbers become numeric JSON values, dates and datetimes are strings with JSON
Schema formats, and attachments remain relative-path strings with diagnostics.

Guidance:
- Keep spreadsheets human-friendly.  Humans tend to use recognizable strings
as identifiers.  For all their flaws, and there are many, this is how typical
spreadsheets will be created and consumed by humans.  Translating string names
into ID's should be the job of backend systems and not burden humans with that
process if it can be avoided.
- Add optional ID fields when downstream systems have stable IDs and no other
means of relating names to ID's.
- Use `number` for values that might contain decimals even if examples happen to
be whole numbers.
- Use `integer` only when fractional values are invalid.

## 9. Typed rows and metadata

A typed row is a row in a table that has a special role. Typed rows are enabled
by a `rowTypeColumn` which can be uniquely specified per-schema, 
commonly `_row_type`.

| _row_type | timestamp            | yield_bu_ac | moisture_pct |
| --------- | -------------------- | ----------: | -----------: |
| UNITS     |                      |       bu/ac |            % |
| DATA      | 2025-10-14T18:00:00Z |       214.2 |         18.4 |

The standard typed-row definitions currently available in `_defs.xlsx` are:
- `UNITS`: captures per-column units in `meta.<section>.units`. This is
currently implemented.
- `DATA`: marks ordinary data rows when a row-type column exists. This is
currently implemented.
- `COMMENT`: intended to capture annotation metadata. It is defined but not
fully parsed today.
- `SETTINGS`: intended to capture section settings. It is defined but not fully
parsed today.

Use typed rows when:
- Units are important and belong near the data in the source spreadsheet.
- A workbook mixes metadata and data rows in the same table.
- You want explicit `DATA` rows to make the table easier to validate.

Avoid typed rows when:
- The table is already a simple header row plus data rows.
- Units can be unambiguously represented in field names or schema documentation.

## 10. Attachments and images

Spreadsheets often point to photos, drone imagery, scans, PDFs, or other 
non-spreadsheet files. TOTS models those as relative paths in cells. Example:

| image_date | ndvi_image_path                      | mean_ndvi |
| ---------- | ------------------------------------ | --------: |
| 2025-06-10 | `attachments/ndvi-2025-06-10-v6.png` |      0.62 |

This would mean that in the same directory as the spreadsheet, there is a folder
named `attachments` that contains a file named `ndvi-2025-06-10-v6.png`.

For image fields, use type `image`. PNG is the recommended default image format
for long-term usability/readability in generic systems.  For general files, use 
`attachment` or `binary`. Attachment path rules:
- Paths must be relative to the workbook location.
- Path segments must be separated by `/` and use standard path-supported
characters.  *No backslashes allowed.*
- Do not use absolute paths.
- Do not use URLs.
- Do not use `..` traversal.
- Missing or unsafe attachments are reported as diagnostics; parsing can still
return data.

## 11. Optional sections and lookup sheets

Many real spreadsheets have optional helper sheets. The field-boundaries example
demonstrates this pattern:

| Sheet     | Required? | Why it exists                   |
| --------- | --------- | ------------------------------- |
| `fields`  | Yes       | The core field inventory        |
| `growers` | No        | Optional grower details and IDs |
| `farms`   | No        | Optional farm details and IDs   |

Use optional sections when:

- The workbook is useful without that section.
- Some users have extra normalization or lookup data and others do not.
- You want one schema to support both simple and fully built-out spreadsheets.

## 12. Schema discovery and catalogs

The local folder tree is not the only way to provide schemas. A project can use:

- `.tots-manifest.json` to point to its schema root.
- `manifest.json` inside a schema root to index leaves, avoiding deep trees.
- `schema.catalog.json` to compose multiple schema roots.
- `package.json.tots` metadata for packaged schema roots.

This allows a given project to define it's own schemas and tell the library 
where they are in the codebase, as well as compose multiple schemas from multiple
sources together into a single, merged ToTs tree.

Catalog sources can be local folders, URLs, Git repositories, npm packages,
bundles, or project manifests. The parser preserves schema source provenance in
`tots.schema` and `tots.sources.schema`.

## 13. Current implementation status

The language intentionally has a broader vocabulary than the current parser
implementation. Implemented strongly today:

- `.xlsx` workbook parsing.
- `table` sections.
- `object` sections.
- Ordered/default and `sentinel-row` section discovery.
- Field type coercion and JSON Schema derivation.
- `UNITS` and `DATA` typed-row handling.
- Dynamic bucket extraction with `segment-after-name` and `segment-in-place`.
- Attachment path diagnostics.
- Schema root, project manifest, and catalog resolution in Node.

Reserved or limited today:

- Full `comment` and `non-tabular` parsing.
- `fixed-range`, `startCell`, `endCell`, transposed orientation, and advanced
layout behavior.
- `COMMENT` and `SETTINGS` typed-row parsing.
- Formula/style preservation.
- High-fidelity spreadsheet reconstruction.
- Strict GeoJSON validation.

## 14. Authoring checklist

When creating a new schema:

1. Choose the tree path based on semantic nature of the data.
2. Decide indexing by adding dynamic bucket positions.
3. Create the `.schema/` leaf.
4. Add `schema.xlsx`.
5. List sections first.
6. List fields second.
7. Prefer defaults for ordinary spreadsheets.
8. Add types only where they matter.
9. Add optional sections for helper sheets.
10. Add typed rows only when metadata lives in rows.
11. Add examples that represent real user spreadsheets.
12. Generate JSON schema and example JSON for spreadsheets.
12. Compare parsed output with expected JSON.
13. Document any behavior that is not parser-supported yet.
