# @treeoftables/lib

Consumer-facing Tree of Tables parser runtime.

Milestone 4 made the public parser surface async-only and started the isomorphic runtime split. Milestone 5 added distributed schema adapters and built schema artifacts. Milestone 6 adds structured inspection output for CLI and UI consumers. Milestone 8’s read-only `@treeoftables/viz` inspector consumes these parser and inspection APIs rather than duplicating resolver logic. The Node build in `dist/` and browser build in `dist-browser/` expose matching high-level parser API names, use the built-in schema catalog bundled from this monorepo's `schema/tots` tree by default, and share structural file-like input types at the public type boundary.

The Node build supports `.xlsx` inputs from filesystem paths, buffers, zip archives, directories, local custom schema roots, project manifests, local catalogs, Git schema sources, npm package roots, URL manifests, and bundle archives. The browser build supports browser-native `File`/`Blob`-like objects, file handles, directory/file-list style inputs, structured entries, and workbook buffers. Browser filesystem paths, project manifests, local catalog path resolution, and schema source inspection intentionally throw `UNSUPPORTED_PLATFORM`.

## Build and test

- `yarn milestone2:build`
- `yarn milestone2:test`
- `yarn milestone2:check`
- `yarn milestone4:browser`
- `yarn milestone5:distributed`
- `yarn milestone6:inspect`
- `yarn milestone8:viz`

`yarn milestone4:browser` runs the automated headless Chromium coverage for the browser build, including workbook buffers, browser `File` inputs, file-list-like virtual trees, structured entries, and wrong-platform/deferred zip errors.

`yarn milestone5:distributed` runs distributed adapter, cache, conflict, built-artifact, and inspection API tests. `yarn milestone6:inspect` runs the inspection-focused library and CLI checks.

`yarn milestone7:attachments` runs parser coverage for relative PNG image attachments and missing-file diagnostics.

`yarn milestone8:viz` runs the read-only inspector coverage that consumes `@treeoftables/lib` inspection, parser, and attachment diagnostics.

## Parse one standalone workbook

Standalone workbooks need an explicit TOTS path so the parser can resolve the matching built-in schema leaf.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/tmp/field_boundaries.xlsx", {
  totsPath: "ag/fields/field_boundaries.xlsx",
});

console.log(parsed.data);
```

## Parse one workbook inside a TOTS data tree

If a workbook lives below a `.tots-manifest.json` or `.tots-data-manifest.json` marker, the parser infers the TOTS path from the file path. If no manifest marker exists, a directory segment named `tots` is treated as the top of the TOTS data tree.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/data/ag/fields/field_boundaries.xlsx");
```

Schema resolution supports both exact schema leaf paths and `segment-in-place`
placeholders. For example, data path
`ag/planting/as-planted/year-index/2026/north80.xlsx` resolves to schema leaf
`ag/planting/as-planted/year-index/2026/field-name-index-inplace.schema` and
reports `{ "year-index": "2026", "field-name-index": "north80" }` in
`parsed.tots.pathContext`.

## Parse a folder or subtree

Folder parsing returns a collection.

```js
import { parseTotsTree } from "@treeoftables/lib";

const collection = await parseTotsTree("/data/ag/fields", {
  totsPath: "ag/fields",
});

for (const item of collection.items) {
  console.log(item.tots.leafPath, item.data);
}
```

## Parse a zip archive

Zip archives are treated like TOTS data trees. If the archive contains a root marker, paths are resolved relative to that marker; otherwise zip-root-relative paths are used.

```js
import { parseTots } from "@treeoftables/lib";

const collection = await parseTots("/tmp/tots-data.zip");
```

Zip buffers are also supported.

```js
import { parseTots } from "@treeoftables/lib";

const collection = await parseTots({
  buffer: await fs.promises.readFile("/tmp/tots-data.zip"),
  filename: "tots-data.zip",
});
```

## Handle binary attachments

Schema fields with type `image`, `attachment`, or `binary` are parsed as string relative paths. For images, PNG is the recommended default. The parser validates attachment paths when it can inspect sibling files from a filesystem workbook, parsed directory, zip archive, or structured-entry set.

Missing image files do not throw by default. They appear as warning diagnostics on the parsed item and on collections:

```js
import { parseTotsTree } from "@treeoftables/lib";

const collection = await parseTotsTree("/data/tots");
for (const diagnostic of collection.diagnostics) {
  if (diagnostic.code === "ATTACHMENT_MISSING") {
    console.warn(diagnostic.attachmentPath, diagnostic.message);
  }
}
```

Attachment paths must be relative to the workbook location in the same TOTS data tree. Absolute paths, URL-like paths, and `..` traversal are reported as non-portable or unsafe diagnostics. Buffer-only parses cannot verify sibling files unless the caller provides a zip or structured entries.

## Parse in a browser

Normal browser-aware bundlers can import `@treeoftables/lib` and use the package `browser` export condition. Advanced callers or smoke tests can import `@treeoftables/lib/browser` explicitly.

Standalone browser `File` or buffer inputs need an explicit TOTS path unless the file has a relative path inside a `tots/` folder.

```js
import { parseTots } from "@treeoftables/lib";

const parsed = await parseTots({
  file,
  totsPath: "ag/fields/field_boundaries.xlsx",
});
```

Structured entries let tests or upload flows parse a virtual tree without using native picker dialogs.

```js
import { parseTotsEntries } from "@treeoftables/lib/browser";

const collection = await parseTotsEntries([
  {
    path: "ag/fields/field_boundaries.xlsx",
    arrayBuffer: () => file.arrayBuffer(),
  },
]);
```

## Resolve context without parsing

Use `resolveTotsContext` to inspect classification, root discovery, inferred TOTS path, and schema resolution.

```js
import { resolveTotsContext } from "@treeoftables/lib";

const context = await resolveTotsContext("/data/ag/fields/field_boundaries.xlsx");
console.log(context);
```

## Use a local custom schema root

By default, parser calls use the bundled schema catalog. To replace it with a local schema root, pass `schemaRootPath`.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/tmp/field_boundaries.xlsx", {
  totsPath: "ag/fields/field_boundaries.xlsx",
  schemaRootPath: "/path/to/project/schema/tots",
});
```

To use a project `.tots-manifest.json`, pass `projectRootPath` or `projectManifestPath`.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/tmp/field_boundaries.xlsx", {
  totsPath: "ag/fields/field_boundaries.xlsx",
  projectRootPath: "/path/to/project",
});
```

To compose enabled local sources from a `schema.catalog.json`, pass `catalogPath`.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/tmp/field_boundaries.xlsx", {
  totsPath: "ag/fields/field_boundaries.xlsx",
  catalogPath: "/path/to/project/schema/tots/schema.catalog.json",
});
```

To augment the bundled catalog with a local schema root, pass `includeBuiltinSchema: true`. Local schemas use higher precedence by default, so a local leaf can replace a bundled leaf while other bundled leaves remain available.

```js
import { parseTotsFile } from "@treeoftables/lib";

const parsed = await parseTotsFile("/tmp/field_boundaries.xlsx", {
  totsPath: "ag/fields/field_boundaries.xlsx",
  schemaRootPath: "/path/to/project/schema/tots",
  includeBuiltinSchema: true,
});
```

Local catalog composition supports enabled `local` and local-file `manifest` sources. Priority is deterministic: lower numeric `priority` wins, `override: true` intentionally replaces an existing mounted leaf, and ambiguous same-priority conflicts fail as configuration errors.

## Use distributed schema sources

Node catalog resolution supports enabled `local`, `manifest`, `url`, `git`, `npm`, and `bundle` sources. Sources mount into one logical schema tree using `namespace` or `mountPath`, lower numeric `priority` wins, `override: true` intentionally replaces an existing mounted leaf, and ambiguous same-priority conflicts fail.

```js
import { resolveCatalog } from "@treeoftables/lib";

const catalog = await resolveCatalog("/path/to/schema/tots/schema.catalog.json");
console.log(catalog.leaves.length, catalog.sources);
```

Git, npm, bundle, URL, and remote manifest sources are materialized into the local schema cache. Set `TOTS_CACHE_DIR` to isolate or override cache location in tests or CI.

## Build schema artifacts

The `tots` CLI builds optional distributable schema artifacts, and `@treeoftables/lib` exposes the same helper for advanced callers.

```js
import { buildSchemaArtifact } from "@treeoftables/lib";

await buildSchemaArtifact({
  schemaRootPath: "/path/to/schema/tots",
  outDir: "/tmp/tots-built",
});
```

Artifacts include `artifact.json`, pre-parsed defs, normalized schema metadata, derived data schemas, and source-compatible leaf packages. Source trees remain first-class; built artifacts are an optimization and inspection aid.

## Inspect schema sources

Use `inspectTotsSource` to get structured resolver/catalog diagnostics for CLIs and future UI tooling.

```js
import { inspectTotsSource } from "@treeoftables/lib";

const inspected = await inspectTotsSource("/path/to/schema/tots");
console.log(inspected.schemaRoot.leafCount);
console.log(inspected.catalog.tree);
```

Catalog files can be inspected directly:

```js
const inspected = await inspectTotsSource("/path/to/schema.catalog.json", {
  mode: "catalog",
});
```

## Fixture acceptance through `lib`

The repository fixture acceptance scripts now call `@treeoftables/lib`. Advanced callers can also import the helpers directly.

```js
import { runTotsAcceptance, validateTotsFixtures } from "@treeoftables/lib";

await validateTotsFixtures("/path/to/project");
const result = await runTotsAcceptance("/path/to/project");
console.log(result.exampleCount);
```

## Current limits

- `.xlsx` is the only parsed workbook/data serialization.
- Attachment checks validate relative path safety and existence only; image decoding and content validation are deferred.
- Browser catalog/distributed source resolution is intentionally unsupported; pass pre-resolved catalogs or structured entries to browser code.
- Browser zip trees are supported via `parseTots(zipFile)` / `parseTots({ buffer, filename })`. Use `isTots(file)` to classify a File, zip, or workbook without throwing.
- `serializeTotsWorkbook` and `serializeTotsTree` write parser-compatible xlsx/zip trees (no styles or formulas).
- High-fidelity workbook reconstruction, formula/style preservation, and editing workflows are deferred.
