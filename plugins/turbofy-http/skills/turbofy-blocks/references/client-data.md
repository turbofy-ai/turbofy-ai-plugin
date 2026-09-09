# Client data helpers

Import these helpers from `@/api`. Supply table ids from the app schema, not table names.

| Read | Signature |
|---|---|
| One record | `useTypeQuery(tableId, recordId)` |
| List | `useListTypes(tableId, options?)` |
| Children | `useListTypesByParent(tableId, parentTableId, parentRecordId, options?)` |
| Structured query | `useQueryTypes(tableId, { where?, orderBy?, order?, limit?, offset?, dynamicArgs? })` |
| Text search | `useSearchTypes(tableId, query, fields, { limit?, dynamicArgs? }?)` |

The first three expose `data`, `isLoading`, `isFetching`, `error`, and `refetch()`. List `data` is the accumulated record array, not `{ items }`. Lists also expose `hasNextPage`, `isFetchingNextPage`, and `fetchNextPage()`. `refetch` and `fetchNextPage` return `void`; consume updates from the hook rather than awaiting a result.

List options include `limit`, `sortOrder`, and `sortRange` on the table's sort key. They do not accept arbitrary field filters. For filtering across records, use `useQueryTypes` on a table with `@fts_searchable`; filtering one fetched page only filters that page.

`useQueryTypes` and `useSearchTypes` expose `isPending`, `error`, and `data: { items, total, error? } | null`. Inspect both the hook error and `data.error`. Do not interpret a missing search index as an empty result. Use an explicit localized search field such as `title.en` when language matters.

## Mutations

```tsx
const { mutateAsync: createRecord, isPending, error } = useCreateType({
  ofType: config.tableId,
});
const record = await createRecord({ fields: { title: input } });
```

`useUpdateType({ ofType })` accepts `{ id, fields }`; `useDeleteType({ ofType })` accepts `{ id }`. Catch mutation failures and preserve user input when a write fails. Query/list hooks react to mutations made through these helpers.

For an imperative read outside React, use `getType`, `listTypes`, `listTypesByParent`, `queryTypes`, or `searchTypes`. Imperative lists return `{ items, nextToken }`; this differs from the list hooks' `data` shape. Consult the project's typings for optional parameters.

## Server updates

Flow and other server writes need a WebSocket subscription when the UI should update immediately:

```tsx
useWsSubscription(
  { ofType: config.tableId, operations: ["INSERT", "UPDATE", "DELETE"] },
  () => refetch(),
);
```

Events expose `operation` and `record`. Delivery requires an authenticated user with read access to the record. Reconcile the event into state or refetch the relevant list. The subscription alone does not rerender your component.

## Links, translations, and files

- `useLinks([{ pageId, path }])` resolves localized page links; read its `result` array in input order.
- `useTranslations` resolves data translations after a fetch. Ordinary UI strings belong in `config.copies`.
- `useFileDocuments` resolves file records after a fetch.
- `useUploadFile()` provides `mutateAsync({ file, key, accessControl?, meta? })` for a browser `File`. Use the returned file document for subsequent data writes.
