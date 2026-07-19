---
name: turbofy-blocks
description: "Use when building or styling a UI section/component in a Turbofy app — navigation bars, hero banners, product grids, search boxes, forms, footers, cards, lists, filters, or any interactive page section. Triggers: 'build a navigation bar', 'create a product listing', 'style the hero', 'add a search box', 'make this section responsive', 'add dark mode', 'show loading/empty states', 'fetch data in this component', or when editing block-types/*/index.tsx. Covers React block components, translations/copies, navigation, data hooks, and UI/accessibility rules. For placing an existing section on a page, load turbofy-apps. For server-side data that feeds the section, load turbofy-dynamic-fields."
disable-model-invocation: false
---

# Turbofy Blocks

This skill covers the runtime React component of a Turbofy block type — the `block-types/<Name>/index.tsx` file inside a pulled app directory (`~/.turbofy/workspaces/<env>/<workspaceId>/apps/<appId>/`). The companion skills `turbofy-apps` (app workflow + data model) and `turbofy-dynamic-fields` (server-side `defaultConfig` / `defaultDynamicData` code) cover those areas.

## When to load this skill

Load when the user wants to **build or restyle a page section** — e.g. "create a product grid", "add a sticky navbar", "make the form look better". Do not wait for them to mention `index.tsx` or block-type names.

---

## 1) Block types

### IBuildingBlockProps interface

```ts
export interface IBuildingBlockProps<TConfig = unknown> {
  blockId: string;
  locale: string;
  config: TConfig;
  dynamicData?: Record<string, unknown>;
  searchParams: Record<string, string>;
  params: Record<string, string>;
  pageId: string;
  slug: string[];
}
```

| Prop           | Description                                                                                                                            |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `blockId`      | Unique ID of this block instance.                                                                                                         |
| `locale`       | Current language code (e.g. `"en"`, `"de"`). **Always read locale from this prop** — never hardcode or derive it from the URL.         |
| `config`       | Static block configuration (copies, links, layout). **Copies live at `config.copies`** — always read them from there.                  |
| `dynamicData`  | Optional server-resolved initial data snapshot. In SSR apps, use this for the initial render and use hooks for later interactions. Blocks that read it **must handle `undefined`**. |
| `searchParams` | Current URL query parameters as key/value pairs. **Always read query params from this prop** — never parse `window.location` directly. |
| `params`       | Resolved entity IDs for dynamic slug segments (e.g. `{ articleId: "1234-..." }`). Same values as `$$args.params` on the server side.   |
| `slug`         | URL path segments after the locale prefix.                                                                                             |
| `pageId`       | ID of the page this block belongs to.                                                                                                  |

### Coding guidelines

Each block type lives in its own directory: `block-types/{Name}/`. The directory contains a `record.ts` manifest plus the optional runtime entry `index.tsx` (or `index.ts`) and any sibling files the block needs. Everything compiles to a single JS bundle for the iframe.

A block type can be **sourceless** — `record.ts` only, no `index.tsx`. This is valid and means the block has no runtime React component (e.g. a content-only or backend-driven block). Push silently skips compile/upload for sourceless block types.

#### Layout rules

- One folder per block type: `block-types/{Name}/`.
- Manifest in `record.ts` — never imported by `index.tsx`.
- Runtime entry: `index.tsx` (or `index.ts`). Sibling files within the directory may import each other relatively.
- Allowed runtime extensions inside the folder: `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.json` (excluding `record.ts`).

On push, the directory is zipped (without `record.ts`) and uploaded as a `.source.zip` archive. On pull, archives are extracted back into the directory next to the freshly-decompiled `record.ts`.

#### Common rules

- **No importing other blocks**: Block types cannot import each other.
- **No imports outside the block folder**: Files in `block-types/{Name}/` may only import siblings within the same folder.
- **Never import `record.ts` from `index.tsx`** — `record.ts` is build tooling, not runtime, and pulls in `@graphapi-io/dsl-builders` (which transitively pulls `@graphapi-io/declarations`) and is not bundleable.
- **Dependencies**: Only use libraries/modules already installed in `app-std-lib` (React, zod, Tailwind, etc.).
- **Exports**: The runtime entry (`index.tsx` or `index.ts`) must have a named export `BuildingBlock`.
- **Styling**: Use Tailwind CSS utilities for styling.
- **Locale**: Always read the current language from `props.locale`. **Never** create helper functions like `getCurrentLocale()`, `getLocale()`, `detectLanguage()`, or similar. **Never** derive locale from the URL, `navigator.language`, `window.location.pathname`, cookies, or any other source. The prop is the single source of truth.
- **Localization**: All user-visible text **must** come from `config.copies`. **Never** hardcode strings inline (e.g. `"Submit"`, `"No results"`, `"Back to home"`).
  - Expect a flat `copies` object inside `config.copies`.
  - The actual copies live in the sibling `record.ts` under the `localizations` field, keyed by language code (e.g. `localizations: { en: { submit: "Submit" }, de: { submit: "Senden" } }`). The dsl-builders auto-inject them into `config.copies` for the current locale at runtime — see the `turbofy-apps` skill's "Localization workflow".
  - When you add a copy key to the React component, add it to **every** locale dictionary in `record.ts` (use `""` as a placeholder for missing translations). `Turbofy_app_pull` always seeds entries for every supported locale; never delete an entry just because it's empty.
  - Provide defaults for all copy keys in the React component (preferably validated/defaulted with `zod`) — they cover the case where a key is missing from the locale dictionary.
  - Access copies directly: `copies.title`, `copies.submitLabel`, etc. **Never** create a translation helper function (`t()`, `translate()` or similar wrappers). Zod defaults handle missing keys — an extra abstraction is unnecessary.
  - Even if the block only supports one language today, all text goes through `config.copies` so translations can be added later without code changes.
  - In VERY RARE cases where you can't resolve all your copies upfront, use the `useTranslations` hook.
- **Navigation**: Always use `navigate` from `@/navigation` for programmatic navigation — never `window.location` or raw `<a>` tags. Read current query parameters from `props.searchParams`, not from `window.location.search`.
- **Slug construction**: Do not build links yourself. Use `useLinks` (client-side) and `$$std.batchLink(...)` (server-side) — these are the only ways to correctly resolve localized paths.
- **Dynamic data loading state**: `dynamicData` is optional. If a block uses `dynamicData`, it must render a loading skeleton when `dynamicData` is `undefined`. The skeleton must match the final component's footprint as closely as possible (same outer spacing, major grid/column structure, image aspect ratios, heading/text line widths, card counts where reasonable) so the block itself can be used as its Suspense fallback without causing layout jumps. Treat `undefined` as "loading"; treat `null` values inside `dynamicData` as "loaded but empty/not found."

**Example A — static page link** (page slug config: `{ en: { slug: "/products" }, de: { slug: "/produkte" } }`):

Resolve the link server-side in `defaultConfig`:

```ts
defaultConfig: `
  const [productsPath] = $$std.batchLink([{ pageId: "${productsPage.id}" }]);
  ({ productsPath });
`,
```

> `copies` is auto-injected by the builder — don't add it yourself. See the `turbofy-apps` skill's "Auto-injected `copies`".

Then navigate in the component:

```tsx
const { productsPath } = config;
// declarative
<Link to={productsPath}>{copies.productsLink}</Link>;
// imperative
navigate(productsPath);
```

**Example B — dynamic page links for a list** (page slug config: `{ en: { slug: "/products/[Product]" }, de: { slug: "/produkte/[Product]" }, paramsCollectionMap: { Product: { ofType: "abcdef" } } }`):

Fetch the list client-side and resolve links with `useLinks`:

```tsx
const { data: products } = useListTypes("abcdef");
const { result: productLinks } = useLinks(
  (products ?? []).map((p) => ({
    pageId: productPageId,
    path: `/products/${p.id}`,
  })),
);
// productLinks[i] is the resolved localized path for products[i]
```

### UI & UX design

Every block must be production-ready, pixel-perfect, responsive, and visually consistent. Follow these rules strictly.
For common interactive primitives, prefer shadcn/ui over vanilla HTML-only implementations.
Use native HTML as the semantic foundation, but default to shadcn components for authored UI unless there is a documented reason not to.

#### Typography hierarchy

A clear type scale is the single biggest driver of visual quality. Follow a consistent hierarchy across all blocks:

| Role                    | Tailwind classes                                               | Usage                                                   |
| ----------------------- | -------------------------------------------------------------- | ------------------------------------------------------- |
| Display / hero headline | `text-4xl md:text-5xl lg:text-6xl font-bold tracking-tight`    | One per block maximum. Landing heroes, splash sections. |
| Section heading (h2)    | `text-2xl md:text-3xl font-semibold tracking-tight`            | Primary heading inside a block.                         |
| Sub-heading (h3)        | `text-xl md:text-2xl font-semibold`                            | Card titles, sub-sections.                              |
| Lead / intro paragraph  | `text-lg md:text-xl text-muted-foreground leading-relaxed`     | Short paragraph directly below a heading.               |
| Body text               | `text-base leading-relaxed`                                    | Default prose, descriptions, list items.                |
| Caption / helper        | `text-sm text-muted-foreground`                                | Timestamps, labels, footnotes.                          |
| Overline / eyebrow      | `text-xs font-semibold uppercase tracking-widest text-primary` | Category labels above headings.                         |

Rules:

- Use **one** display heading per block at most. Multiple oversized headings fight for attention.
- Pair every heading with a sub-heading or lead paragraph for context — headings in isolation feel unfinished.
- Set `leading-relaxed` (1.625) or `leading-7` on body text. Tight line heights hurt readability.
- Limit body text line length to `max-w-prose` (~65 characters) for comfortable reading.
- Use `tracking-tight` on headings ≥ `text-2xl` and `tracking-normal` or `tracking-wide` on small text.
- Use semantic heading tags (`h1`–`h4`) in order — never skip levels.

#### Visual hierarchy & whitespace

Great design is mostly about what you leave empty. Use whitespace intentionally to group, separate, and guide the eye.

- **Section breathing room**: separate major content groups with `space-y-12 md:space-y-16` or equivalent margin. Tight sections look cramped.
- **Content grouping**: related elements (heading + paragraph + CTA) should be close together (`space-y-4`), with larger gaps between unrelated groups.
- **Visual weight**: use size, color, and weight to create a clear reading order: primary heading → supporting text → CTA → secondary content. The user should know where to look first within 1 second.
- **Asymmetry over monotony**: hero sections benefit from a text-left + image-right (or vice versa) split (`grid-cols-1 lg:grid-cols-2`). Avoid centering everything — centered text works for short headings and CTAs, not for paragraphs.
- **Limit content density**: each block should communicate one idea. If a block has more than 3–4 distinct content groups, split it into multiple blocks.
- **Z-pattern / F-pattern**: for content-heavy blocks (articles, feature grids), arrange elements along the natural F-pattern reading flow (left-to-right, top-to-bottom).

#### Layout & structure

- Each block is one reusable website section (hero, product grid, testimonials, FAQ, footer, etc.).
- All blocks must be full width (`w-full`) with inner content wrapped in `max-w-7xl mx-auto`.
- Apply consistent horizontal padding: `px-4` mobile, `md:px-6` tablet, `lg:px-8` desktop.
- Block vertical padding should be between `16px` and `40px`.
- Use **flexbox** for general layouts. For card/widget grids, use Tailwind grid (see below).
- Do **not** use viewport-specific units (`100vh`, `100dvh`, etc.) — blocks render inside iframes.
- Every block must set a **background color**.

#### Grid system

- Use Tailwind grid with consistent column definitions (`grid-cols-*`).
- All grid items must be equal width and equal height (`w-full h-full`).
- Use `items-stretch` on the grid container so cells auto-align.
- Apply consistent gaps (`gap-6`, `gap-8`).
- Collapse to `grid-cols-1` on mobile, distribute evenly (`grid-cols-2`, `grid-cols-3`, `grid-cols-4`) on larger screens.
- `grid-cols-*` must never exceed the number of elements shown.

#### Images

- All images must use **absolute public URLs** (Unsplash, Wikimedia, etc.) in defaults/examples.
- Images must fill their container: `w-full h-full object-cover`.
- Maintain aspect ratios using `aspect-video`, `aspect-square`, etc.
- Add `overflow-hidden` when using `aspect-*` or `rounded-*` classes.
- No letterboxing, distortion, or blank gaps.
- For card/widget layouts, images go either edge-to-edge or have equal padding on all sides — never mix.

#### Color theming strategy

Blocks should feel cohesive across an entire app, not like isolated islands with random colors.

- **Build from neutrals**: the majority of a block's surface area should be neutral (white, off-white, gray, near-black). Color is an accent, not a background for everything.
- **One primary, one accent at most**: use the `primary` theme color for CTAs and key interactive elements. Introduce at most one secondary accent (e.g. for badges, highlights). More than two accent colors creates visual noise.
- **Surface layering**: create depth with subtle background shifts — e.g. a section at `bg-background` containing cards at `bg-muted` (or vice versa). Avoid flat, single-color layouts.
- **Semantic color usage**: use color consistently for meaning — success (green), warning (amber), error (red), info (blue). Don't repurpose these for decoration.
- **Tint, don't saturate**: for colored backgrounds (e.g. a hero with a brand-colored backdrop), use a low-opacity tint (`bg-primary/5`, `bg-primary/10`) rather than the full saturated color. Full saturation backgrounds make text hard to read and look dated.
- **Border and divider colors**: use `border-border` or a neutral with low opacity (`border-gray-200`) — never primary or accent colors for structural dividers.

#### Colors & contrast (accessibility)

- Meet or exceed **WCAG AA** contrast; prefer **AAA** for body text (≥ 7:1 ratio).
- Light text on dark surfaces, dark text on light surfaces.
- Large or bold text may use a 4.5:1 ratio; normal text should aim for 7:1.
- Text and backgrounds must always pass contrast checks — verify when choosing theme colors.
- Use `text-foreground` for primary text, `text-muted-foreground` for secondary text. Avoid raw gray values that might break under different themes.
- For text on colored backgrounds (hero banners, CTAs), always verify the specific combination passes contrast — don't assume.

#### Dark mode

- Use Tailwind's semantic color tokens (`bg-background`, `text-foreground`, `bg-muted`, `text-muted-foreground`, `bg-card`, `border-border`) rather than hardcoded colors like `bg-white` or `text-gray-900`. Semantic tokens adapt automatically when a dark theme is active.
- When a hardcoded color is unavoidable, always provide a `dark:` variant (e.g. `bg-white dark:bg-gray-900`).
- Test that images, shadows, and borders still look intentional in dark mode — a white `shadow-lg` card on a dark background looks broken. Use `shadow-sm` or `shadow-none` with a `border` on dark surfaces instead.
- Avoid pure white (`#fff`) text on dark backgrounds and pure black (`#000`) backgrounds — use `gray-50`/`gray-100` for text and `gray-900`/`gray-950` for surfaces to reduce eye strain.

#### CTAs (call-to-action buttons)

- **Primary CTA**: use a `primary` theme color as background with readable text on top (e.g. white text on a dark primary).
- **Secondary CTA**: outline style — a `primary` color for borders and text, lower opacity or subtle tint.
- Both must maintain at least WCAG AA contrast with surrounding backgrounds.
- Primary CTAs must always look like buttons, even when wrapping a `Link`:

```tsx
<Button asChild>
  <Link to={path}>{copies?.cta ?? "Learn more"}</Link>
</Button>
```

#### Spacing & radius

- Block padding: `16px`–`40px`.
- Corner radius: `0px`–`32px` (use `rounded-lg` / `rounded-xl` consistently).
- Keep spacing rhythm consistent with the Tailwind spacing scale.

#### Responsiveness

- Mobile-first approach.
- Grids and content must scale smoothly across all breakpoints.
- No overflow, text wrapping issues, or broken layouts.
- Links, titles, and buttons should stay on a single line when possible.

#### Accessibility

- All interactive components must follow `shadcn/ui` accessibility patterns (focus states, ARIA roles).
- Touch targets ≥ 44px.
- Use semantic HTML elements (`header`, `nav`, `main`, `section`, `footer`).
- Decorative images get `alt=""`. Informational images get descriptive `alt` text from copies.
- Ensure keyboard navigation works: all interactive elements reachable via Tab, activatable via Enter/Space.
- Never rely on color alone to convey information — pair with text, icons, or patterns.

#### Motion & transitions

Subtle motion makes the difference between "functional" and "polished." Overdone motion makes it feel gimmicky.

- Add `transition-colors duration-200` to interactive elements (buttons, links, cards) for smooth hover/focus feedback.
- Use `hover:shadow-md` or `hover:-translate-y-0.5` on clickable cards for a lift effect — pick one, not both.
- For reveal/expand animations (accordions, dialogs), use `transition-all duration-300 ease-out`.
- **Never** add entry animations (fade-in, slide-up on mount) — blocks render inside iframes and may load at unpredictable times, causing jarring sequential animations.
- Keep all transitions under `300ms`. Anything slower feels sluggish.
- Respect `prefers-reduced-motion`: wrap non-essential motion in `motion-safe:` (e.g. `motion-safe:hover:-translate-y-0.5`).

#### Empty & loading states

A block that shows nothing while loading looks broken. Design for all data states:

- **Loading**: show skeleton placeholders that mirror the final layout shape. Use `animate-pulse` on `bg-muted rounded` divs matching the dimensions of headings, text lines, and images.
- **Empty**: when a list/grid has zero items, show a centered message with an icon and a brief explanation from copies (e.g. "No products found"). Never show an empty container with no context.
- **Error**: show a user-friendly message from copies. Never expose raw error text, stack traces, or technical IDs.
- Match skeleton/empty state dimensions to the real content so the layout doesn't jump when data arrives.

#### Iconography

- Use `lucide-react` for all icons.
- Keep icon sizes consistent: `size-4` (16px) inline with text, `size-5` (20px) in buttons, `size-6` (24px) standalone or in nav.
- Align icons vertically with adjacent text using `inline-flex items-center gap-2`.
- Use `text-muted-foreground` for decorative/secondary icons, `text-foreground` or `text-primary` for actionable icons.
- Don't use icons without adjacent text for critical actions — icon-only buttons must have `aria-label`.
- Prefer outlined icons (lucide default) over filled variants for consistency.

#### Content rules

- A block must **only** contain the page's content, images, and UI elements.
- **Never** include tips, hints, tooltips, guidance, or "how to use" messages.
- **Never** insert meta-commentary (phrases like "tip", "hint", "note", "advice").
- **Never** insert information about color palettes or styling choices.
- Every text element must belong to the website's actual interface or content (headings, paragraphs, CTAs, labels).

---

## 2) Data fetching patterns

### Critical: table IDs, not names

> **All `@/api` hooks and `$$std` helpers require table IDs, not table names.**
> For workspace schema tables, IDs are workspace-specific. If you've decompiled or initialized an app, the IDs should be in the `schema.ts` file in each of the table builder options, or for new tables accessible with `SomeTable.id`. As a last resort, you can use the `Turbofy_table_list` MCP tool to find the correct ID.
> For system CMS tables (App/Page/BuildingBlock/BuildingBlockType/Localization/Image), IDs are stable and come from `CmsOfTypeEnum.*`.
> Never hardcode a table name like `"Product"` where a table ID is expected.

### Available tools

| Tool                                                 | Where it runs | Reactive                                                                                 | Description                                                                                |
| ---------------------------------------------------- | ------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `config` (dynamic field)                             | Server-side   | No (resolved once at load)                                                               | Static configuration — copies, links, layout settings                                      |
| `dynamicData` (dynamic field)                        | Server-side   | No (initial server-provided data snapshot)                                               | Route/query-dependent initial data — collections, single records, aggregations, pagination |
| `useTypeQuery`/`useListTypes`/`useListTypesByParent` | Client-side   | Yes — automatically updates when data changes (e.g. after mutations via `useCreateType`) | Client-side data fetching with live updates                                                |
| `useSearchTypes`/`searchTypes`                       | Client-side   | No (one-shot, re-fetches when `query` or `fields` change)                                | Full-text search across indexed fields on a table (requires workspace search to be enabled) |
| `useLinks`/`useTranslations`/`useFileDocuments`      | Client-side   | No (one-shot, re-fetches when params change)                                             | Resolving links, translations, and file documents for data fetched client-side             |
| `useUploadFile`                                      | Client-side   | Mutation                                                                                 | Uploading files (returns file document metadata)                                           |

### Builder DSL syntax

In each `block-types/{Name}/record.ts`, use `defaultConfig` for static server-side config and `defaultDynamicData` for route/query-dependent server-side data. Both are JavaScript code strings — see the `turbofy-dynamic-fields` skill for the full runtime model. **Copies are auto-injected** into `defaultConfig` (and into block-instance `config`) by the dsl-builders — your code only needs to return the rest of the configuration. The wrap is a hard lock: any `copies` key you write yourself is overridden by the runtime-resolved dictionary. Only set `defaultConfig` when you need extra static config beyond copies (links, layout settings).

```ts
// block-types/ProductDetail/record.ts — page slug: /[productId]
export const productDetailBlock = appBuilder.blockType({
  id: "abc123",
  name: "ProductDetail",
  defaultDynamicData: `
    const productId = $$args.params?.productId;
    const product = $$std.getRecord('${ProductTable.id}', productId, { dynamicArgs: $$args });
    ({ product });
  `,
  localizations: {
    en: { title: "Product details", notFound: "Product not found" },
    de: { title: "", notFound: "" }, // empty placeholders — translate later
  },
});
```

Per-instance overrides use `config` / `dynamicData` on `appBuilder.block()`.

**Override** — replace the parent's dynamicData entirely:

```ts
appBuilder.block({
  type: productDetailBlock,
  dynamicData: `
    const productId = $$args.params?.productId;
    const product = $$std.getRecord('${ProductTable.id}', productId, {
      fields: ['id', 'name', 'price', 'imageUrl'],
      dynamicArgs: $$args,
    });
    ({ product });
  `,
});
```

**Extend** — inherit the parent's dynamicData and add extra fields using `blockDynamicDataCode()`:

```ts
appBuilder.block({
  type: productDetailBlock,
  dynamicData: `
    const parent = ${blockDynamicDataCode(BlockTypeTable.id)};
    const reviews = $$std.listRecords('${ReviewTable.id}', { limit: 5 })
      .filter(r => r.productId === parent?.product?.id);
    ({ ...parent, reviews });
  `,
});
```

### When to use what

The right data-fetching strategy depends on whether the app behaves like a **SPA** (single-page application — behind a login, internal tool, dashboard) or an **SSR app** (server-rendered pages with optional client-side interactivity afterwards).

#### `config`

- Use `config` for the static part of the page: copies, links, layout settings, and other data that depends on `$$args.lang`.
- Treat `config` as server-side setup for the initial render. Do not rely on route params there.
- Just return your config object — `copies` is auto-injected:
  ```ts
  defaultConfig: `({ columns: 3 });`;
  ```

#### `dynamicData`

- Use `dynamicData` for server-side data that depends on `$$args.params` and `$$args.searchParams`.
- In SSR apps, `dynamicData` is the right place for the initial route/query-dependent payload.
- For later interactions after the page loads (filters, pagination, mutations, re-fetches), switch to client hooks.

#### SPA apps (internal tools, dashboards, logged-in experiences)

- **`config`**: use for static copies, links, and layout only.
- **`dynamicData`**: do not use.
- **Hooks for data fetching**: use `useTypeQuery`, `useListTypes`, `useListTypesByParent`, `useSearchTypes`, and mutation hooks for all dynamic data.
- **`useLinks`/`useTranslations`/`useFileDocuments`**: use when links, translations, or file metadata depend on data fetched client-side.
- **`useUploadFile`**: use for file uploads.

#### SSR apps (server-rendered pages with client-side interactivity)

- **`config`**: use for static data — copies, links, layout settings. Resolve links via `$$std.batchLink(...)` and translations via `$$std.batchTranslate(...)` server-side when you know them upfront.
- **`dynamicData`**: use for the initial route/query-dependent payload — single records, collections, aggregations, pagination cursors, and similar data derived from `$$args.params` / `$$args.searchParams`.
- **Hooks for subsequent interactions**: after the initial render, use `useTypeQuery`, `useListTypes`, `useListTypesByParent`, `useSearchTypes`, and mutation hooks for interactive behavior (search, filters, load more, form submissions).
- **Login-gated sections**: if part of the app behaves like an SPA, treat that section as SPA-style and fetch dynamic data with hooks.

#### Resolution hooks reference

**`useLinks`/`useTranslations`/`useFileDocuments`** resolve links, translations, and file documents client-side. Use them when the underlying records were fetched client-side, or when an SSR app needs additional client-side resolution after the initial render. If you know the values upfront, prefer `$$std.batchLink`/`$$std.batchTranslate` in `config`/`dynamicData` instead — those resolve server-side and are faster.

```tsx
import {
  useLinks,
  useTranslations,
  useFileDocuments,
  useListTypes,
  useSearchTypes,
} from "@/api";

// Resolve links for records fetched client-side
const { data: products } = useListTypes(productTableId);
const { isPending: linksPending, result: links } = useLinks(
  (products ?? []).map((p) => ({
    pageId: productPageId,
    path: `/products/${p.id}`,
  })),
);

// Resolve translations client-side
const { isPending: translationsPending, result: translations } =
  useTranslations([
    { id: "home-page", path: "title" },
    { id: "home-page", path: "subtitle" },
  ]);

// Resolve file documents client-side (pass file document IDs)
const fileDocIds = (products ?? [])
  .map((p) => p.fileDocId as string)
  .filter(Boolean);
const { isPending: fileDocsPending, result: fileDocuments } =
  useFileDocuments(fileDocIds);
// fileDocuments: Array<{ id, name, key, mimeType, url, meta? } | null>
```

Arguments are deeply memoized — passing the same values will not trigger redundant requests.

**`useSearchTypes`/`searchTypes`** run a full-text search against a table's indexed search fields (`GET /rest/search/:ofType`). The workspace must have search enabled and fields configured for the target table (see data explorer search settings). There is no `$$std.searchTypes` server-side helper — use the client hook (or `searchTypes()` for imperative one-shot calls).

- **`typeId`** — table ID to search (not the table name).
- **`query`** — search string. Must be non-empty (after trim) for a request to fire.
- **`fields`** — array of field names to search within. Must be non-empty for a request to fire.
- **`options.dynamicArgs`** (optional) — passed through to nested dynamic field evaluation (e.g. `{ lang: props.locale }`).
- **`options.limit`** (optional) — max results (default 100).

Return shape: `{ items: IGenericItem[]; total: number; error?: string }`.

Hook return shape: `{ isPending: boolean; data: ISearchTypesResult | null; error: IApiError | null }`. When search is disabled (`query` empty or `fields` empty), the hook returns `{ items: [], total: 0 }` without an API call.

```tsx
import { useSearchTypes } from "@/api";

// Reactive search — re-fetches when query or fields change
const { isPending, data, error } = useSearchTypes(
  productTableId,
  searchQuery,
  ["name", "description"],
  { dynamicArgs: { lang: locale }, limit: 50 },
);

const products = data?.items ?? [];
const total = data?.total ?? 0;
```

```tsx
import { searchTypes } from "@/api";

// Imperative one-shot (e.g. on form submit)
const result = await searchTypes(productTableId, "widget", ["name", "sku"], {
  dynamicArgs: { lang: locale },
});
```

**`useUploadFile`** uploads a file (`uploadFile` → S3 PUT) and returns the resulting file document metadata.

```tsx
import { useUploadFile } from "@/api";

const {
  mutateAsync: uploadFile,
  isPending,
  error,
  data,
  reset,
} = useUploadFile();

// Upload a file from a file input
const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
  const file = e.target.files?.[0];
  if (!file) return;

  const result = await uploadFile({
    file,
    key: file.name, // storage key (will be normalized)
    accessControl: "PUBLIC", // optional, defaults to "PUBLIC"
    meta: { width: 800, height: 600 }, // optional metadata
  });
  // result: { id, name, key, mimeType, url }
};
```

### Accessing query params and route data

#### Server-side `config`

Use `config` for the static server-side part of the page:

- `$$args.lang` — current language

Do not rely on route-dependent values such as `$$args.params`, `$$args.slug`, or `$$args.searchParams` in `config`.

#### Server-side `dynamicData`

Use `dynamicData` for route/query-dependent server-side data:

- `$$args.lang` — current language
- `$$args.slug` — URL path segments as array (after the lang prefix)
- `$$args.params` — resolved entity IDs for dynamic slug segments (e.g. `{ articleId: "1234-..." }`)
- `$$args.searchParams` — current URL query parameters
- Custom args passed by the runtime when evaluating dynamic fields

**Extracting record IDs from slug segments:** Use `$$args.params` (preferred) or fall back to parsing slug segments.

```js
// Preferred: use resolved params (entity IDs already resolved from slugs)
const productId = $$args.params?.productId;
```

#### Client-side (block source code)

The same data is available via `IBuildingBlockProps`:

- `props.locale` — same as `$$args.lang`
- `props.slug` — same as `$$args.slug`
- `props.params` — same as `$$args.params`
- `props.searchParams` — current URL query parameters (e.g. `{ page: "2", q: "shoes" }`)

Always read these from props — never parse `window.location` directly.

### Data fetching examples

#### Single record (server-side via dynamicData)

**`block-types/ProductDetail/record.ts`:**

```ts
export const productDetailBlock = appBuilder.blockType({
  name: "ProductDetail",
  defaultDynamicData: `
    const productId = $$args.params?.productId;
    const product = $$std.getRecord('${ProductTable.id}', productId, { dynamicArgs: $$args });
    ({ product });
  `,
});
```

**`block-types/ProductDetail/index.tsx`:**

```tsx
import type { IBuildingBlockProps } from "../lib/types";

interface IConfig {
  copies?: { title?: string; notFound?: string };
}

interface IDynamicData {
  product: { id: string; name: string; price: number } | null;
}

const ProductDetailSkeleton = ({ title }: { title?: string }) => (
  <section className="w-full bg-background px-4 py-10 md:px-6 lg:px-8">
    <div className="mx-auto max-w-7xl">
      <p className="text-sm font-medium text-muted-foreground">
        {title}
      </p>
      <div className="mt-4 h-12 max-w-xl animate-pulse rounded bg-muted" />
      <div className="mt-6 h-6 max-w-2xl animate-pulse rounded bg-muted" />
    </div>
  </section>
);

export const BuildingBlock = ({
  config,
  blockId,
  locale,
  dynamicData,
}: IBuildingBlockProps<IConfig>) => {
  const data = dynamicData as IDynamicData | undefined;
  const copies = config?.copies;

  if (!data) {
    return <ProductDetailSkeleton title={copies?.title} />;
  }

  if (!data?.product)
    return <div>{copies?.notFound ?? "Product not found"}</div>;

  return <h1>{data.product.name}</h1>;
};
```

#### Collection initial load (dynamicData)

**`block-types/ProductList/record.ts`:**

```ts
export const productListBlock = appBuilder.blockType({
  name: "ProductList",
  defaultDynamicData: `
    const cursor = $$args.searchParams?.cursor;
    const result = $$std.listRecords('${ProductTable.id}', { limit: 10, cursor }, true);
    const currentPath = "/" + $$args.lang + "/" + $$args.slug.join("/");
    const nextPageHref = result.nextToken
      ? currentPath + "?cursor=" + encodeURIComponent(result.nextToken)
      : null;
    ({ products: result.items, nextPageHref });
  `,
});
```

**`block-types/ProductList/index.tsx`:**

```tsx
import { Link } from "@/navigation";
import type { IBuildingBlockProps } from "@/lib/types";

interface IConfig {
  copies?: { title?: string };
}

interface IDynamicData {
  products: Array<{ id: string; name: string }>;
  nextPageHref?: string | null;
}

export const BuildingBlock = ({
  config,
  dynamicData,
}: IBuildingBlockProps<IConfig>) => {
  const data = dynamicData as IDynamicData | undefined;
  const copies = config?.copies;

  if (!data) {
    return (
      <div>
        <h1>{copies?.title ?? "Products"}</h1>
        <ul>
          {[0, 1, 2].map((item) => (
            <li key={item} className="my-2 h-5 animate-pulse rounded bg-muted" />
          ))}
        </ul>
      </div>
    );
  }

  return (
    <div>
      <h1>{copies?.title ?? "Products"}</h1>
      <ul>
        {data?.products?.map((p) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
      {data?.nextPageHref && <Link to={data.nextPageHref}>Next page</Link>}
    </div>
  );
};
```

If the list needs client-side pagination, filtering, or refetching after the initial render, switch that interaction to `useListTypes` / `useListTypesByParent` rather than trying to refresh `dynamicData`.

#### Full-text search (useSearchTypes)

Use `useSearchTypes` when the user types into a search box and you need results from the workspace search index — not for simple field filters (those belong in `useListTypes` with client-side filtering, or in `dynamicData` with `$$std.listRecords`).

**`block-types/ProductSearch/index.tsx`:**

```tsx
import { useState } from "react";
import { useSearchTypes } from "@/api";
import type { IBuildingBlockProps } from "@/lib/types";

interface IConfig {
  productTableId: string;
  searchFields: string[];
  copies?: { placeholder?: string; noResults?: string };
}

export const BuildingBlock = ({
  config,
  locale,
}: IBuildingBlockProps<IConfig>) => {
  const [query, setQuery] = useState("");
  const copies = config?.copies;

  const { isPending, data } = useSearchTypes(
    config.productTableId,
    query,
    config.searchFields,
    { dynamicArgs: { lang: locale }, limit: 20 },
  );

  const items = data?.items ?? [];

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder={copies?.placeholder ?? "Search products"}
      />
      {isPending && <div className="h-5 animate-pulse rounded bg-muted" />}
      {!isPending && query.trim() && items.length === 0 && (
        <p>{copies?.noResults ?? "No results"}</p>
      )}
      <ul>
        {items.map((p) => (
          <li key={p.id as string}>{p.name as string}</li>
        ))}
      </ul>
    </div>
  );
};
```

Store `searchFields` in `defaultConfig` (static) or pass them from workspace schema knowledge. The field names must match those configured as searchable in the workspace.

#### Aggregating multiple sources for initial render

**`block-types/ProductPage/record.ts`:**

```ts
export const productPageBlock = appBuilder.blockType({
  name: "ProductPage",
  defaultDynamicData: `
    const productId = $$args.params?.productId;
    const product = $$std.getRecord('${ProductTable.id}', productId, { dynamicArgs: $$args });
    const manufacturer = $$std.getRecord('${ManufacturerTable.id}', product?.manufacturerId, { dynamicArgs: $$args });
    const relatedProducts = $$std.listRecords('${ProductTable.id}', { limit: 5 })
      .filter((p) => p.manufacturerId === product?.manufacturerId && p.id !== productId);
    ({ product, manufacturer, relatedProducts });
  `,
});
```

**`block-types/ProductPage/index.tsx`:**

```tsx
import type { IBuildingBlockProps } from "../lib/types";

interface IConfig {
  copies?: { byLabel?: string; relatedTitle?: string };
}

interface IDynamicData {
  product: { id: string; name: string; manufacturerId: string };
  manufacturer: { id: string; name: string };
  relatedProducts: Array<{ id: string; name: string }>;
}

export const BuildingBlock = ({
  config,
  blockId,
  locale,
  dynamicData,
}: IBuildingBlockProps<IConfig>) => {
  const data = dynamicData as IDynamicData | undefined;
  const copies = config?.copies;

  if (!data) {
    return (
      <div>
        <div className="h-10 max-w-lg animate-pulse rounded bg-muted" />
        <div className="mt-4 h-6 max-w-sm animate-pulse rounded bg-muted" />
        <div className="mt-8 space-y-3">
          {[0, 1, 2].map((item) => (
            <div key={item} className="h-5 animate-pulse rounded bg-muted" />
          ))}
        </div>
      </div>
    );
  }

  if (!data?.product) return <div>Not found</div>;

  return (
    <div>
      <h1>{data.product.name}</h1>
      <p>
        {copies?.byLabel ?? "By"} {data.manufacturer.name}
      </p>
      <h2>{copies?.relatedTitle ?? "Related"}</h2>
      <ul>
        {data.relatedProducts.map((p) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </div>
  );
};
```

#### Simple client-side fetch (useTypeQuery fallback)

**`block-types/QuickLookup/index.tsx`:**

```tsx
import { useTypeQuery } from "@/api";
import type { IBuildingBlockProps } from "../lib/types";

interface IConfig {
  productTableId: string;
  productId: string;
}

export const BuildingBlock = ({
  config,
  locale,
}: IBuildingBlockProps<IConfig>) => {
  const { data: product, isLoading } = useTypeQuery(
    config.productTableId,
    config.productId,
  );

  if (isLoading) return <div>Loading...</div>;
  if (!product) return <div>Not found</div>;

  return <h1>{product.name as string}</h1>;
};
```

No `defaultDynamicData` needed — the block fetches client-side. The query hooks are **reactive**: if another component creates/updates/deletes a record via `useCreateType`/`useUpdateType`/`useDeleteType`, all active `useTypeQuery`/`useListTypes`/`useListTypesByParent` subscriptions automatically receive the updated data. Use this pattern for SPA blocks and for post-load interactions in SSR apps.

Note: `refetch()` and `fetchNextPage()` on these hooks are fire-and-forget (`() => void`) — the updated data arrives reactively via the subscription, not as a return value.

#### Real-time WebSocket events (useWsSubscription)

`useWsSubscription` subscribes to real-time INSERT/UPDATE/DELETE events for a given table. Use it for imperative side-effects (toasts, cursor tracking, custom animations) and for real-time data sync in scenarios like chat, live feeds, or collaborative UIs where new records should appear immediately.

```tsx
import { useWsSubscription } from "@/api";
import type { IWsEvent } from "@/api";

export const BuildingBlock = ({
  config,
  locale,
}: IBuildingBlockProps<IConfig>) => {
  useWsSubscription(
    { ofType: config.productTableId, operations: ["INSERT", "UPDATE"] },
    (event: IWsEvent) => {
      // event.operation: "INSERT" | "UPDATE" | "DELETE"
      // event.record: the affected record
      console.log("Real-time event:", event.operation, event.record);
    },
  );

  return <div>Listening for changes...</div>;
};
```

- **`ofType`** is a **table ID** (not a table name) — same as all other hooks.
- **`operations`** (optional) filters which event types to receive. Omit to receive all three.
- The callback is **imperative** — it does not cause React re-renders. Store data in state yourself if needed.
- Under the hood, the iframe sends a `WS_SUBSCRIBE` message to the parent, which manages a shared WebSocket connection per workspace. Events are forwarded back as `WS_EVENT` messages.

### Performance guidelines

1. **Keep `config` static** — use it for copies, layout, and links that depend on `$$args.lang`, not for route/query-dependent record lookups.
2. **Mixed sites: use `dynamicData` for the initial route/query-dependent payload** — this avoids client round-trips on first render.
3. **Mixed sites: use hooks after load** — for search (`useSearchTypes`), filters, pagination, and other post-render interactions, switch to client hooks.
4. **SPA apps: use hooks for dynamic data** — `config` is only for upfront static copies/links.
5. **Resolve links/translations server-side when possible** — use `$$std.batchLink(...)` and `$$std.batchTranslate(...)` in `config`/`dynamicData`. Use `useLinks`/`useTranslations`/`useFileDocuments` only when resolving for data fetched client-side.
6. **Batch lookups** — when you need many records, prefer `$$std.batchGetRecordsByInputs(...)` over many `$$std.getRecord(...)` calls.

---

## 3) Building block links

### Core rules

1. **Resolve links server-side when the inputs are known server-side** — compute paths in `config` or `dynamicData`. If the underlying records are fetched client-side, resolve links with `useLinks`.
2. **Use `$$std.batchLink(...)` for page links** — it resolves `Page.localizedConfig.canonicalPath` for many pages in one call.
3. **Static links** (no fetched data needed) go in `defaultConfig`. In SSR apps, **data-dependent links for the initial render** can go in `defaultDynamicData`. For client-fetched data, use `useLinks`.
4. **Use `@/navigation`** in source code — never raw `<a>` tags or `window.location`.

### `$$std.batchLink(entries)`

`$$std.batchLink(...)` is the supported way to resolve page links. It fetches the target `Page` records and returns each page's resolved `localizedConfig.canonicalPath`.

Requirements:

- `$$args.lang` must be present (e.g. "en")
- `cmsOfTypes.page` must be configured (must be a **type ID**; for system CMS pages use `CmsOfTypeEnum.Page`)

Signature:

- `entries`: `Array<string | { pageId: string; path?: string; dynamicArgs?: object }>`
  - string entries are treated as `{ pageId: entry }`
  - `path` is converted to `slug` segments and merged into `dynamicArgs` if `dynamicArgs.slug` is not provided
- returns: `Array<string | null>` aligned to `entries` order

#### Static pages (no slug params)

```ts
// block-types/Navigation/record.ts
export const navBlock = appBuilder.blockType({
  name: "Navigation",
  defaultConfig: `
    const [aboutPath, pricingPath] = $$std.batchLink([
      { pageId: "${aboutPage.id}" },
      { pageId: "${pricingPage.id}" },
    ]);
    ({ aboutPath, pricingPath });
  `,
});
```

#### Page with dynamic slug segments

```ts
// block-types/ProductDetail/record.ts — productPage slug: /products/[productId]
export const productDetailBlock = appBuilder.blockType({
  name: "ProductDetail",
  defaultDynamicData: `
    const productId = $$args.params?.productId;
    const [productPath] = $$std.batchLink([
      { pageId: "${productPage.id}", path: \`/products/\${productId}\` },
    ]);
    ({ productPath });
  `,
});
```

#### Links in a list (single batch)

```ts
// block-types/ProductList/record.ts
export const productListBlock = appBuilder.blockType({
  name: "ProductList",
  defaultDynamicData: `
    const products = $$std.listRecords('${ProductTable.id}', { limit: 50, dynamicArgs: $$args }) || [];
    const paths = $$std.batchLink(products.map((p) => ({
      pageId: "${productPage.id}",
      path: \`/products/\${p.id}\`,
    })));
    const items = products.map((p, i) => ({ ...p, path: paths?.[i] ?? null }));
    ({ products: items });
  `,
});
```

### `@/navigation` in source code

Always use `Link` and `navigate` from `@/navigation` — never raw `<a>` tags or `window.location`.

```tsx
import { Link } from "@/navigation"; // declarative
import { navigate } from "@/navigation"; // imperative (e.g. after form submit)
```

Read `locale`, `slug`, and `searchParams` from `IBuildingBlockProps` — never parse `window.location` directly.

```tsx
export const BuildingBlock = ({
  config,
  blockId,
  locale,
  slug,
  searchParams,
  dynamicData,
}: IBuildingBlockProps<IConfig>) => {
  const data = dynamicData as IDynamicData | undefined;
  const copies = config?.copies;

  const handleNavigate = () => {
    navigate(`/${locale}/products/${data?.product?.id}`);
  };

  return (
    <div>
      {config.aboutPath && (
        <Link to={config.aboutPath}>{copies?.about ?? "About"}</Link>
      )}
      {data?.relatedProductPath && (
        <Link to={data.relatedProductPath}>
          {copies?.related ?? "Related product"}
        </Link>
      )}
      <button onClick={handleNavigate}>{copies?.viewProduct ?? "View"}</button>
    </div>
  );
};
```

---

## 4) System blocks (Login & Signup)

These are built-in blocks provided by the platform for end-user authentication. They are already in the block registry and do not need a `record.ts` or custom source — use them by referencing their block type when placing blocks on a page.

### Login block (`Login`)

**Purpose:** email + password sign-in form for end users.

Behavior:

- Uses the `login()` helper from `@/lib/auth` which POSTs to `/api/auth/login`.
- After successful login, redirects to `safeNextPath()` (honors the `?next=` query param for deep linking).
- The page containing the Login block should use `searchParams: dynamic` (it reads `?next=` from the URL). This is handled automatically by the page generator.
- **Must be placed on a public page** (since users need to reach it without auth). Set the page as `loginPageId` in the app's `auth` settings.

Localizable copies (via `config.copies`):

`title`, `description`, `imageUrl`, `emailLabel`, `emailPlaceholder`, `passwordLabel`, `passwordPlaceholder`, `forgotPassword` (link), `signIn` (button), `signupPrompt`, `signup` (link), `quote`, `quoteAuthor`, `imageAlt`

### Signup block (`Signup`)

**Purpose:** registration form (name, email, password) with an email confirmation code step.

Behavior:

- Uses `signup()` and `confirmSignup()` from `@/lib/auth`, then auto-logs in via `login()`.
- Two-step flow: registration → email confirmation code → auto sign-in.
- **Must be placed on a public page** (registration must be accessible without auth).
- Only shown/useful when `auth.allowSignup` is `true` in the app's auth settings.

Localizable copies (via `config.copies`):

`title`, `description`, `imageSrc`, `nameLabel`, `namePlaceholder`, `emailLabel`, `emailPlaceholder`, `passwordLabel`, `passwordPlaceholder`, `submitButton`, `confirmTitle`, `confirmDescription`, `confirmCodeLabel`, `confirmCodePlaceholder`, `confirmButton`, `loginPrompt`, `loginLink`, `quote`, `quoteAuthor`, `imageAlt`

### Placing auth blocks in `app.ts`

```ts
import { loginBlock, signupBlock } from "./block-types/index.js";

const loginPage = appBuilder.page({
  name: "Login",
  slug: "login",
  // public (default) — login page must be reachable without auth
  blocks: [
    appBuilder.block({ type: loginBlock }),
  ],
});

const signupPage = appBuilder.page({
  name: "Signup",
  slug: "signup",
  blocks: [
    appBuilder.block({ type: signupBlock }),
  ],
});
```

See `turbofy-apps` § "Authentication settings" for full `auth` configuration and page `visibility`.

---

## 5) Common gotchas

- **Blocks are self-contained**: A block is either a single `.tsx` file or a directory with an `index.tsx` entry point. Files within a multi-file block directory can import each other, but blocks cannot import files outside their own block boundary or from other blocks.
- **Always read locale from `props.locale`** — never hardcode it, derive it from the URL, or use `window.navigator.language`. **Never** create `getCurrentLocale()`, `getLocale()`, or similar helper functions.
- **Always read copies from `config.copies`** — the dsl-builders auto-inject the resolved dictionary into `config.copies`. Provide zod-validated defaults for every copy key. **Never** hardcode user-visible strings inline — every label, heading, placeholder, button text, and error message must come from `config.copies`. Access keys directly (`copies.title`), **never** create `t()` / `translate()` / `useTranslation()` wrapper functions.
- **Always use `navigate` from `@/navigation`** for programmatic navigation — never `window.location` or raw `<a>` tags. Read query parameters from `props.searchParams`.
- **Construct locale-prefixed paths from props** — use `props.locale` and `props.slug` (e.g. `/${locale}/products/${id}`). For server-resolved page links, use `$$std.batchLink(...)`.
- `Turbofy_app_push` automatically compiles and typechecks blocks — you cannot skip this step.
- Blocks without an `id` in the DSL file are treated as new (create). Blocks with an `id` that no longer appear are deleted.
- If a block dynamic field reads `$$self.someField`, ensure the fetch includes it via `dynamicArgs.fields`.
- Table IDs are workspace-specific; resolve them with `Turbofy_table_list`.
- For UI verification (editor/preview, screenshots/recordings), use the `agent-browser` skill.

---

## See also

- **`turbofy-platform`** — platform orientation, workspaces & environments, MCP tool surface, core rules, schema workflow, data-builder DSL.
- **`turbofy-apps`** — Apps CMS data model, `Turbofy_app_*` workflow, file layout, localization workflow, macros.
- **`turbofy-dynamic-fields`** — full `$$std` API reference, runtime model, reserved `dynamicArgs` keys, debugging checklist for dynamic-field code.
