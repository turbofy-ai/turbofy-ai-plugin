---
name: turbofy-dynamic-fields
description: "Use when server-side data logic in a Turbofy app is wrong or needs to be written — pre-fetching records, resolving page-specific titles from the URL, wiring translations, resolving links between pages, or debugging why a section shows null/empty data. Triggers: 'load data for this section', 'fetch products server-side', 'show the right title for this URL', 'why is this block empty', 'resolve links between pages', 'get translations for this content', or when editing @dynamic_field / $$std code. Covers the Secure VM runtime, $$self/$$args/$$std, and common data-fetch patterns. For the React UI that displays the data, load turbofy-blocks. For page/section placement, load turbofy-apps."
disable-model-invocation: false
---

# Turbofy Dynamic Fields

Dynamic fields are string fields whose contents are executed as JavaScript in a server-side Secure VM. They are how a Turbofy app computes derived values, fetches related records, resolves localized copies, and resolves page links — all without a deployed backend. This skill covers the runtime model, the `$$std` API, and common patterns.

The companion skills `turbofy-apps` (app workflow + data model) and `turbofy-blocks` (writing block React components) cover those areas.

## When to load this skill

Load when **server-side data is wrong or missing** — e.g. "why is this section empty?", "load featured products on the homepage", "show the article title from the URL". Users rarely say "dynamic field"; they describe the symptom or outcome.

---

## Runtime model

- Return value:
  - If the last statement is an expression, that expression is returned.
  - `return ...` works normally.
  - If nothing is returned, result is `undefined`.
- Errors:
  - Syntax/runtime errors yield `null` for that field (without detailed error info).

Globals:

- `$$self`: record snapshot at start of evaluation
- `$$args`: dynamic arguments (request-scoped)
- `$$std`: standard library helpers

**Snapshot gotcha:** If one dynamic field reads another dynamic field on the same record via `$$self.otherField`, it sees the original string, not the computed value.

---

## `$$std` API reference

### Record/args helpers

- `$$std.getCurrentRecord()` — legacy alias for `$$self`
- `$$std.getDynamicArg(key, defaultValue?)` — reads from `$$args` using lodash-style paths (e.g. `"config.lang"`)
- `$$std.get` — re-export of `lodash/get`

### `$$std.getRecord(tableId, recordId, options?)`

Fetches a single record. Returns the record object, or `null` when fetch fails.

| Option        | Type      | Description                                                                                                             |
| ------------- | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| `dynamicArgs` | `object`  | Passed through to nested dynamic field evaluation. **Always pass `$$args`** when the fetched record has dynamic fields. |
| `normalize`   | `boolean` | Default `true`. Set `false` to return the raw provider value.                                                           |
| `limit`       | `number`  | Forwarded to the provider (primarily used by list operations).                                                          |

```js
const product = $$std.getRecord("tbl_abc", productId, { dynamicArgs: $$args });
```

### `$$std.listRecords(tableId, options?, withToken?)`

Fetches a page of records from a collection. **Does not accept `filter`** — filter client-side on the returned items.

| Option        | Type            | Description                                                                                                          |
| ------------- | --------------- | -------------------------------------------------------------------------------------------------------------------- |
| `cursor`      | `string`        | Pagination cursor (from a previous `nextToken`).                                                                     |
| `limit`       | `number`        | Max records to return.                                                                                               |
| `dynamicArgs` | `object`        | Passed through to nested evaluation.                                                                                 |
| `normalize`   | `boolean`       | Default `true`.                                                                                                      |
| `sortRange`   | `object`        | Range/filter condition on the sort key (defaults to `updatedAt` or the field marked `@sortby`). See operators below. |
| `sortOrder`   | `DESC` or `ASC` | Sort direction. Defaults to `DESC` if omitted.                                                                       |

**Sort range operators:**

| Operator     | Shape                                   | Meaning               |
| ------------ | --------------------------------------- | --------------------- |
| `eq`         | `{ eq: "value" }`                       | Equal                 |
| `beginsWith` | `{ beginsWith: "prefix" }`              | Starts with           |
| `lt`         | `{ lt: "value" }`                       | Less than             |
| `le`         | `{ le: "value" }`                       | Less than or equal    |
| `gt`         | `{ gt: "value" }`                       | Greater than          |
| `ge`         | `{ ge: "value" }`                       | Greater than or equal |
| `between`    | `{ between: ["fromValue", "toValue"] }` | Inclusive range       |

```js
// Records updated today
const today = new Date().toISOString().slice(0, 10);
const recent = $$std.listRecords("tbl_abc", {
  sortRange: { ge: today },
  sortOrder: "DESC",
  limit: 20,
});
```

Return shape depends on `withToken`:

- **default** (`withToken` omitted or `false`): returns the `items` array directly
- **`withToken === true`**: returns `{ items, nextToken }`

```js
// Simple — returns array
const products = $$std.listRecords("tbl_abc");

// With pagination — returns { items, nextToken }
const result = $$std.listRecords(
  "tbl_abc",
  { limit: 10, cursor: nextToken },
  true,
);
result.items; // array
result.nextToken; // string | undefined
```

### `$$std.listRecordsByParent(tableId, parentTableId, parentRecordId, options?, withToken?)`

Fetches a page of records filtered by parent relationship.

- **options**: same as `listRecords` (`cursor`, `limit`, `dynamicArgs`, `normalize`)
- **return shape**: same as `listRecords`

```js
const blocks = $$std.listRecordsByParent("tbl_block", "tbl_page", pageId);
```

### `$$std.batchGetRecords(tableId, recordIds, options?)`

Batch-fetches multiple records from a single table.

- **returns**: `{ items, unprocessedKeys }`

```js
const { items } = $$std.batchGetRecords("tbl_abc", ["id1", "id2", "id3"]);
```

### `$$std.batchGetRecordsByInputs(inputs)`

Batch-fetch multiple records in one go, where each entry can target a different table and have different `dynamicArgs`.

- **inputs**: `Array<{ ofType: string; id: string; options?: { dynamicArgs?: object; normalize?: boolean } }>`
- **returns**: `{ items, unprocessedKeys }`
  - `items`: `Array<Record<string, unknown> | null>` aligned to `inputs` order (duplicates preserved)

Use this as an optimization tool when you would otherwise call `$$std.getRecord(...)` many times.

### `$$std.translate(localizationPartialKey, copyPath?)`

Resolves a localized value from a Localization record.

1. Looks up a Localization record with id `${lang}_${localizationPartialKey}`
2. Reads its `dictionary` field
3. If `copyPath` is provided, returns `$$std.get(dictionary, copyPath)`; otherwise returns the full dictionary

```js
// Full dictionary
const copies = $$std.translate("blocktype_" + $$self.id);

// Single value via path
const title = $$std.translate("home-page", "title");
```

Returns `null` if the localization record or key doesn't exist.

### `$$std.batchTranslate(entries, copyPath?)`

Resolves many localization values in one go.

- **entries**: `Array<string | { key?: string; localizationPartialKey?: string; copyPath?: string }>`
  - string entries are treated as the localization partial key
  - `copyPath` can be provided per-entry or as the second argument
- **returns**: `Array<unknown | null>` aligned to `entries` order
  - missing localization records resolve to `null`

```js
const titles = $$std.batchTranslate(["home-page", "footer"], "title");
```

### `$$std.getImage(imageId)`

Fetches an Image record. Returns the record object (`{ url, height, width }`) or `null`.

```js
const image = $$std.getImage($$self.heroImageId);
// image?.url, image?.height, image?.width
```

### `$$std.batchLink(entries)`

Resolves many page links in one call by returning each target page's resolved `localizedConfig.canonicalPath`.

- **entries**: `Array<string | { pageId: string; path?: string; dynamicArgs?: object }>`
  - string entries are treated as `{ pageId: entry }`
  - `path` is converted to `slug` segments under the hood (used by the page's `localizedConfig` resolver)
- **returns**: `Array<string | null>` aligned to `entries` order

Requirements:

- `$$args.lang` must be set
- `cmsOfTypes.page` must be configured (use the system CMS type id from `CmsOfTypeEnum.Page`)

---

## Reserved `dynamicArgs` keys

| Key                   | Type                               | Description                                                                                                                                                                                                                               |
| --------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fields`              | `string[]`                         | Projection: evaluate only the listed dynamic fields. Missing or `[]` evaluates all. If your code uses `$$self.someField`, ensure `fields` includes it.                                                                                    |
| `fieldsCode`          | `Record<string, string>`           | Per-field code override (highest priority over record value and `defaultCode`).                                                                                                                                                           |
| `skipDynamicResolver` | `boolean`                          | When `true`, disables recursive dynamic-field evaluation for fetched records (useful for debugging or performance).                                                                                                                       |
| `lang`                | `string`                           | Used by CMS helpers (`translate`, `getImage`).                                                                                                                                                                                            |
| `cmsOfTypes`          | `{ page?, localization?, image? }` | Type IDs for CMS helpers. For system CMS tables, use `CmsOfTypeEnum.Page` / `CmsOfTypeEnum.Localization` / `CmsOfTypeEnum.Image` (stable IDs).                                                                                            |
| `slug`                | `string[]`                         | URL path segments (after the lang prefix). Used by page routing logic in `dynamicData`.                                                                                                                                                  |
| `params`              | `Record<string, string>`           | Resolved entity IDs for dynamic slug segments. Keys match the param names from `paramsCollectionMap` (e.g. `{ articleId: "1234-..." }`). Available in `dynamicData`. Use `$$args.params.articleId` to access.                           |
| `searchParams`        | `Record<string, string>`           | Current URL query parameters. Available in `dynamicData` for route/query-dependent server-side data.                                                                                                                                      |

---

## Dynamic field patterns

**A) Read dynamic args safely**

```js
const lang = $$std.getDynamicArg("lang", "en");
const recordId = $$std.getDynamicArg("recordId");
if (!recordId) return null;
```

**B) Translation lookups**

```js
const copies = $$std.translate("blocktype_" + $$self.id);
({ copies });
```

**C) Fetch a single record**

```js
const product = $$std.getRecord("tbl_product", productId, {
  dynamicArgs: $$args,
});
({ product });
```

**D) Fetch a list with pagination**

```js
const result = $$std.listRecords("tbl_product", { limit: 10 }, true);
({ products: result.items, nextToken: result.nextToken });
```

**E) Client-side filtering (no server-side filter)**

```js
const allProducts = $$std.listRecords("tbl_product", { limit: 100 });
const featured = allProducts.filter((p) => p.featured === true);
({ featured });
```

**F) Prefer serializable shapes** — Return plain objects/arrays/strings/numbers/booleans/null. Avoid functions, Dates, class instances, DOM APIs (not available), cyclic refs.

---

## Debugging checklist

- If the field returns `null` unexpectedly:
  - Check syntax (missing commas, stray trailing characters).
  - Guard missing args (`recordId`, `slug`, etc).
  - Temporarily simplify to a constant object and add pieces back.
- If nested records behave oddly:
  - Try `skipDynamicResolver: true` in dynamicArgs for nested fetches.
- If `$$self.someField` is `undefined`:
  - Ensure the query included it via `dynamicArgs.fields`.

---

## See also

- **`turbofy-platform`** — platform orientation, workspaces & environments, the full MCP tool surface and core rules, the schema workflow, and the data-builder DSL (useful when a dynamic field references a workspace table by `SomeTable.id`).
- **`turbofy-apps`** — Apps CMS data model (where dynamic fields appear on `Page.localizedConfig`, `BuildingBlockType.defaultConfig`/`defaultDynamicData`, `BuildingBlock.config`/`dynamicData`), the `Turbofy_app_*` workflow, the auto-injected `copies` mechanism.
- **`turbofy-blocks`** — when and how to push state into `dynamicData` versus client-side `@/api` hooks; how `config.copies` is consumed in the React component; `$$std.batchLink` usage examples.
