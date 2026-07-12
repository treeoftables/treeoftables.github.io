# TOTS schema root

`schema/tots` is this repository's canonical TOTS schema root. The main language
documentation now lives at the monorepo root:

- `../../LANGUAGE_quickstart.md` - short introduction for readers and schema

authors

- `../../LANGUAGE_learning.md` - educational guide to tree schemas, dynamic

buckets, sections, fields, typed rows, and attachments

- `../../LANGUAGE_reference.md` - exhaustive language reference with

implementation status notes

This file stays focused on the local schema-root package: what files live here,
how fixtures are organized, and how this root is discovered by tools.

## Schema root files

Important files in this directory:

- `_defs.xlsx` - language-level dynamic bucket and typed-row definitions
- `schema.meta.json` - global default JSON Schema for parsed `meta`
- `schema.tots.json` - global default JSON Schema for parsed `tots`
- `manifest.json` - generated index of fixture leaves and example files
- `schema.catalog.json` - example master catalog for local and distributed

schema roots

- `schema.catalog.schema.json` - JSON Schema for catalog files
- `schema.root-manifest.schema.json` - JSON Schema for project-level

`.tots-manifest.json` files

## Leaf package structure

A schema-tree leaf is a directory whose name ends in `.schema`. Each leaf
usually contains:

- `schema.xlsx` - workbook schema for the leaf
- `schema.data.json` - expected JSON Schema for parsed `data`
- `example.N.xlsx` - example spreadsheet input
- `example.N.json` - expected parser output for the example
- optional `attachments/` - binary files referenced by examples
- optional `schema.meta.json` - leaf-level extension or override for `meta`
- optional `schema.tots.json` - leaf-level extension or override for `tots`
- optional `README.md` - human notes for the leaf

The `.schema` directory represents an abstract TOTS leaf resource. Actual API or
storage names such as `.xlsx`, `.csv`, `.csvf`, or `.csvf.zip` are caller and
tooling concerns.

## Discovery files

The repository root has `.tots-manifest.json`, which points tools at this schema
root. External projects can use the same pattern:

- `.tots-manifest.json` points to a project schema root.
- `manifest.json` inside the schema root indexes leaves and examples.
- `schema.catalog.json` composes multiple schema roots.
- `package.json.tots` metadata can point package consumers to a schema root or

manifest.

Catalog entries can describe local folders, URLs, Git repositories, npm
packages, bundles, or direct project manifests. Runtime tools mount enabled
sources into one logical schema tree and preserve provenance in parsed results.

## Fixture suite

The generated agriculture fixture suite is intentionally broad enough to guide
parser implementation. Current themes include:

- field inventory with optional grower/farm lookup sheets
- harvest and planting map points
- weather station readings
- crop scouting reports with attached PNG photos
- aerial NDVI imagery with attached PNG images
- soil test metadata and nutrient results
- equipment filter inventories
- spray, manure, irrigation, storage, ration, budget, pesticide, and tile-map

records

The language docs explain how these examples demonstrate tree paths, dynamic
buckets, workbook sections, typed rows, field types, and attachments.

## Binary attachment examples

Two fixture families demonstrate relative image references:

- `ag/crop-scouting/year-index/2026/field-index/north80.schema`
- `ag/aerial-imagery/year-index/2026/field-index/north80/ndvi.schema`

Image fields store workbook-relative paths such as
`attachments/gray-leaf-spot-closeup.png`. The parser reports missing or unsafe
attachment paths in `tots.diagnostics` while still returning parsed data.

Several fixtures also demonstrate `segment-in-place` dynamic indexes. For
example, schema leaf
`ag/planting/as-planted/year-index/2026/field-name-index-inplace.schema` matches
data path `ag/planting/as-planted/year-index/2026/north80.xlsx`; the placeholder
does not appear literally in user data paths.

## Common commands

Run these from the monorepo root:

- `yarn milestone1:fixtures` - regenerate fixture workbooks and JSON files
- `yarn milestone1:validate` - validate fixture structure and JSON examples
- `yarn milestone1:acceptance` - derive fixture data schemas and parse/compare

all example workbooks

- `yarn milestone1:tree` - inspect the generated schema tree
- `yarn tots-cli tree schema/tots` - print the TOTS-aware schema tree
- `yarn tots-cli inspect schema/tots` - inspect this schema root as structured

JSON

- `yarn viz schema/tots` - open the read-only visual inspector

## Validation role

Each `.schema/` leaf is also an acceptance-test package:

1. Read `schema.xlsx`.
2. Derive `schema.data.json`.
3. Compose global or leaf-level `schema.meta.json` and `schema.tots.json`.
4. Validate `example.N.json`.
5. Parse `example.N.xlsx`.
6. Compare parser output to expected JSON.

This keeps language documentation, schemas, examples, and parser behavior
aligned.
