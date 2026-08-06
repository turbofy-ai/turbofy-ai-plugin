---
name: turbofy-blocks
description: "Use when building or styling a UI section/component in a Turbofy app — navigation bars, hero banners, product grids, search boxes, forms, footers, cards, lists, filters, or any interactive page section. Triggers: 'build a navigation bar', 'create a product listing', 'style the hero', 'add a search box', 'make this section responsive', 'add dark mode', 'show loading/empty states', 'fetch data in this component', or when editing a block type's React source. Covers the remote block_type_* build session, React block components, translations/copies, navigation, data hooks, and UI/accessibility rules. For placing a section on a page or editing defaultConfig/localizations, load turbofy-apps (app_push). For server-side data that feeds the section, load turbofy-dynamic-fields."
disable-model-invocation: false
---

# Turbofy Blocks

Runtime React component for a Turbofy block type. Sources live in a **remote build session** (not under `~/.turbofy`). Companions: `turbofy-apps` (placement / manifest / `app_push`), `turbofy-dynamic-fields` (`defaultConfig` / `defaultDynamicData`).

## When to load this skill

Load when the user wants to **build or restyle a page section** — e.g. "create a product grid", "add a sticky navbar".

---

## 1) Remote build session (`block_type_*`)

There is no local app checkout. Edit React sources through MCP:

| Tool | Purpose |
|---|---|
| `app_pull` | Materialize the **whole app** (all block sources + pages + schema + docs) into the session tree; starts dependency install (async). Preferred when touching more than one block type — see `turbofy-apps` § 5. |
| `block_type_open` | Open a single block type by `blockTypeId` into the user's build session; starts dependency install (async). Returns file paths and the block type's `appId`. |
| `fs_list` | List files (`apps/<appId>/block-types/…` or `workspaces/<environment>/<workspaceId>/apps/<appId>/block-types/…`) |
| `fs_read` | Read a file (optional `offset` / `limit` line range) |
| `fs_write` | Write full UTF-8 file content (new files; prefer `fs_edit` for existing) |
| `fs_edit` | Exact string replacements in an existing file (atomic, in-order) |
| `fs_search` | Grep the session tree (regex or `fixedStrings`), skips `node_modules` |
| `block_type_check` | `tsc` + esbuild check (requires `appId`; needs session deps ready) |
| `block_type_push` | Compile, upload source + artifact, update the `cmsbuildingblocktype` record. Requires `appId` + `blockTypeId`. Run `block_type_check` first. |

All calls need `orgId` + `workspaceId`. Session edits persist across sessions (write-through to workspace storage; restored transparently).

### Loop

1. Ensure the block type exists on the app (via `app_get` / `app_push` — metadata, `localizations`, `defaultConfig`).
2. Resolve `blockTypeId` (and `Name`) from `app_get`.
3. `app_pull` (whole app — sources for every block type land in the tree) **or** `block_type_open` (one type) → wait until the install is ready (`install.state` / `workspaceSetup.state`).
4. `fs_read` / `fs_write` under `apps/<appId>/block-types/<Name>/` (or the scoped `workspaces/…` form returned by open).
5. `block_type_check` with `{ orgId, workspaceId, appId, blockTypeName }` → fix errors → `block_type_check` again.
6. `block_type_push` with `{ orgId, workspaceId, appId, blockTypeId }`.
7. Confirm with `app_get` (`sourceCodeUrl` / `compiledCodeUrl` updated).

`app_push` does **not** compile React sources. `block_type_push` does **not** reconcile pages/block placement — use `app_push` for that.

### Creating a new block type

1. `app_get` → add a `blockTypes[]` entry (`name`, `localizations`, `defaultConfig` / `defaultDynamicData` as needed). For a brand-new type, follow dry-run feedback on whether to omit `id` or supply a client id.
2. Place instances under `pages[].blocks` with `blockTypeId` + `position`.
3. `app_push` (dry-run then apply).
4. Run the `block_type_*` loop above for `index.tsx`.

### Layout inside the session

```
workspaces/<environment>/<workspaceId>/apps/<appId>/
  package.json       # session-managed scaffold
  tsconfig.json
  block-types/<Name>/
    index.tsx        # required entry — named export BuildingBlock
    helpers.ts       # optional siblings
    …
```

The session mirrors the stdio MCP's `~/.turbofy` tree, one app tree per app.

- One folder per block type. Sibling imports only; **no imports of other block types**.
- Allowed: `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.json`.
- Entry must export `BuildingBlock`.
- Dependencies: only what the session stdlib provides (React, zod, Tailwind, lucide-react, shadcn via `@/components/ui/*`, `@/api`, `@/navigation`, …).

---

## 2) `IBuildingBlockProps`

```ts
export type IBuildingBlockProps<TConfig = any> = PropsWithChildren<{
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

| Prop | Notes |
|---|---|
| `locale` | **Only** source of language — never URL / navigator |
| `config` | Static config; **UI strings at `config.copies`** |
| `dynamicData` | Optional SSR snapshot; if used, handle `undefined` as loading |
| `searchParams` / `params` / `slug` | From props only — never `window.location`. `searchParams` is the render-time snapshot; for live values after shallow navigation use `useSearchParams()` from `@/navigation` |

### Coding rules

- **Copies:** all user-visible text from `config.copies` (flat object). Prefer zod defaults for missing keys. No `t()` helpers. Rare escape: `useTranslations`. Maintain dictionaries in the app manifest (`blockTypes[].localizations`) via `app_push`.
- **Navigation:** `navigate` / `Link` from `@/navigation` only. Resolve paths with `useLinks` (client) or `$$std.batchLink` (server) — do not hand-build localized URLs.
- **Query-param state (filters / search / tabs):** `navigate(path, { shallow: true, replace: true })` updates the URL without re-rendering the page; a query-only path (`"?q=shoes"`) keeps the current pathname. Read reactively with `useSearchParams()` from `@/navigation` (returns `Record<string, string>`); the `searchParams` prop remains the render-time snapshot for initial state. Use `replace: true` when syncing rapidly-changing state (typing in a search box) to avoid history spam.
- **Styling:** Tailwind. Prefer shadcn/ui for interactive primitives.
- **Loading:** if you read `dynamicData`, skeleton when `undefined`; treat inner `null` as empty/not found. Skeleton footprint ≈ final layout.
- **Table IDs:** all `@/api` hooks and `$$std` helpers take **table ids** from `workspace_get` → `schema.types[].id`, never table names. System CMS: `"cmspage"`, `"cmslocalization"`, `"filedocument"`, etc.

### Localizations & `defaultConfig`

Edit via **`app_push`** on the manifest (`turbofy-apps`). Prefer unwrapped user config JS; the platform applies copies guards. React-only changes: `block_type_*` loop above.

---

## 3) Data fetching patterns

| Tool | Where | Use for |
|---|---|---|
| `config` (`defaultConfig`) | Server | Static: copies (injected), links, layout; depends on `$$args.lang` |
| `dynamicData` | Server | Initial route/query data (`$$args.params` / `searchParams`) — SSR apps |
| `useTypeQuery` / `useListTypes` / `useListTypesByParent` | Client | Live lists/records (SPA + post-SSR interactivity) |
| `useSearchTypes` / `searchTypes` | Client | Full-text search (workspace search must be enabled) |
| `useLinks` / `useTranslations` / `useFileDocuments` | Client | Resolve after client fetch |
| `useUploadFile` | Client | Uploads |

**SPA / internal tools:** `config` for copies/links only; **no** `dynamicData`; fetch with hooks.

**SSR sites:** `config` + `dynamicData` for first paint; hooks for filters/pagination/mutations after load.

### Server-side examples (dynamic field JS)

```js
// defaultConfig — static
const [productsPath] = $$std.batchLink([{ pageId: "PAGE_ID" }]);
({ productsPath, columns: 3 });
```

```js
// defaultDynamicData — route-dependent
const productId = $$args.params?.productId;
const product = $$std.getRecord("TABLE_ID", productId, { dynamicArgs: $$args });
({ product });
```

See `turbofy-dynamic-fields` for the full `$$std` API.

### Client hooks (essentials)

```tsx
import { useListTypes, useLinks, useSearchTypes, useUploadFile } from "@/api";
import { Link, navigate, useSearchParams } from "@/navigation";

// URL-synced filter state: read live, write shallow
const searchParams = useSearchParams();
const setQuery = (q: string) =>
  navigate(`?q=${encodeURIComponent(q)}`, { shallow: true, replace: true });

const { data: products } = useListTypes(productTableId);
const { result: productLinks } = useLinks(
  (products ?? []).map((p) => ({
    pageId: productPageId,
    path: `/products/${p.id}`,
  })),
);

const { isPending, data } = useSearchTypes(
  productTableId,
  searchQuery,
  ["name", "description"],
  { dynamicArgs: { lang: locale }, limit: 50 },
);
```

`useSearchTypes`: non-empty `query` + `fields` required or it no-ops. No server-side `$$std.searchTypes`.

---

## 4) UI & UX (required)

Production-ready, responsive, consistent. Prefer shadcn/ui over raw HTML for interactive controls.

### Typography

| Role | Classes |
|---|---|
| Display | `text-4xl md:text-5xl lg:text-6xl font-bold tracking-tight` (max one) |
| h2 | `text-2xl md:text-3xl font-semibold tracking-tight` |
| h3 | `text-xl md:text-2xl font-semibold` |
| Lead | `text-lg md:text-xl text-muted-foreground leading-relaxed` |
| Body | `text-base leading-relaxed` + `max-w-prose` where appropriate |
| Caption | `text-sm text-muted-foreground` |
| Eyebrow | `text-xs font-semibold uppercase tracking-widest text-primary` |

Semantic heading order; no skipped levels.

### Layout & color

- Intentional whitespace; avoid cramped or sparse extremes.
- Use theme tokens (`bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `bg-primary`, …). Support light + dark.
- Cards only when they group an interaction; prefer open layouts for marketing sections.

### Interaction & a11y

- Visible focus rings; `aria-label` on icon-only controls.
- `transition-colors duration-200` on interactive elements; motion ≤ 300ms; `motion-safe:` for non-essential motion. **No mount entry animations** (iframe timing).
- Loading skeletons (`animate-pulse` + `bg-muted`); empty + error states from **copies**, never raw errors.
- Icons: `lucide-react`, consistent sizes (`size-4` / `size-5` / `size-6`).

### Content rules

Only real product UI copy — no tips, meta-commentary, or styling explanations in the rendered block.

---

## 5) Auth system blocks

When an app uses end-user auth (`app.settings.auth`), Login/Signup system blocks call `/api/auth/login` and `/api/auth/signup`. Custom login pages must stay `visibility: "public"`. Configure via `app_push` (`turbofy-apps`).

---

## See also

- **`turbofy-apps`** — manifest, `app_push`, localization, placement.
- **`turbofy-dynamic-fields`** — `$$std` / reserved `dynamicArgs`.
- **`turbofy-platform`** — `workspace_get` for table ids, CMS ofTypes.
