# Server data helpers

Use these helpers in dynamic-field JavaScript. Calls are synchronous; do not add `await`. Return plain serializable values.


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

```js
const product = $$std.getRecord("tbl_abc", productId, { dynamicArgs: $$args });
```

### `$$std.listRecords(tableId, options?, withToken?)`

No server-side `filter` — filter the returned page in JavaScript, or use
`$$std.queryRecords` when the table is searchable.

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

Scoped by parent. Same return shape as `listRecords`; options are `cursor`, `limit`, `dynamicArgs`, `normalize`, `sortOrder`. **`sortRange` is silently ignored here** (only `listRecords` implements it) — filter/sort the returned page instead.

### `$$std.queryRecords(tableId, input?)`

Server-side filtering, sorting, and pagination over the workspace's query
index. Works only on tables carrying the `@fts_searchable` directive; on a
table or stack without the index the result carries an `error` marker
instead of throwing. Check it and show an error or use an explicitly designed fallback; an unfiltered list is not equivalent to a filtered query.

→ `{ items, total, error? }`

`input`: `where`, `orderBy`, `order` (`ASC`|`DESC`), `limit`, `offset`.
`where` maps field → `{ operator: value }` with operators `eq`, `ne`, `lt`,
`le`, `gt`, `ge`, `contains`, `beginsWith`, `in`, `isNull`. Queryable
fields: any scalar column, dotted Json key paths (`attributes.voltage`)
when the field's search config sets `indexKeys`, and per-locale localized
paths (`title.en`).

```js
const res = $$std.queryRecords("tbl_abc", {
  where: { "attributes.voltage": { ge: 100 }, inStock: { eq: true } },
  orderBy: "price",
  order: "ASC",
  limit: 10,
});
if (res.error) return null;
({ products: res.items, total: res.total });
```

### `$$std.searchRecords(tableId, query, fields, options?)`

Full-text search over the same index: prefix terms, relevance-ranked.
`fields` scopes the match — text columns match their words, Json fields
their string leaves. For a localized string field, its plain name searches
the field's **first declared locale**; pass an explicit `name.<locale>`
(e.g. `` `title.${$$args.lang}` ``) to search another language — a search
never matches across languages. Same `@fts_searchable` requirement and
`error` marker as `queryRecords`. `options`: `{ limit }`.

→ `{ items, total, error? }`

```js
const lang = $$std.getDynamicArg("lang", "en");
const res = $$std.searchRecords("tbl_abc", $$args.searchParams.q, [
  `title.${lang}`,
  "attributes",
]);
if (res.error) return null;
({ results: res.items, total: res.total });
```

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

`entries`: `Array<string | { key?: string; localizationPartialKey?: string; copyPath?: string }>`. Returns an array aligned to `entries`; missing translations are `null`.

### `$$std.getImage(imageId)`

→ the full image record with `width` / `height` merged in from its metadata (so `url`, `width`, `height` plus the record's other fields), or `null`.

### `$$std.batchLink(entries)`

Resolves localized page URLs.

- `entries`: `Array<string | { pageId, path?, dynamicArgs? }>`
- Requires `$$args.lang`
- Returns `Array<string | null>`

Use system page ofType `"cmspage"` in surrounding CMS config; pass real page record ids as `pageId`.

A single-page variant `$$std.link(pageId, params?)` also exists; prefer `batchLink` when resolving several links.

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
