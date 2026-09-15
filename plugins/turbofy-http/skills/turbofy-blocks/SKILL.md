---
name: turbofy-blocks
description: "Build or style Turbofy React blocks, shared utilities and state, client data interactions, or custom authentication and social-login UI. Uses hosted app sources and block validation. For page placement use turbofy-apps; for server data logic use turbofy-dynamic-fields."
---

# Turbofy Blocks

A block type has build metadata in `record.ts` and optional React runtime source beginning at `index.tsx`. Both live in the hosted session tree.

## Source workflow

For app-wide work:

1. `app_pull` materializes every block under `workspaces/<environment>/<workspaceId>/apps/<appId>/block-types/`.
2. Inspect and edit with shared `fs_*` tools.
3. Run `block_type_check` for each changed sourced block.
4. `app_push` dry-runs, then compiles and publishes changed block sources together with app declarations when called with `dryRun: false`.

For app-scoped shared modules, use the full `app_pull` tree. For an isolated existing block without shared-module dependencies, `block_type_open` can stage just that block before the same `fs_*` and `block_type_check` loop. It may start dependency installation asynchronously; wait for the returned setup state before checking.

```text
block-types/<Name>/
  record.ts   # appBuilder.blockType declaration; not runtime code
  index.tsx   # optional entry; named export BuildingBlock
  helpers.ts  # optional sibling runtime source
  styles.css  # optional sibling runtime source
```

Rules:

- Do not import `record.ts` from runtime source.
- Runtime files may import siblings, platform modules, installed dependencies, and `@/shared/<Name>` exports. Do not import another block type or unrelated app declarations. See [shared utilities and state](references/shared-modules.md) for reusable code and Zustand stores.
- Supported runtime sources are `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, and `.json`.
- A block type without `index.tsx`/`index.ts` is valid and sourceless.
- Add custom dependencies to the app's user-owned `package.json`; pulls preserve it. Install them in the app project with `fs_exec` and wait for completion before validation.
- Never edit `.base/` or `app.base.json`.

## Runtime props

```ts
import type { PropsWithChildren } from "react";

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

export const BuildingBlock = ({
  config,
  dynamicData,
}: IBuildingBlockProps<IConfig>) => {
  if (dynamicData === undefined)
    return <div className="h-32 animate-pulse rounded-xl bg-muted" />;
  return (
    <section>
      <h2>{config.copies?.title}</h2>
    </section>
  );
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

Import client data helpers from `@/api`. Use table ids from `schema.ts`/`table_list`, never display names. Read [client data helper signatures](references/client-data.md) when implementing reads, mutations, search, uploads, or subscriptions.

| Surface                                                | Use                                                            |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `defaultConfig`                                        | Copies, static links, stable layout settings                   |
| `defaultDynamicData`                                   | SSR first paint based on route/search params                   |
| `useTypeQuery`, `useListTypes`, `useListTypesByParent` | Client reads and interaction                                   |
| mutation hooks                                         | Client creates, updates, and deletes                           |
| `useQueryTypes`, `useSearchTypes`                      | Structured filtering and full-text search on searchable tables |
| `useLinks`, `useTranslations`, `useFileDocuments`      | Client-side resolution after a fetch                           |
| `useUploadFile`                                        | Browser uploads                                                |
| `useWsSubscription`                                    | Server-side record changes delivered to authenticated clients  |

For SPA dashboards, prefer client hooks and use config only for copies/links. For SSR sites, use `dynamicData` for the first render and hooks for subsequent filtering, pagination, or mutation.

## Navigation — mandatory

- ALL navigation between pages in the same app MUST use `Link` or
  `navigate` imported from `@/navigation`.
- NEVER author an HTML `<a>` element for an internal app destination.
  `<a>` is permitted ONLY for external links.
- A destination is internal if it points to a page in this app, whether
  expressed as a relative path, an absolute URL, a localized URL, or a
  URL returned by `useLinks` or `$$std.batchLink`. An absolute URL does
  not make an app page external.
- Use `Link` for clickable internal links, including menus, breadcrumbs,
  cards, logos, and buttons styled as links. Use `navigate` for
  programmatic navigation from event handlers or effects.
- This rule also applies to shared components and UI primitives:
  do not bypass it with a component that renders a raw anchor.
  When using an `asChild` composition for internal navigation, supply
  `Link` as the child.
- Resolving a URL does not perform navigation. Pass internal URLs
  resolved by `useLinks` or `$$std.batchLink` to `Link` or `navigate`,
  NEVER to `<a href={...}>`.
- Resolve localized links with client `useLinks` or server
  `$$std.batchLink`; do not concatenate locale paths.
- Import route hooks `useParams` and `useSearchParams` from
  `@/navigation`; see [navigation hooks](#navigation-hooks).
- Query-only navigation such as
  `navigate("?q=shoes", { shallow: true, replace: true })`
  keeps the current pathname. Do not use `shallow: true` when
  navigating to a different page or dynamic entity.
- Do not read or write `window.location` directly.

## Authentication

Use `@/lib/auth` for app sessions, including `useCurrentUser`, password forms, recovery, and `signInWithSocialProvider`. Read [auth helper usage](references/auth.md) for signatures, result handling, and preview/published callbacks. Configure access and provider prerequisites with `turbofy-apps`.

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
4. Data calls use table ids and handle loading/empty/error states. For private dynamic pages, also verify a valid entity, a missing entity, resolution errors, and navigation between entities.
5. Inspect navigation in changed blocks and shared components before
   publishing. Every internal destination MUST use `Link` or `navigate`
   from `@/navigation`. Inspect every authored `<a>` and every component
   that renders an anchor: each must point to an explicitly external
   destination. Trace variable URLs to their source; do not assume they
   are external. Fix violations before running `app_push`.
6. `block_type_check` passes before `app_push`; inspect block and shared-module failures in both dry-run and apply results.
7. Verify related blocks together when they consume shared state, and test the auth flow in the intended preview or published environment.

## See also

- `turbofy-apps` — app tree, page placement, block records, localization, auth
- `turbofy-dynamic-fields` — server-side config/data runtime
- `turbofy-platform` — schema, records, and file uploads
- `turbofy-chatbot` — Thread/Message, flow, and WebSocket chat recipe
