# TOTS language reference

This reference lists the current Tree of Tables language and the
implementation status of each major feature. Status labels:

- **Implemented**: Current parser behavior handles this directly.
- **Derived/validated**: Current tooling derives schemas or validates examples
for this feature.
- **Normalized**: Current code reads and preserves the vocabulary, but parser
behavior is limited.
- **Reserved**: Documented language direction, not complete parser behavior.

## Top-level language concepts

TOTS combines:
- A route-like folder tree path.
- Dynamic indexing buckets.
- Spreadsheet workbook sections in the leaves of the tree path.
- Field definitions for each section.
- Parser output split into `data` (actual data), `meta` (information
about the data derived from spreadsheet or shema), and `tots` (detailed parsing
info needed to debug, determine provenance, and fully reconstruct spreadsheets).

## Schema root files

A schema root is usually `schema/tots`. Files:
- `_defs.xlsx`: language-level dynamic bucket and typed-row definitions.
**Implemented**.
- `manifest.json`: schema-root manifest listing fixture leaves and examples.
**Implemented**.
- `schema.meta.json`: global JSON Schema for parsed `meta`. **Implemented**.
- `schema.tots.json`: global JSON Schema for parsed `tots`. **Implemented**.
- `schema.catalog.json`: example or project catalog for composing schema roots.
**Implemented in Node resolution**.
- `schema.catalog.schema.json`: JSON Schema for catalog files.
**Derived/validated**.
- `schema.root-manifest.schema.json`: JSON Schema for project
`.tots-manifest.json`. **Derived/validated**.

## Leaf package files
A schema leaf is a directory whose name ends in `.schema`. Files:
- `schema.xlsx`: source workbook schema. **Implemented**.
- `schema.data.json`: derived model-specific JSON Schema for parsed `data`.
**Implemented**.
- `example.N.xlsx`: example workbook input. **Implemented**.
- `example.N.json`: expected parser output. **Implemented**.
- `attachments/`: optional binary files referenced by example workbooks.
**Implemented for path checks**.
- `schema.meta.json`: optional leaf-level metadata schema override.
**Implemented by loader**.
- `schema.tots.json`: optional leaf-level provenance schema override.
**Implemented by loader**.
- `README.md`: optional human notes. **Documentation only**.

## Data path and schema leaf path

Logical data paths end in a data serialization such as `.xlsx`. Schema leaf
paths end in `.schema`. Current conversion examples:
- Data path: `ag/fields/field_boundaries.xlsx`
Schema path: `ag/fields/field_boundaries.schema`
- Data path: `ag/weather/station-index/home/year-index/2026.xlsx`
Schema path: `ag/weather/station-index/home/year-index/2026.schema`
- Data path: `ag/planting/as-planted/year-index/2026/north80.xlsx`
Schema path: `ag/planting/as-planted/year-index/2026/field-name-index-inplace.schema`

The parser currently supports `.xlsx` as the parsed workbook/data serialization.
`csv`, `csvf`, and `csvf.zip` are listed as canonical future serialization
targets in provenance, but parsing them is deferred.  `csvf` means a folder 
containing multiple CSV files, each of which represents a "worksheet" in an
overall "workbook", allowing CSV's to gain that spreadsheet-level capability.
`csvf.zip` is just that csv folder zipped into a zip file.

## Parse result contract

Every parsed workbook returns:

```json
{
  "data": {},
  "meta": {},
  "tots": {}
}
```

### `data`

Status: **Implemented**. `data` is clean semantic domain data. Derivation:

- `table` sections become arrays of objects.
- `object` sections become objects.
- `comment` sections derive as strings.
- Other non-table/object sections derive as objects today.

### `meta`

Status: **Implemented for units; broader shape reserved**. `meta` contains
semantic spreadsheet metadata associated with sections. Common keys:
- `meta.<section>.units`: per-column units. **Implemented for `UNITS` typed
rows**.
- `meta.<section>.comments`: comments or annotations. **Reserved**.
- `meta.<section>.settings`: section settings. **Reserved**.

### `tots`
Status: **Implemented**. `tots` contains parser provenance and
reconstruction/tooling metadata. Common keys:
- `version`: current value is `tots/v1`.
- `leafPath`: schema leaf path supplied by the resolved schema.
- `pathContext`: dynamic bucket values captured from the schema path.
- `encoding`: input and canonical serialization hints.
- `sheets`: workbook sheet order.
- `sections`: normalized section metadata used by the parser.
- `input`: input path, archive entry, TOTS path, schema leaf path, and TOTS path
source.
- `schema`: resolved schema source summary.
- `sources`: root discovery and schema source provenance.
- `diagnostics`: non-fatal diagnostics such as attachment warnings.

### `trace`

Status: **Implemented when requested**. `trace` is an optional sibling of
`data`, `meta`, and `tots`, returned when parser options request trace output.
It records section-level parse details such as start rows, header rows, parsed
record cells, skipped typed rows, JSON paths, and schema paths. Trace output is
intended for inspection and UI tooling, not as domain data.

## `_defs.xlsx`

`_defs.xlsx` contains language-level definitions shared by schemas.

### `dynamic-bucket-defs`

Status: **Implemented for `segment-after-name` and `segment-in-place`**.
Columns:

- `name`: bucket marker segment, such as `year-index`.
- `bucketType`: semantic bucket category, such as `year` or `id`.
- `description`: human-readable explanation.
- `valueType`: expected value type.
- `regex`: pattern used when extracting a value from a leaf-name segment.
- `exampleValue`: representative value.
- `matchMode`: matching strategy. Current implemented strategies are
`segment-after-name` and `segment-in-place`.

Current generated definitions:

| name            | example | matchMode            |
| --------------- | ------- | -------------------- |
| `field-index`   | north80 | `segment-after-name` |
| `id-index`      | north80 | `segment-after-name` |
| `name-index`    | north80 | `segment-after-name` |
| `station-index` | home    | `segment-after-name` |
| `year-index`    | 2026    | `segment-after-name` |
`field-index`, `station-index`, `id-index`, and `name-index` are string buckets.
`year-index` is an integer year bucket. `name-index` and `id-index` are reusable
base definitions: schema segments such as `field-name-index-inplace` or
`ration-name-index-inplace` derive their regex and value type from `name-index`
without needing one-off `_defs.xlsx` rows.

Extraction behavior:
- For `ag/weather/station-index/home/year-index/2026.xlsx`, the parser
captures `station-index = home` and `year-index = 2026`.
- For schema leaf `ag/planting/as-planted/year-index/2026/field-name-index-inplace.schema`
and data path `ag/planting/as-planted/year-index/2026/north80.xlsx`, the parser
captures `year-index = 2026` from `segment-after-name` and
`field-name-index = north80` from `segment-in-place`.
- For schema leaf `ag/irrigation/application-records/year-index-inplace.schema`
and data path `ag/irrigation/application-records/2026.xlsx`, the parser captures
`year-index = 2026`.

`segment-in-place` placeholders are schema-tree markers only. The literal
placeholder segment, including the `-inplace` suffix, does not appear in the
data tree. Exact `_defs.xlsx` names take precedence over derived suffix matches,
and ambiguous schema matches fail instead of picking one arbitrarily.

### `typed-row-defs`

Status: **Definitions implemented; parser support partial**. Columns:
- `setName`: reusable typed-row set name, such as `standard`.
- `name`: row marker value, such as `UNITS`.
- `outputKey`: target metadata key.
- `occurrence`: expected occurrence count.
- `parseMode`: how cells should be interpreted.
- `appliesTo`: section kinds where the row type applies.
- `behavior`: human-readable behavior.
- `includeInData`: whether rows of this type become normal `data`.
- `description`: human-readable explanation.

Current standard definitions:

| name       | outputKey  | Status      |
| ---------- | ---------- | ----------- |
| `UNITS`        | `units`         | Implemented |
| `COMMENT`      | `comments`      | Implemented |
| `SETTINGS`     | `settings`      | Implemented for key/value JSON in the second cell |
| `DATA`         |                 | Implemented |
| `MODUSTESTID`  | `modusTestIds`  | Implemented (per-column strings; not an enum of Modus codes) |

`UNITS`, `COMMENT`, `SETTINGS`, and `MODUSTESTID` are excluded from `data`.
`DATA` rows are ordinary data rows. If `rowTypeColumn` is absent, a row is
classified when any cell equals a typed-row name (`UNITS`, `COMMENT`,
`MODUSTESTID`, `DATA`).

Section `additionalProperties` (default `false`) is **Implemented**. When
`true`, unknown table columns and object keys are kept in `data` and the
derived JSON Schema allows them. Use this for open analyte sets such as Modus
soil lab results.

## `schema.xlsx`

`schema.xlsx` is the source schema workbook for a leaf. Current primary sheets:

- `sections`: section definitions. **Implemented/normalized**.
- `fields`: field definitions. **Implemented/normalized**.

Unknown advanced columns should be ignored by current readers unless they are
documented and implemented later.

## `sections` sheet

Each row defines one section. Columns:
- `sectionId`: stable key for this section in `data`, `meta`, and
`tots.sections`. Required for useful rows. **Implemented**.
- `kind`: section kind. Blank defaults to `table`. **Implemented for `table` and
`object`; derived/normalized for others**.
- `required`: whether the section must be present. Blank defaults to `true`.
**Implemented**.
- `sheet`: worksheet name. Blank means first worksheet. **Implemented**.
- `locateMode`: how to find the section. Blank defaults to `ordered-default`.
**Implemented for ordered/default and `sentinel-row`; reserved for
`fixed-range`**.
- `order`: section ordering. Blank defaults to source row order.
**Implemented**.
- `startPattern`: sentinel text for `sentinel-row`. **Implemented**.
- `startColumn`: intended column for finding a start marker.
**Normalized/reserved**.
- `requiredHeaders`: pipe-delimited expected headers. **Normalized/reserved for
parser validation**.
- `endMode`: how the section ends. Blank defaults to
`before-next-section-or-blank-gap`. **Normalized; current parser ends at blank
row for implemented section types**.
- `minBlankRows`: number of blank rows that mark a gap. Blank defaults to `1`.
**Normalized; current parser uses blank row behavior**.
- `rowTypeColumn`: header name for typed rows, commonly `_row_type`.
**Implemented**.
- `rowTypeSet`: typed-row set name. **Normalized; current parser behavior is
hard-coded for standard row names**.
- `startCell`: fixed range start. **Normalized/reserved**.
- `endCell`: fixed range end. **Normalized/reserved**.
- `orientation`: row or future transposed orientation. Blank defaults to `row`.
**Normalized; non-row behavior reserved**.
- `payloadFormat`: intended payload format for non-tabular sections.
**Normalized/reserved**.
- `notes`: human notes. **Documentation/inspection only**.

## Section kinds

### `table`

Status: **Implemented**. Meaning: header row plus data rows. JSON shape: array
of objects. Use when the spreadsheet section contains repeated records, logs,
line items, map points, inventories, readings, optional lookup sheets, or other
row-shaped data.

### `object`

Status: **Implemented**. Meaning: key/value rows. JSON shape: object. Use when
the spreadsheet section describes one thing: a report header, summary,
application, station, bin, ration, budget header, or configuration-like block.

### `comment`

Status: **Derived/normalized; parser behavior limited**. Meaning: raw or lightly
structured notes. JSON Schema derivation maps it to a string. Use when narrative
notes are a first-class section.

### `non-tabular`

Status: **Normalized/reserved**. Meaning: small payloads that are not ordinary
tables. Use only when the schema explicitly needs a non-table layout and parser
behavior is defined for it.

## Locate modes

### Blank or `ordered-default`

Status: **Implemented**. Meaning: find the first non-blank row on the selected
sheet for implemented section types. Best for one simple section per sheet.

### `sentinel-row`

Status: **Implemented**. Meaning: find a row containing `startPattern`; parse
the section after that marker. Best for multiple titled sections on one
worksheet.

### `fixed-range`

Status: **Reserved**. Meaning: parse from `startCell` to `endCell`. Best for
rigid templates, once parser behavior is implemented.

## `fields` sheet

Each row defines one field in one section. Columns:

- `sectionId`: section that owns the field. Required. **Implemented**.
- `name`: field/property name. Required. **Implemented**.
- `type`: field type. Blank defaults to `string`. **Implemented**.
- `required`: whether parsed records must include the field. Blank defaults to
`false` in the parser. **Implemented in JSON Schema validation**.
- `sourcePath`: source-specific location or extraction hint.
**Normalized/reserved**.
- `fkResource`: foreign-key related resource. **Normalized/reserved**.
- `fkField`: foreign-key related field. **Normalized/reserved**.
- `fkName`: human-readable relationship name for foreign key. **Normalized/reserved**.
- `notes`: human notes. **Documentation/inspection only**.

## Field types

### `string`

Status: **Implemented**. Default field type. Use for text, names, IDs, labels,
notes, and values that should not be numerically coerced.

### `integer`

Status: **Implemented**. JSON Schema: `{ "type": "integer" }`. Parser behavior:
finite numeric values are truncated with `Math.trunc`; otherwise the original
value remains and validation can fail. Use for crop years, whole-number counts,
intervals, and values where fractions are invalid.

### `number`

Status: **Implemented**. JSON Schema: `{ "type": "number" }`. Use for
measurements, rates, acres, coordinates, percentages, money, and decimals.

### `boolean`

Status: **Implemented**. JSON Schema: `{ "type": "boolean" }`. Accepted
truthy/falsy text includes `true`, `yes`, `1`, `false`, `no`, and `0`.

### `date`

Status: **Implemented as string coercion and JSON Schema format**. JSON Schema:
`{ "type": "string", "format": "date" }`. Use ISO-like calendar dates such as
`2025-07-15`.

### `datetime`

Status: **Implemented as string coercion and JSON Schema format**. JSON Schema:
`{ "type": "string", "format": "date-time" }`. Use timestamp values such as
`2025-10-14T18:00:00Z`.

### `geometry`

Status: **Implemented as string; strict geometry validation reserved**. JSON
Schema: string with description. Use WKT or compact geometry text in current
milestones.

### `image`

Status: **Implemented as string plus attachment diagnostics**. JSON Schema:
string with `contentMediaType: "image/png"`. Use for relative paths to image
attachments. PNG is the recommended default.

### `attachment`

Status: **Implemented as string plus attachment diagnostics**. Use for relative
paths to general files.

### `binary`

Status: **Implemented as string plus attachment diagnostics**. Alias-style
binary reference for relative file paths.

## Defaults

Defaults are a part of the language to simplify schema creation for well-formed
spreadsheets:

- Blank `kind` means `table`.
- Blank section `required` means `true`.
- Blank `sheet` means the first worksheet.
- Blank `locateMode` means `ordered-default`.
- Blank `order` means schema row order.
- Blank `endMode` means `before-next-section-or-blank-gap`.
- Blank `minBlankRows` means `1`.
- Blank `orientation` means `row`.
- Blank field `type` means `string`.
- Blank field `required` means `false` in the parser.
- Blank `rowTypeColumn` means all non-empty table rows are data rows.

## Table parsing behavior

Status: **Implemented**. For a `table` section:
1. Find the section start.
2. Use the first non-blank row as the header row.
3. Match headers to fields by exact field name.
4. Stop at the next blank row.
5. Skip `UNITS` typed rows and write their values to `meta.<section>.units`.
6. Skip non-empty row types other than `DATA`.
7. Parse blank row-type or `DATA` rows as data.
8. Omit blank cell values from records.

## Object parsing behavior

Status: **Implemented**. For an `object` section:

1. Find the section start.
2. Read rows as key/value pairs from the first two columns.
3. Match keys to field names.
4. Stop at the next blank row.
5. Omit blank values.
6. Return one object.

## Coercion behavior

Status: **Implemented**. Cell coercion:

- Blank cells become omitted values.
- `integer` uses finite numeric conversion and truncation.
- `number` uses finite numeric conversion.
- `boolean` uses TOTS boolean parsing.
- `date`, `datetime`, `geometry`, and `string` become strings.
- Attachment-like types remain the cell value and are later checked as paths
when possible.

## JSON Schema derivation

Status: **Implemented**. Rules:
- Top-level schema type is object.
- Each section becomes a top-level property.
- Required sections are included in top-level `required`.
- `table` sections become arrays of objects.
- `object` sections become objects.
- `comment` sections become strings.
- Fields become object properties.
- Required fields become item/object `required` entries.
- Section objects disallow additional properties.
- `date` and `datetime` map to JSON Schema string formats.
- `image` includes `contentMediaType: "image/png"`.
- `geometry`, `attachment`, and `binary` are strings in current milestones.

## Attachments

Status: **Implemented for relative path safety and existence diagnostics**.
Attachment field types:

- `image`
- `attachment`
- `binary`

Rules:
- Values must be strings.
- Paths must be relative.
- URLs are not allowed.
- Absolute filesystem paths are not allowed.
- `..` traversal is not allowed.
- Paths are resolved relative to the workbook location, archive entry, or
structured entry path.
- Missing files produce warnings, not fatal parse failures.

Diagnostic codes:
- `ATTACHMENT_PATH_INVALID`: value is not a string path.
- `ATTACHMENT_PATH_NOT_RELATIVE`: path is a URL or absolute path.
- `ATTACHMENT_PATH_UNSAFE`: path includes unsafe traversal.
- `ATTACHMENT_MISSING`: referenced attachment was not found when sibling files
were available for checking.

## Input and schema resolution

Status: **Implemented in Node; browser support uses browser-native structured
inputs and has intentional limits**. Supported parser input shapes include:
- Standalone workbook file with explicit `totsPath`.
- Workbook inside a TOTS data tree.
- Directory tree.
- Subtree with explicit TOTS path prefix.
- Zip file path.
- Zip buffer.
- Structured entries.
- Browser `File`/`Blob`-like inputs in the browser build.

Root discovery methods include:
- Explicit `rootPath`.
- Marker files such as `.tots-manifest.json` or `.tots-data-manifest.json`.
- A directory segment named `tots`.
- Provided folder root for tree parsing.
- Explicit `totsPath` for standalone workbooks.

Schema resolution can use:
- Built-in schema catalog.
- Explicit local `schemaRootPath`.
- Project `.tots-manifest.json`.
- Local or distributed `schema.catalog.json`.
- Optional built-in schema fallback with local overrides.

Resolution first checks for an exact schema leaf path, then tries
`segment-in-place` placeholder leaves. A placeholder matches only when fixed path
segments are identical and the concrete data segment satisfies the resolved
dynamic bucket regex.

## Project manifest

Project manifest file: `.tots-manifest.json`. Important fields:

- `schemaRoot`: folder containing the TOTS schema tree.
- `rootManifest`: schema-root `manifest.json`.
- `catalog`: optional master catalog path.
- `namespace`: optional mount prefix.
- `package`: optional package metadata.

## Schema catalog

Catalog file: `schema.catalog.json`. Purpose: compose multiple schema roots into
one logical tree. Source types:

- `local`
- `url`
- `git`
- `npm`
- `bundle`
- `manifest`

Source metadata:

- `id`
- `enabled`
- `namespace`
- `mountPath`
- `priority`
- `override`
- source-specific location fields

Conflict behavior:

- Lower numeric `priority` wins.
- `override: true` intentionally replaces an existing mounted leaf.
- Ambiguous same-priority conflicts fail.

## Validation and acceptance

Status: **Implemented**. Fixture acceptance flow:

1. Load schema root.
2. Validate manifest and required files.
3. Read `_defs.xlsx`.
4. Read each leaf `schema.xlsx`.
5. Derive `schema.data.json`.
6. Compare derived schema to checked-in `schema.data.json`.
7. Validate expected `example.N.json`.
8. Parse `example.N.xlsx`.
9. Compare parsed output to expected JSON.


## Current Schema Examples

The current set of schemas and examples includes:
- Field boundaries and optional grower/farm lookup sheets.
- Harvest yield/moisture map points.
- Planting population map points.
- Weather station readings.
- Crop scouting reports with attached images.
- Aerial NDVI imagery with attached images.
- Soil test metadata and results.
- Equipment and filter inventories.
- Spray application records.
- Fertilizer/manure application records.
- Irrigation logs.
- Grain storage inventory snapshots.
- Livestock feed rations.
- Pesticide inventory.
- Crop budgets.
- Tile and conservation infrastructure.

## Current limitations

Current parser and tooling intentionally defer:

- Non-`.xlsx` data parsing.
- Full high-fidelity workbook reconstruction.
- Spreadsheet formula preservation or evaluation.
- Style preservation.
- Arbitrary merged-cell layouts.
- Strict GeoJSON validation.
- Complete `comment` and `non-tabular` parsing.
- Complete `COMMENT` and `SETTINGS` typed-row parsing.
- Full `fixed-range` and transposed orientation support.
- Production API route serialization.
- Editing workflows.


