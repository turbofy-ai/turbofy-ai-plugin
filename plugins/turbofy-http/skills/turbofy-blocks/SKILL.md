---
name: turbofy-blocks
description: "Use when building or styling a Turbofy UI block such as navigation, hero, product grid, search, form, footer, card, filter, or interactive section, or when editing block-types/* runtime source. Covers the hosted app tree, block_type_open/block_type_check, React props, copies, navigation, data hooks, and UI/accessibility rules. For placement and record.ts metadata load turbofy-apps; for server data logic load turbofy-dynamic-fields."
---

# Turbofy Blocks

A block type has build metadata in `record.ts` and optional React runtime source beginning at `index.tsx`. Both live in the hosted session tree.

## Source workflow

For app-wide work:

1. `app_pull` materializes every block under `workspaces/<environment>/<workspaceId>/apps/<appId>/block-types/`.
2. Inspect and edit with shared `fs_*` tools.
3. Run `block_type_check` for each changed sourced block.
4. `app_push` dry-runs, then compiles and publishes changed block sources together with app declarations when called with `dryRun: false`.

For an isolated existing block, `block_type_open` can stage just that block before the same `fs_*` and `block_type_check` loop. It may start dependency installation asynchronously; wait for the returned setup state before checking.

```text
block-types/<Name>/
  record.ts   # appBuilder.blockType declaration; not runtime code
  index.tsx   # optional entry; named export BuildingBlock
  helpers.ts  # optional sibling runtime source
  styles.css  # optional sibling runtime source
```

Rules:

- Do not import `record.ts` from runtime source.
- Runtime files may import siblings, but never another block type or app files outside their boundary.
- Supported runtime sources are `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, and `.json`.
- A block type without `index.tsx`/`index.ts` is valid and sourceless.
- Add custom dependencies to the app's user-owned `package.json`; pulls preserve it.
- Never edit `.base/` or `app.base.json`.

## Runtime props

```ts
export type IBuildingBlockProps<TConfig = unknown> = PropsWithChildren<{
  blockId: string;
  locale: string;
  config: TConfig;
  dynamicData?: Record<string, unknown>;
  searchParams: Record<string, string>;
  params: Record<string, string>;
  pageId: string;
  slug: string[];
}>;
```

- `locale` is the language source; do not infer it from the browser URL.
- User-visible strings come from `config.copies`.
- `dynamicData` is the optional server-rendered snapshot. Treat `undefined` as loading and an inner `null` as empty/not found.
- Read route state from `params`, `slug`, and `searchParams`, not `window.location`.
- For live query-string state after shallow navigation, use `useSearchParams()` from `@/navigation`.

Example:

```tsx
import type { IBuildingBlockProps } from "@/lib/types";

interface IConfig {
  copies?: { title?: string; empty?: string };
}

export const BuildingBlock = ({ config, dynamicData }: IBuildingBlockProps<IConfig>) => {
  if (dynamicData === undefined) return <div className="h-32 animate-pulse rounded-xl bg-muted" />;
  return <section><h2>{config.copies?.title}</h2></section>;
};
```

## Config, data, and copies

Edit `record.ts` for type-level metadata:

- `defaultConfig`: static server-side config such as copies, links, and layout settings
- `defaultDynamicData`: route/query-dependent initial data
- `localizations`: flat copy dictionaries keyed by app locale

Page modules may add per-instance `config`, `dynamicData`, and `localizations` on `appBuilder.block(...)`.

The builder injects localized dictionaries into `config.copies`. Never recreate that wrapper or hardcode translated UI text in React. Add every copy key to every app locale; empty placeholders are allowed.

Use `turbofy-dynamic-fields` for `$$self`, `$$args`, and `$$std` syntax.

## Data access

Use table ids from `schema.ts`/`table_list`, never display names.

| Surface | Use |
|---|---|
| `defaultConfig` | Copies, static links, stable layout settings |
| `defaultDynamicData` | SSR first paint based on route/search params |
| `useTypeQuery`, `useListTypes`, `useListTypesByParent` | Client reads and interaction |
| mutation hooks | Client creates, updates, and deletes |
| `useSearchTypes` | Full-text search when workspace search is enabled |
| `useLinks`, `useTranslations`, `useFileDocuments` | Client-side resolution after a fetch |
| `useUploadFile` | Browser uploads |
| `useWsSubscription` | Server-side record changes delivered to authenticated clients |

For SPA dashboards, prefer client hooks and use config only for copies/links. For SSR sites, use `dynamicData` for the first render and hooks for subsequent filtering, pagination, or mutation.

## Navigation

- Import `Link`, `navigate`, and `useSearchParams` from `@/navigation`.
- Resolve localized links with client `useLinks` or server `$$std.batchLink`; do not concatenate locale paths.
- Query-only navigation such as `navigate("?q=shoes", { shallow: true, replace: true })` keeps the current pathname and avoids full rerenders.
- Do not read or write `window.location` directly.

## UI requirements

- Use semantic HTML and a correct heading hierarchy.
- Use Tailwind theme tokens (`bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `bg-primary`) and support dark mode.
- Prefer shadcn/ui primitives for interactive controls and `lucide-react` for icons.
- Make layouts responsive; use a centered max-width container and intentional spacing.
- Provide copy-driven loading, empty, and error states. Skeletons should approximate the final footprint.
- Keep visible focus states and label icon-only controls with `aria-label`.
- Use short, non-essential motion with `motion-safe:`; avoid mount animations in the iframe.
- Render only product UI, not implementation notes or design commentary.

## Validation checklist

1. `BuildingBlock` is a named export.
2. TypeScript and imports are clean.
3. Copy keys exist for every locale.
4. Data calls use table ids and handle loading/empty/error states.
5. Navigation uses Turbofy helpers.
6. `block_type_check` passes before `app_push`.

## See also

- `turbofy-apps` — app tree, page placement, block records, localization, auth
- `turbofy-dynamic-fields` — server-side config/data runtime
- `turbofy-platform` — schema, records, and file uploads
- `turbofy-chatbot` — Thread/Message, flow, and WebSocket chat recipe
