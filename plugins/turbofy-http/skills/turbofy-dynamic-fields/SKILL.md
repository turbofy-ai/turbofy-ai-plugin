---
name: turbofy-dynamic-fields
description: "Write or debug Turbofy server-side data for blocks and pages: initial records, route-dependent content, translations, and localized links. Use for defaultConfig, defaultDynamicData, config, dynamicData, or other dynamic-field JavaScript. For React UI use turbofy-blocks."
---

# Turbofy Dynamic Fields

Dynamic fields compute config and data using server-side JavaScript and the `$$std` helpers. Use TypeScript for the surrounding app declarations; dynamic-field code itself is JavaScript.

## Where to edit

- Type defaults: `defaultConfig` and `defaultDynamicData` in `block-types/<Name>/record.ts`.
- Instance overrides: `config` and `dynamicData` on `appBuilder.block(...)` in the relevant page module.
- Use the app pull/edit/push workflow in `turbofy-apps`.

Use config for stable layout settings and links. Use dynamic data for route-dependent initial content, then client hooks for subsequent interactions. UI strings belong in `localizations`; the platform supplies `config.copies` automatically.

## Evaluation contract

- `$$self` is the current record snapshot. Another dynamic field on it is still its original source, not its computed result.
- `$$args` contains request arguments such as `lang`, `params`, `slug`, and `searchParams`.
- `$$std` provides record, translation, and link helpers. Calls are synchronous.
- Return the last expression or use `return`. Return plain objects, arrays, scalars, or `null`; no functions, Dates, or browser objects.
- Syntax or runtime errors can appear as `null` data. Handle absent arguments explicitly.

Read [server data helper signatures](references/server-data.md) for fetching, filtering, pagination, translations, links, and reserved arguments. Use actual table ids from `schema.ts` or `table_list`.

## Initial record example

Inside a block type's `defaultDynamicData` string:

```js
const productId = $$std.getDynamicArg("params.product");
if (!productId) return { product: null };
const product = $$std.getRecord("<product-table-id>", productId, {
  dynamicArgs: $$args,
});
({ product });
```

Use the route parameter name declared by the app. Pass `dynamicArgs: $$args` when fetching records whose nested dynamic fields need the same language or route context.

## Lists and filtering

`listRecords` returns one page; pass its returned `nextToken` as `cursor` to continue. Filtering that array only filters the fetched page. For complete server-side filtering/sorting use `queryRecords` on a table marked `@fts_searchable`, and handle its `error` result.

```js
const result = $$std.listRecords("<product-table-id>", {
  limit: 10,
  cursor: $$std.getDynamicArg("cursor"),
}, true);
({ products: result.items, nextToken: result.nextToken });
```

## Debugging

For unexpected `null`, verify the table id, argument names, and return shape. Temporarily return a constant to isolate evaluation errors, then restore the intended code. If nested evaluation is the problem, use `skipDynamicResolver` only when raw source is the desired input. Keep requested fields available when using `dynamicArgs.fields`.

The React block must handle the initial `undefined` loading state and distinguish it from loaded-but-empty data. See `turbofy-blocks`.
