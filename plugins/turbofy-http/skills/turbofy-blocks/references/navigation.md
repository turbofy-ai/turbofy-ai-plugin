# Navigation hooks

Import `useParams`, `useSearchParams`, `Link`, and `navigate` from `@/navigation`. Use these helpers in both workspace preview and published apps.

## `useParams()` — resolved route record IDs

```tsx
const { params, isLoading, notFound, error } = useParams();
```

The return type is `IRouteParamsState`, also exported from `@/navigation`:

```ts
interface IRouteParamsState {
  params: Record<string, string> | undefined;
  isLoading: boolean;
  notFound: boolean;
  error: Error | null;
}
```

Parameter names come from the page route configuration. For a `[product]` route mapped to a product table's slug field, `params.product` is the resolved record ID, not the slug text. The hook resolves IDs, not record contents; fetch the record separately through `@/api`.

| State | What to do |
|---|---|
| `isLoading: true`, `params: undefined` | Show loading UI; defer record-dependent queries. |
| Resolution succeeded | Read IDs from `params`; a route without parameters returns `{}`. |
| `notFound: true`, `params: undefined` | Show an unavailable state or navigate to a localized fallback. |
| `error` is non-null, `params: undefined` | Show a resolution error, separate from a missing record. |

Private dynamic pages (`authenticated`, `group`, or `user`) do not provide SSR-resolved `params` or automatically redirect to a 404 for a missing entity. Handle these outcomes on the client. A rendered unavailable state does not change the server response to HTTP 404. Public dynamic pages retain server entity validation.

Do not treat pending `params` as a route without parameters, an empty list, or a request to create a new record. The hook updates as the route changes. In published apps, resolution also refreshes after sign-in, sign-out, and window focus. Handle subsequent loading and error states as well as the first load.

### Private detail-page example

Use the page's configured parameter name and table ID. Define `loading`, `notFound`, and `loadError` in every locale's block copies. Keep fetching in a child so it only runs with a resolved ID; key that child by ID so navigation cannot briefly render the previous record.

```tsx
import { useTypeQuery } from "@/api";
import type { IBuildingBlockProps } from "@/lib/types";
import { useParams } from "@/navigation";

interface IConfig {
  tableId: string;
  copies: { loading: string; notFound: string; loadError: string };
}

const ProductDetails = ({
  productId,
  config,
}: { productId: string; config: IConfig }) => {
  const { data, isLoading, error } = useTypeQuery(config.tableId, productId);
  if (isLoading) return <p role="status">{config.copies.loading}</p>;
  if (error) return <p role="alert">{config.copies.loadError}</p>;
  if (!data) return <p>{config.copies.notFound}</p>;
  return <h1>{typeof data.title === "string" ? data.title : ""}</h1>;
};

export const BuildingBlock = ({ config }: IBuildingBlockProps<IConfig>) => {
  const { params, isLoading, notFound, error } = useParams();
  if (isLoading) return <p role="status">{config.copies.loading}</p>;
  if (error) return <p role="alert">{config.copies.loadError}</p>;
  if (notFound || !params?.product) return <p>{config.copies.notFound}</p>;
  return <ProductDetails key={params.product} productId={params.product} config={config} />;
};
```

The record query can still return an error or no record after route resolution succeeds, so handle those states too. Do not gate this client-only block on `dynamicData` or fall back to the `params` prop while the hook is loading.

If the app calls for navigation instead of inline unavailable UI, resolve the destination with `useLinks`, then call `navigate(resolvedUrl, { replace: true })` from an effect after `notFound` becomes true. Do not navigate during render or use `shallow: true` when changing pages.

## `useSearchParams()` — live query-string values

Returns `Record<string, string>` directly, with no loading/error wrapper. It is not a `URLSearchParams` object: read `searchParams.q`, not `.get("q")`. Missing keys are absent; for repeated query keys, the first value is used.

```tsx
import type { IBuildingBlockProps } from "@/lib/types";
import { navigate, useSearchParams } from "@/navigation";

interface IConfig { copies: { search: string } }

export const BuildingBlock = ({ config }: IBuildingBlockProps<IConfig>) => {
  const searchParams = useSearchParams();
  const query = searchParams.q ?? "";
  const updateQuery = (value: string) => {
    const next = new URLSearchParams(searchParams);
    if (value) next.set("q", value);
    else next.delete("q");
    navigate(`?${next.toString()}`, { shallow: true, replace: true });
  };
  return <input type="search" aria-label={config.copies.search} value={query} onChange={(event) => updateQuery(event.target.value)} />;
};
```

Define the `search` copy in every block locale. Shallow query navigation keeps the current page and updates the hook; it does not rerun server dynamic data. Drive client filtering or data hooks from the returned values. Use normal `Link` or `navigate` for a different page or dynamic entity, and resolve localized destinations with `useLinks` or server `$$std.batchLink`.
