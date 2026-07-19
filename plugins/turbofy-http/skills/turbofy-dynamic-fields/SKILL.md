---
name: turbofy-dynamic-fields
description: "Use when server-side data logic in a Turbofy app is wrong or needs to be written — pre-fetching records, resolving page-specific titles from the URL, wiring translations, resolving links between pages, or debugging why a section shows null/empty data. Triggers: 'load data for this section', 'fetch products server-side', 'show the right title for this URL', 'why is this block empty', 'resolve links between pages', 'get translations for this content', or when editing @dynamic_field / $$std code. Covers the Secure VM runtime, $$self/$$args/$$std, and common data-fetch patterns. For the React UI that displays the data, load turbofy-blocks. For page/section placement, load turbofy-apps."
disable-model-invocation: false
---

# Turbofy Dynamic Fields

Dynamic fields are string (or configured) fields whose contents run as JavaScript in a server-side Secure VM. They compute derived values, fetch related records, resolve localized copies, and resolve page links — without a deployed backend.

Companions: `turbofy-apps` (where these fields live on CMS entities), `turbofy-blocks` (React consumption).

**Writing `defaultConfig` / `defaultDynamicData` through MCP:** put the JS on the app manifest (`blockTypes[]` / block instances) and apply with `app_push` (`turbofy-apps`). Prefer unwrapped user JS; the platform applies copies guards.

## When to load this skill

Load when **server-side data is wrong or missing** — e.g. "why is this section empty?", "load featured products on the homepage".

---

## Runtime model

- Return value: last expression, or `return …`. Otherwise `undefined`.
- Errors: syntax/runtime → `null` for that field (little detail).

Globals:

- `$$self` — record snapshot at start of evaluation
- `$$args` — request-scoped dynamic arguments
- `$$std` — standard library

**Snapshot gotcha:** reading another dynamic field on the same record via `$$self.otherField` sees the original string, not the computed value.

---

## `$$std` API reference

### Record/args helpers

- `$$std.getCurrentRecord()` — legacy alias for `$$self`
- `$$std.getDynamicArg(key, defaultValue?)` — lodash-style paths into `$$args`
- `$$std.get` — `lodash/get`

### `$$std.getRecord(tableId, recordId, options?)`

Returns the record or `null`.

| Option | Description |
|---|---|
| `dynamicArgs` | Pass through for nested dynamic fields — **always pass `$$args`** when needed |
| `normalize` | Default `true` |
| `limit` | Forwarded to provider |

```js
const product = $$std.getRecord("tbl_abc", productId, { dynamicArgs: $$args });
```

### `$$std.listRecords(tableId, options?, withToken?)`

No server-side `filter` — filter client-side on returned items.

Options: `cursor`, `limit`, `dynamicArgs`, `normalize`, `sortRange`, `sortOrder` (`ASC`|`DESC`).

**Sort range operators:** `eq`, `beginsWith`, `lt`, `le`, `gt`, `ge`, `between: [from, to]`.

- Default: returns `items` array
- `withToken === true`: `{ items, nextToken }`

```js
const recent = $$std.listRecords("tbl_abc", {
  sortRange: { ge: new Date().toISOString().slice(0, 10) },
  sortOrder: "DESC",
  limit: 20,
});
```

### `$$std.listRecordsByParent(tableId, parentTableId, parentRecordId, options?, withToken?)`

Same options/return shape as `listRecords`, scoped by parent.

### `$$std.batchGetRecords(tableId, recordIds, options?)`

→ `{ items, unprocessedKeys }`

### `$$std.batchGetRecordsByInputs(inputs)`

`inputs`: `Array<{ ofType, id, options? }>` → `{ items, unprocessedKeys }` aligned to inputs (duplicates preserved). Prefer this over many `getRecord` calls.

### `$$std.translate(localizationPartialKey, copyPath?)`

Looks up Localization id `${lang}_${localizationPartialKey}`, reads `dictionary`, optional path.

```js
const copies = $$std.translate("blocktype_" + $$self.id);
const title = $$std.translate("home-page", "title");
```

### `$$std.batchTranslate(entries, copyPath?)`

Many translations in one call → array aligned to `entries`.

### `$$std.getImage(imageId)`

→ `{ url, height, width }` or `null`.

### `$$std.batchLink(entries)`

Resolves page `localizedConfig.canonicalPath` values.

- `entries`: `Array<string | { pageId, path?, dynamicArgs? }>`
- Requires `$$args.lang`
- Returns `Array<string | null>`

Use system page ofType `"cmspage"` in surrounding CMS config; pass real page record ids as `pageId`.

---

## Reserved `dynamicArgs` keys

| Key | Description |
|---|---|
| `fields` | Evaluate only listed dynamic fields |
| `fieldsCode` | Per-field code override |
| `skipDynamicResolver` | Skip nested dynamic evaluation |
| `lang` | Language for CMS helpers |
| `cmsOfTypes` | `{ page?, localization?, image? }` type ids — use `"cmspage"`, `"cmslocalization"`, `"filedocument"` (or image id used by the workspace) |
| `slug` | Path segments after lang |
| `params` | Resolved entity ids from `paramsCollectionMap` |
| `searchParams` | Query string map |

---

## Patterns

**A) Safe args**

```js
const lang = $$std.getDynamicArg("lang", "en");
const recordId = $$std.getDynamicArg("recordId");
if (!recordId) return null;
```

**B) Translations**

```js
({ copies: $$std.translate("blocktype_" + $$self.id) });
```

(Platform wraps block-type `defaultConfig` to inject `copies` itself — user code usually returns non-copy config only. See `turbofy-apps`.)

**C) Single record**

```js
const product = $$std.getRecord("TABLE_ID", productId, { dynamicArgs: $$args });
({ product });
```

**D) List + pagination**

```js
const result = $$std.listRecords("TABLE_ID", { limit: 10 }, true);
({ products: result.items, nextToken: result.nextToken });
```

**E) Client-side filter**

```js
const all = $$std.listRecords("TABLE_ID", { limit: 100 });
({ featured: all.filter((p) => p.featured === true) });
```

**F) Serializable only** — plain objects/arrays/scalars/null. No functions, Dates, DOM, cycles.

Resolve **table ids** via `workspace_get` → `schema.types[].id` — never table names.

---

## Debugging checklist

- Unexpected `null`: syntax, missing args, simplify to a constant then rebuild.
- Nested weirdness: try `skipDynamicResolver: true`.
- `$$self.someField` undefined: include it in `dynamicArgs.fields`.

---

## See also

- **`turbofy-platform`** — schema JSON (`workspace_get`), CMS ofTypes.
- **`turbofy-apps`** — where dynamic fields hang on the app manifest; `app_push`; auto-injected `copies`.
- **`turbofy-blocks`** — `config.copies` / `dynamicData` vs client hooks.
