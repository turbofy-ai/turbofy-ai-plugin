---
name: turbofy-apps
description: "Use when building or editing a Turbofy website/app — creating an app, syncing it locally, adding/removing/reordering pages, placing or rearranging sections on a page, changing site settings, adding private pages or authentication, translating content, fixing URLs/slugs, or pushing changes live. Triggers: 'build my app', 'add a page', 'update the homepage', 'change the layout', 'add a header to every page', 'translate to German', 'fix this URL', 'reorder sections', 'make this page private', 'add login', 'require authentication', 'edit my Turbofy app' — even when the user doesn't mention files or tools. Load BEFORE calling any Turbofy_app_* MCP tool. Covers app.ts, pages, page visibility, auth settings, section placement, localization, and pull → edit → push. Do NOT use for database schema changes or writing React UI code — load turbofy-platform or turbofy-blocks instead."
disable-model-invocation: false
---

# Turbofy Apps

This skill covers building and modifying Turbofy apps — the Apps CMS data model, the `Turbofy_app_*` workflow, the app file layout, localization, and block-instance editing.

For platform-level concerns (workspaces, environments, org/workspace discovery, the data schema, the `Turbofy_workspace_*` workflow, the data-builder DSL, and core MCP rules like "pass `workspaceId` explicitly" and "system CMS vs workspace IDs"), see `turbofy-platform`. The companion skills `turbofy-blocks` (writing block React components) and `turbofy-dynamic-fields` (server-side dynamic-field JavaScript) cover those areas in detail.

## When to load this skill

Load when the user wants to **change their site structure or content placement** — e.g. "add a contact page", "put the hero above the footer", "translate the site to French". Do not wait for them to mention `app.ts`, MCP tools, or block-instance IDs.

---

## 1) What are Apps?

Apps are how Turbofy lets users assemble web applications from a data schema and a set of building blocks. They can run in two modes:

- **In-dashboard** — the default. Apps execute inside the Turbofy dashboard with direct access to the workspace's data (including non-public tables, gated by the dashboard's auth). No publishing or deployment step.
- **Published / standalone** — the same app served as a production-ready public site. Published apps support **private (auth-gated) pages** via end-user authentication (see § "Page visibility" and § "Authentication settings" below). Public tables are still required for data access, but individual pages can be restricted to authenticated users.

The runtime, file layout, and `Turbofy_app_*` workflow are identical for both modes — the difference is the table-visibility constraint that publishing imposes.

### Data model

This section describes the core tables, their purpose, key fields, and how they relate to each other.

Tables are referenced **by name** in Turbofy MCP (e.g. `Page`, `Localization`).

The **source of truth for the Apps CMS app definition** is the decompiled workspace files (`app.ts` and `block-types/` under the app directory after `Turbofy_app_pull`). The MCP writes them to `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`. Use them to understand pages, blocks, block types, and macros.

Important: the core Apps CMS tables (`App`, `Page`, `BuildingBlockType`, `BuildingBlock`, `Localization`, `Image`) are **system types** and are **not part of the workspace schema**. Their IDs are stable — do not look them up via `Turbofy_table_list`. Reference them via `CmsOfTypeEnum.X` inside the DSL (`schema.ts`, `app.ts`, `record.ts`) or via the literal string value (`"cmspage"`, `"cmsbuildingblock"`, …) when calling MCP tools directly. See `turbofy-platform` § "Core MCP rules" for the full mapping table and the legacy `@cms_*` migration note.

In dynamic-field code, tables are often referenced **by table id** (e.g. `$$std.getRecord(<tableId>, ...)`). For workspace schema tables, resolve IDs via `Turbofy_table_list` or read them from `schema.ts`. For system CMS tables, interpolate `CmsOfTypeEnum.X` into the dynamic-field template literal (the runtime sees the resolved string).

#### App

An **App** is the top-level container for a set of pages.

Key fields:

- `id`
- `name`
- `createdAt`, `updatedAt`

An app **has many Pages**.

#### Page

A **Page** represents a route in the webapp (e.g. homepage, product detail, cart).

Key fields:

- `id`
- `name`
- `slugConfig` (`Json`) — config used to resolve/construct route slugs. Format: `{ "<lang>": { "slug": "/path/[param]" }, "paramsCollectionMap": { "<param>": { "ofType": "<table-ID>", "slugField": "<field>" } } }`. For dynamic pages `paramsCollectionMap` is **required** — it maps `[param]` placeholders to collections. **`ofType` must be the table ID (not the table name)**; use `Turbofy_table_list` to find it. `slugField` is optional — when set, the named record field (e.g. `"name"`) is used as a human-readable slug segment.
- `localizedConfig` (`@dynamic_field`, typed as `String`) — returns page metadata derived from `slugConfig` and dynamic args. In this workspace it computes:
  - `canonicalPath` — the canonical URL path (with slugs resolved)
  - `params` — resolved entity IDs for dynamic segments (e.g. `{ articleId: "1234-..." }`). These are passed as `$$args.params` to building block dynamic fields.
  - `notFound` — `true` if any referenced entity doesn't exist
- `createdAt`, `updatedAt`

A page **belongs to an App** and **has many BuildingBlocks**.

#### BuildingBlockType

A **BuildingBlockType** defines a category of building block (e.g. "Navigation", "ProductTable", "Footer"). There are multiple instances of the same block type across the application.

Key fields:

- `id`
- `name`
- `defaultConfig` (`@dynamic_field`, typed as `String`) — default configuration for blocks of this type. Use it for static server-side data such as copies, links, and layout settings. It should depend on `$$args.lang`, not on route params.
- `defaultDynamicData` (`@dynamic_field`, typed as `String`) — default server-side data for blocks of this type. Use it for data that depends on `$$args.params` and `$$args.searchParams`. In SSR apps, this provides the initial data snapshot; later interactions should use hooks.
- `sourceCodeUrl` (`String`) — URL of the block type's source code. Points to a `.tsx` file (single-file block) or a `.source.zip` archive (multi-file block).
- `compiledCodeUrl` (`String`) — URL of the compiled/bundled block code.
- `dependencies` (`Json`) — block type's dependency metadata.
- `createdAt`, `updatedAt`

A building block type **has many BuildingBlocks**.

#### BuildingBlock

A **BuildingBlock** is a UI component instance placed on a page.

Key fields:

- `id`
- `name` (`@dynamic_field`) — derived from its `BuildingBlockType` (here it resolves to the type's `name`).
- `config` (`@dynamic_field`, typed as `String`) — configuration for rendering. In this workspace it resolves to the type's `defaultConfig`.
- `position` (`String`) — determines rendering order within the page. Convention: use multiples of 10 and (optionally) hierarchical positions like `20/10` to render a child block at position `10` inside the block at position `20`.
- `dynamicData` (`@dynamic_field`) — server-resolved data for the block. Use it for route-dependent initial data (for example data keyed by `$$args.params` or `$$args.searchParams`), especially in SSR apps.
- `createdAt`, `updatedAt`

Relationships:

- A building block **belongs to a Page** and **belongs to a BuildingBlockType**.

#### Localization

The **Localization** table stores translated strings for the CMS.

Key fields:

- `id` — follows `${lang}_${kind}_${entityId}` (see naming patterns below).
- `dictionary` (`Json`) — translation key/value map. Used by `$$std.translate(...)` and `$$std.batchTranslate(...)`.
- `content` (`String`) — freeform content (project-specific usage).
- `createdAt`, `updatedAt`

**Naming patterns** (canonical):

| Pattern                          | Example IDs             | Purpose                                                                             |
| -------------------------------- | ----------------------- | ----------------------------------------------------------------------------------- |
| `{lang}_blocktype_{blockTypeId}` | `en_blocktype_2vaEf74y` | Block type copies — written by `Turbofy_app_push` from `record.ts` `localizations`. |
| `{lang}_block_{blockId}`         | `en_block_ghi789`       | Per-instance block overrides.                                                       |
| `{lang}_page_{pageId}`           | `en_page_def456`        | Page-level localized content.                                                       |

Note: `appId` is not included in the pattern since localizations already belong to an app via the `appId` field.

`{lang}_blocktype_{blockTypeId}` records are managed by the workspace — never edit them directly. They are produced by `Turbofy_app_push` from each block type's `record.ts` `localizations` field, and surfaced back into `record.ts` by `Turbofy_app_pull`.

**Legacy IDs**: older workspaces may contain Localization rows with ad-hoc IDs (`en_product`, `de_footer`, `en_navigation`, `en_table`, `de_search`, `en_manufacturer_2vaEf74y...`). They predate the canonical scheme. Inspect with `Turbofy_data_list` on the Localization table (`ofType: "cmslocalization"`) and migrate to one of the patterns above when touching them.

#### SlugMapping

The **SlugMapping** table maps human-readable URL slugs to entity IDs, scoped by type and language.

Key fields:

- `id` — composite key: `{appId}#{typeId}#{lang}#{slug}` (e.g. `1234-1234-1234-1234#gbqlrk#en#what-to-eat`). Mutable — update to change the slug without deleting the record.
- `key` (`String`, `@sortby`) — composite sort key: `{appId}#{typeId}#{lang}#{entityId}#{timestamp}` (e.g. `1234-1234-1234-1234#gbqlrk#en#1234-abcd#2026-03-20T17:29:51.407Z`). Stable — one record per entity per language.

Usage:

- **Slug → entity**: query by sort key `eq` to find the entity ID
- **Entity → slug**: direct `Turbofy_data_get` by ID to read the slug from `key`
- Managed via the **Slug Management** UI in the page editor

The `pageLocalizedConfigCode()` macro resolves slugs automatically. The engine later passes resolved entity IDs as `$$args.params` to building blocks.

#### Image

The **Image** table stores image assets.

Key fields:

- `id` (`ID`) — image identifier.
- `url` (`String`) — image URL.
- `width`, `height` (`Integer`) — dimensions.
- `createdAt`, `updatedAt`

---

## 2) MCP tools

The full Turbofy MCP tool surface and the core MCP rules (`workspaceId` is explicit, read-before-write, system-CMS-vs-workspace IDs, pagination limits) live in `turbofy-platform` § 3–4. This section covers only the **app-specific** tools and the system CMS `ofType` map.

### App-specific tools

- `Turbofy_app_init` — create app + default Home page remotely AND scaffold the local workspace in one step
- `Turbofy_app_pull` — pull app into `~/.turbofy/workspaces/<env>/<workspaceId>/apps/<appId>/`
- `Turbofy_app_push` — compile schema + blocks, upload, reconcile from the unified workspace (supports `dryRun: true`)

Always call `Turbofy_app_push` with `dryRun: true` first to preview the diff.

### Reading Apps CMS entities via `Turbofy_data_*`

When you need pages/block types/building blocks/localizations/images, use the generic data tools (`Turbofy_data_list`, `Turbofy_data_get`, etc. — see `turbofy-platform`). For the `ofType` argument, pass the **literal string value** below — `CmsOfTypeEnum.X` only works inside the DSL (TypeScript), not in MCP tool calls.

| Entity               | `ofType` (MCP)           |
| -------------------- | ------------------------ |
| App                  | `"cmsapp"`               |
| Page                 | `"cmspage"`              |
| BuildingBlockType    | `"cmsbuildingblocktype"` |
| BuildingBlock        | `"cmsbuildingblock"`     |
| Localization         | `"cmslocalization"`      |
| Image (FileDocument) | `"filedocument"`         |
| SlugMapping          | `"slugmapping"`          |

See `turbofy-platform` § 4 for the full mapping table (including the DSL enum equivalents and the less common entries `EntityLocalization`, `Api`, `Code`).

Apps, pages, block types, and block instances are normally managed via the workspace files + `Turbofy_app_push` — the generic data tools are for one-off reads/edits and for things the workspace doesn't own (e.g. arbitrary Localization rows, SlugMapping entries, FileDocument records).

---

## 3) App workflow

### Primary workflow: `Turbofy_app_init` → `Turbofy_app_pull` / edit / `Turbofy_app_push`

**For a brand new app**, call `Turbofy_app_init`. This creates the app + a default Home page remotely and scaffolds the local workspace in one step.

The recommended way to modify an existing app's schema, pages, block types, block instances, **and block type source code** is the unified workspace:

1. **Pull** — `Turbofy_app_pull` writes everything to `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`:
   - `schema.ts` — GraphQL data schema (DSL)
   - `app.ts` — schema re-export + page definitions + blocks (DSL); imports block types from `./block-types/index.js`
   - `app.base.json` / `schema.base.json` — serialized baseline used by push for diffing; do not edit by hand
   - `package.json` + `tsconfig.json` — scaffolded Node/TypeScript config so the directory is self-contained
   - `block-types/index.ts` — barrel re-exporting every block type variable
   - `block-types/{Name}/record.ts` — one **record** per block type (the `appBuilder.blockType({...})` call)
   - `block-types/{Name}/index.tsx` (+ siblings) — optional runtime React code for the block type, pulled from the `.source.zip` archive when present
2. **Edit** — modify files directly (add/remove/update pages, block types, blocks, schema, block source code).
3. **Push** — `Turbofy_app_push` compiles schema, compiles+uploads each block type that has source code on disk, and reconciles the app state.

Call `Turbofy_app_push` with `dryRun: true` to preview changes before applying them.

Block source location is **convention-based**: push looks for `block-types/{Name}/index.{tsx,ts}`. If absent, the block type is treated as sourceless (valid — useful for content-only blocks where the runtime is provided elsewhere). When present, push compiles + uploads it and writes the resulting URLs back to the CMS automatically.

The reconciler diffs the edited file against the current API state and executes only the necessary operations.

### File layout

After `Turbofy_app_pull` or `Turbofy_app_init`, files live under `~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/`:

```
~/.turbofy/workspaces/<environment>/<workspaceId>/apps/<appId>/
  schema.ts                 # GraphQL data schema (DSL)
  app.ts                    # schema re-export + pages + blocks (DSL); imports from ./block-types
  app.base.json             # serialized baseline app state (do not edit)
  schema.base.json          # serialized baseline schema declaration (do not edit)
  package.json              # scaffolded so the directory is self-contained
  tsconfig.json
  block-types/
    index.ts                # barrel: re-exports every block type variable
    Navigation/
      record.ts             # one row of the BuildingBlockType table, in code
      index.tsx             # runtime React code (optional)
    ProductTable/
      record.ts
      index.tsx
    Dashboard/              # multi-file block type
      record.ts
      index.tsx             # entry point
      helpers.ts
      chart-utils.ts
    Footer/
      record.ts             # sourceless block type — record only, no index.tsx
```

`<environment>` is the active MCP environment (`alpha` or `prod`) — the `turbofy` MCP uses `prod`, `turbofy-alpha` uses `alpha`. `<workspaceId>` and `<appId>` are the IDs reported by `Turbofy_list_workspaces` and `Turbofy_app_pull` / `Turbofy_app_init`.

`record.ts` is a per-block-type manifest that maps 1:1 to a row in the `BuildingBlockType` table. It imports `appBuilder` and exports a single `const`:

```ts
// block-types/Navigation/record.ts
import { appBuilder } from "@graphapi-io/dsl-builders";

export const navigationBlock = appBuilder.blockType({
  id: "...",
  name: "Navigation",
  defaultConfig: `...`,
});
```

`record.ts` does **not** belong to the runtime bundle. The block's `index.tsx` must not import it — push and the iframe runtime keep them separate by design.

`block-types/index.ts` is a generated barrel that re-exports each block type from its sibling `record.ts`:

```ts
// block-types/index.ts
export { navigationBlock } from "./Navigation/record.js";
export { dashboardBlock } from "./Dashboard/record.js";
export { footerBlock } from "./Footer/record.js";
```

In `app.ts`, block types are imported from the barrel and used in page definitions:

```ts
export { schema } from "./schema.js";
import { appBuilder } from "@graphapi-io/dsl-builders";
import {
  navigationBlock,
  dashboardBlock,
  footerBlock,
} from "./block-types/index.js";

const home = appBuilder.page({ ... });

export const app = appBuilder.buildApp({
  name: "My App",
  i18n: { locales: ["en", "de"], default: "en" },
  pages: [home, productDetail],
  blockTypes: [navigationBlock, dashboardBlock, footerBlock],
});
```

### Page visibility

Each page can restrict access via the `visibility` option on `appBuilder.page()`:

```ts
const dashboard = appBuilder.page({
  name: "Dashboard",
  slug: "dashboard",
  visibility: "user",  // requires authentication
  blocks: [...],
});
```

Valid values (`PageVisibility` type):

| Value             | Meaning                                                                 |
| ----------------- | ----------------------------------------------------------------------- |
| `"public"`        | Default (can be omitted). Accessible to anyone.                         |
| `"authenticated"` | Requires a valid session (any logged-in user).                          |
| `"group"`         | Requires group membership (enforced at data layer via PBAC).            |
| `"user"`          | Requires the specific user match (enforced at data layer).              |

In practice, `"user"` is the most common value for "private page that requires login." When `visibility` is omitted the page is public.

The decompiler (`Turbofy_app_pull`) emits `visibility` on pages that aren't public, so round-tripping works correctly.

### Authentication settings

Enable end-user authentication on a published app via the `auth` option on `buildApp()`:

```ts
export const app = appBuilder.buildApp({
  name: "My App",
  i18n: { locales: ["en"], default: "en" },
  auth: {
    enabled: true,
    allowSignup: true,
    signupFields: [
      { name: "full_name", label: "Full Name" },
    ],
    redirectPageId: "<page-id>",  // where to land after login
    loginPageId: "<page-id>",     // custom login page (must contain Login block)
  },
  pages: [...],
});
```

`IAppAuthSettings` fields:

| Field             | Type                                                                                             | Description                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| `enabled`         | `boolean?`                                                                                       | Whether end-user authentication is enabled.                                      |
| `allowSignup`     | `boolean?`                                                                                       | Whether visitors can self-register.                                              |
| `signupFields`    | `{ name: string; label?: string; type?: "text" \| "number" \| "email"; required?: boolean }[]?`  | Extra profile fields beyond email + password.                                    |
| `redirectPageId`  | `string?`                                                                                        | Page to redirect to after login (absent = locale home page).                     |
| `loginPageId`     | `string?`                                                                                        | Page serving as login screen (absent = built-in `/login` page).                  |

### How private pages work at runtime

When `auth.enabled` is `true`:

1. Pages with `visibility` other than `"public"` are protected by a proxy that checks for a valid JWT session cookie.
2. Unauthenticated users hitting a private page are redirected to the login page (either the built-in one or the custom `loginPageId`).
3. After login, users are redirected back to the page they tried to access (deep-linking), or to `redirectPageId` if they navigated to login directly.
4. The **Login** and **Signup** system blocks handle the auth forms — they call `/api/auth/login` and `/api/auth/signup` endpoints backed by Cognito. See `turbofy-blocks` for details on these system blocks.

#### Usage pattern for a typical app with auth

```ts
import { appBuilder } from "@graphapi-io/dsl-builders";
import { loginBlock, signupBlock, dashboardBlock } from "./block-types/index.js";

const loginPage = appBuilder.page({
  name: "Login",
  slug: "login",
  // visibility is public (default) — login page must be reachable
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

const dashboard = appBuilder.page({
  name: "Dashboard",
  slug: "dashboard",
  visibility: "user",  // private — requires authentication
  blocks: [
    appBuilder.block({ type: dashboardBlock }),
  ],
});

export const app = appBuilder.buildApp({
  name: "My App",
  i18n: { locales: ["en"], default: "en" },
  auth: {
    enabled: true,
    allowSignup: true,
    loginPageId: loginPage.id,       // use our custom login page
    redirectPageId: dashboard.id,    // after login, go to dashboard
  },
  pages: [loginPage, signupPage, dashboard],
});
```

### Localization workflow

Localized copies for block types live **inside the workspace**, alongside the block type definition. The agent edits them directly in `record.ts`; `Turbofy_app_push` materializes them into Localization rows in the CMS, and `Turbofy_app_pull` reverses the round trip.

#### `i18n` on `buildApp` — the source of truth for supported locales

Every workspace declares its supported languages in `buildApp({ i18n: ... })`. This is the only place that lists locales for the app.

```ts
export const app = appBuilder.buildApp({
  name: "My App",
  i18n: {
    locales: ["en", "de"], // every locale the app supports
    default: "en", // root-locale redirect target AND copies-fallback locale
  },
  pages: [home, productDetail],
  blockTypes: [navigationBlock, dashboardBlock, footerBlock],
});
```

Rules enforced by `buildApp`:

- `i18n.locales` must be non-empty.
- `i18n.default` must appear in `i18n.locales`.
- Every block type's `localizations` keys must be a subset of `i18n.locales`. Adding a key for an unsupported locale is a hard error — the only way to localize into a new language is to first add it to `i18n.locales`.

Push reconciles `i18n` into `App.settings.i18n` on the server. Pull reads it back from there. There is no other place to declare locales.

> `i18n.default` plays two runtime roles: (1) it controls the root-locale redirect, and (2) it is the locale used as the **fallback dictionary** for auto-injected `copies` (see "Auto-injected `copies`" below). They are intentionally the same value — there is no separate "fallback" knob.

#### Per-block-type `localizations` in `record.ts`

Each `record.ts` carries a `localizations` field — a record keyed by language code, whose values are flat dictionaries of copies:

```ts
// block-types/Navigation/record.ts
import { appBuilder } from "@graphapi-io/dsl-builders";

export const navigationBlock = appBuilder.blockType({
  id: "blktype_2vaEf74y",
  name: "Navigation",
  defaultConfig: `({ columns: 3 });`,
  localizations: {
    en: { home: "Home", about: "About", contact: "Contact" },
    de: { home: "Startseite", about: "Über uns", contact: "Kontakt" },
  },
});
```

At push time each non-empty entry becomes a `${lang}_blocktype_${blockTypeId}` Localization row. At runtime the dictionary for the current locale is exposed as `config.copies` to the React component (see the `turbofy-blocks` skill).

> **Auto-injected `copies` (hard lock).** When you set `defaultConfig` on a block type, the builder wraps your code so the runtime always returns `{ ...yourResult, copies: <localized dictionary> }`. You **never need to spread `blockTypeCopiesCode()` yourself** — and even if you tried to set a `copies` key in your object, it would be overridden. Same for block-instance `config`: any user-supplied code is wrapped to always emit `copies` as the merge of the block type's translations and the block instance's `localizations` overrides. Only `defaultConfig` and `config` are wrapped — `defaultDynamicData` and `dynamicData` are left untouched. The wrap is reversible: `Turbofy_app_pull` strips it back to your original source.
>
> **Fallback-language merge.** The wrap also fetches the dictionary for the app's `i18n.default` locale (`"en"` when `i18n` is unset) and merges it **under** the active locale. Any key missing in the active locale falls back to the default locale's value, so a half-translated locale degrades gracefully instead of returning `undefined`.
>
> For block instances the merge order is **language tier first, scope tier second** — every active-locale dict beats every fallback-locale dict; within a tier, block-instance beats block-type. From lowest → highest priority: block-type default → block-instance default → block-type active → block-instance active. This guarantees that translating a block type into the active locale wins over a default-locale-only block-instance override (otherwise the German page would render the English instance copy instead of the translated type copy).
>
> The default locale is baked into the wrap at `Turbofy_app_push` time from `i18n.default`, so changing it requires a re-push.
>
> **Single-batch fetch.** All required dictionaries (2 for block-type `defaultConfig`, 4 for block-instance `config`) are pulled in a single `$$std.batchGetRecordsByInputs(...)` call. That means **one runtime interruption** per dynamic field regardless of how many locales are merged — and the runner internally dedupes by `(ofType, id)` so a request where active === fallback collapses to a single underlying read.

#### Empty per-locale dictionaries are intentional

`Turbofy_app_pull` always seeds **every** locale declared in `i18n.locales` into each block type's `localizations`, even when the CMS has no data for that locale. You will see entries like:

```ts
localizations: {
  en: { joinAs: "You are", roomLabel: "Room" },
  de: { joinAs: "", roomLabel: "" },
}
```

The empty `de` strings are a contract: this app supports German, this block type has not been translated yet, and these are the keys that need translations. **Do not delete empty per-locale entries** — that would silently drop the language from the contract for that block type. Instead, fill in the translations.

If a workspace pull surfaces a locale you didn't expect (e.g. `fr: { ... }` while `i18n.locales` is `["en", "de"]`), it means a stray Localization row exists in the CMS. Pull preserves it so it doesn't get silently dropped, but `buildApp` will reject the push until you either add `"fr"` to `i18n.locales` or remove the locale from the block type.

#### Adding a new language

1. Add the locale code to `i18n.locales` in `buildApp` (and update `default` if needed).
2. Run `Turbofy_app_push`. This writes the updated `App.settings.i18n` to the server.
3. Run `Turbofy_app_pull`. Every block type's `record.ts` now has an empty dictionary for the new locale, ready to fill in.
4. Translate, then `Turbofy_app_push` again to materialize the localizations.

#### Per-block instance overrides on `appBuilder.block(...)`

Individual block instances can override the type's copies by setting `localizations` on the `block(...)` call inside a page. The shape mirrors block-type `localizations` — a record keyed by language code with flat dictionaries — but with two important differences:

- **It is optional.** Most blocks inherit copies from their type and leave `localizations` unset. Add it only when this particular instance on this particular page needs different copy than the default.
- **`Turbofy_app_pull` does not auto-seed empty dictionaries here.** Pull writes `localizations` only when at least one CMS override actually exists. This keeps page files clean — overrides stand out instead of drowning in empty placeholders.

```ts
// app.ts
const home = appBuilder.page({
  name: "Home",
  blocks: [
    appBuilder.block({
      type: navigationBlock,
      localizations: {
        en: { home: "Home" },
        de: { home: "Zuhause" }, // override only the keys that differ
      },
    }),
    // No `localizations` — inherits everything from `navigationBlock.localizations`.
    appBuilder.block({ type: footerBlock }),
  ],
});
```

Push materializes each non-empty dictionary into a `${lang}_block_${blockId}` Localization row. Validation in `buildApp` enforces the same locale subset rule as block types: every key under `localizations` must be in `i18n.locales`.

To remove an override, delete the `localizations` field (or the specific locale entry) and run `Turbofy_app_push`. The reconciler does not delete dropped Localization rows automatically — use `Turbofy_data_delete` on the Localization row (`ofType: "cmslocalization"`) if you need the CMS row gone, otherwise the empty/stale row is harmless.

#### When to use which localization mechanism

- **Block type copies** — strings owned by a block type's React component (button labels, headings, error messages). Live in `record.ts` `localizations`. The builder auto-injects them into `config.copies` at runtime regardless of what you put in `defaultConfig`. **This is where almost all UI strings belong.**
- **Per-block instance overrides** — when one specific block instance on one specific page needs different copies than the block type default. Set `localizations` on the corresponding `appBuilder.block(...)` call in `app.ts` (writes `${lang}_block_${blockId}` rows on push). Or, for ad-hoc one-off edits, use `Turbofy_data_create` / `Turbofy_data_update` with `ofType: "cmslocalization"`.
- **Page-level localized content** (`{lang}_page_{pageId}`) — page metadata (title, description) consumed by `pageLocalizedConfigCode()`. Manage via `Turbofy_data_*` with `ofType: "cmslocalization"`.
- **Schema/data localizations** (e.g. translated entity descriptions) — store on the entity itself (the original "manufacturer description in `de`" use case). Use ad-hoc dictionaries the entity's dynamic fields can read via `$$std.translate(...)`.

### Schema changes inside an app

When app work requires schema changes (new tables, fields, enums), edit `schema.ts` at the workspace root — `Turbofy_app_push` reconciles schema changes as a side-effect of the app push, so you don't need a separate `Turbofy_workspace_push`. For pure schema work outside an app context, use the `Turbofy_workspace_pull` → edit → `Turbofy_workspace_push` loop directly. The schema DSL itself (builder calls, field types, parents, critical rules) is documented in `turbofy-platform` § 5–6.

### Macros

The `app.ts` file imports macros from `@graphapi-io/declarations`:

```ts
import { macros } from "@graphapi-io/declarations";
const {
  blockConfigCode,
  blockDynamicDataCode,
  blockNameCode,
  blockTypeCopiesCode,
  pageLocalizedConfigCode,
} = macros;
```

| Macro                                     | Used in                     | Description                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `blockTypeCopiesCode()`                   | _internal_                  | Resolves localized copies for the block type from `${lang}_blocktype_${$$self.id}` Localization records. Used as the schema default for `BuildingBlockType.defaultConfig` (when no user code is provided). **Do not spread it into `defaultConfig` yourself** — the dsl-builders auto-inject `copies` into every user-supplied `defaultConfig` (hard lock; see "Auto-injected `copies`"). |
| `blockConfigCode(BlockTypeTable.id)`      | _internal_                  | Inherits the parent block type's `defaultConfig` and merges block-instance copies. Used as the schema default for `BuildingBlock.config`. User-supplied `config` is wrapped automatically (the wrap merges block-type + block-instance translations into `copies`); you do not need to call this macro from `app.ts`.                                                                     |
| `blockDynamicDataCode(BlockTypeTable.id)` | `BuildingBlock.dynamicData` | Inherits the parent block type's `defaultDynamicData`. Used in the schema definition and in per-instance `dynamicData` overrides to extend parent data.                                                                                                                                                                                                                                   |
| `blockNameCode(BlockTypeTable.id)`        | `BuildingBlock.name`        | Derives the block instance name from its block type. Used in the schema definition.                                                                                                                                                                                                                                                                                                       |
| `pageLocalizedConfigCode()`               | `Page.localizedConfig`      | Computes localized page metadata (canonical path, notFound). Used in the schema definition.                                                                                                                                                                                                                                                                                               |

---

## 4) Block instances

### Editing the DSL file

- **Add a block**: add a `block()` call inside a page's `blocks` array.
- **Remove a block**: delete the `block()` call.
- **Change position**: reorder blocks in the array (positions auto-compute) or set an explicit `position`.
- **Change config/dynamicData**: edit the `config` or `dynamicData` string on the block.

Example — adding a block to a page:

```ts
const home = appBuilder.page({
  id: "existing-page-id",
  name: "Home",
  blocks: [
    appBuilder.block({ id: "existing-block-id", type: hero }),
    appBuilder.block({ type: card, config: "({ title: 'New card' })" }), // NEW — no id means create
  ],
});
```

Blocks without an `id` in the DSL file are treated as new (create). Blocks with an `id` that no longer appear are deleted.

### Adjusting localization

Most localization edits happen in the workspace, not via MCP data tools:

- **Block type copies** — edit `block-types/{Name}/record.ts` `localizations`, then `Turbofy_app_push`. See "Localization workflow" above.
- **Per-block instance overrides** — edit `localizations` on the relevant `appBuilder.block(...)` call in `app.ts`, then `Turbofy_app_push`. See "Per-block instance overrides" above.
- **App-level supported locales** — edit `i18n.locales` in `buildApp`, then `Turbofy_app_push`.

Use the generic data tools below for ad-hoc edits or things the workspace doesn't own:

- `Turbofy_data_list` with `ofType: "cmslocalization"` — list localizations (filter the returned items client-side by id prefix, e.g. starting with `en_page_<pageId>`)
- `Turbofy_data_get` with `ofType: "cmslocalization"` — fetch a localization record
- `Turbofy_data_create` / `Turbofy_data_update` — create or update a localization record

These are the right tools for **page-level content** (`{lang}_page_{pageId}`) and one-off per-block instance edits when you don't want to round-trip through the workspace. Avoid using them to write `{lang}_blocktype_{blockTypeId}` rows directly — those are owned by the workspace and the next `Turbofy_app_pull` will reflect whatever is in `record.ts`. The same applies to `{lang}_block_{blockId}` rows when the block has `localizations` declared on the `block(...)` call in `app.ts`: pull will surface them and push will overwrite anything you wrote outside the workspace.

When a block is created via `Turbofy_app_push`, an empty `en_block_<blockId>` Localization row is created as a side-effect (so the editor UI has something to write to). It is harmless — pull skips empty dictionaries and the workspace remains the source of truth.

### Inspecting

Use the generic data tools to read CMS entities:

- Pages: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: "cmspage"`
- Block types: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: "cmsbuildingblocktype"`
- Block instances: `Turbofy_data_list` / `Turbofy_data_get` with `ofType: "cmsbuildingblock"`

For block instances on a specific page, list with `ofType: "cmsbuildingblock"` and filter client-side by `pageId`.

---

## 5) Building or modifying a multi-block app (orchestration)

Creating an app from scratch and modifying an existing one are the **same
workflow** with two switches: `Turbofy_app_init` vs `Turbofy_app_pull` at the
front, and a full build vs a delta in the middle. Once the schema and a per-block
contract are fixed, blocks are largely independent of each other, so the block
work **parallelizes** — but the pieces _within_ one block (dynamic field →
`record.ts` → `index.tsx`) are a dependency chain that must stay together in one
worker.

**Scale to the work.** For one or two blocks, just do it inline. Fan out only
when there are several independent blocks to build/modify — parallelism buys
wall-clock at the cost of N× context.

### Roles & file ownership

Parallel workers must write **disjoint** files. The shared files have exactly one
owner: the **orchestrator** (the main agent driving this skill).

| Owner                                   | Writes                                                                                       | Never writes                                                  |
| --------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Orchestrator (main agent)               | `app.ts`, `schema.ts`, the `block-types/index.ts` barrel; runs init/pull and the single push | individual `block-types/<Name>/` internals                    |
| `turbofy-block-builder` (one per block) | only its own `block-types/<Name>/` (`record.ts`, `index.tsx`)                                | `app.ts`, the barrel, `schema.ts`, other blocks; never pushes |
| `turbofy-schema-builder` (optional)     | `schema.ts`                                                                                  | `app.ts`, `block-types/`                                      |

> The fan-out happens **at the orchestrator (main-agent) level** — a subagent
> can't spawn further subagents. If custom plugin agents aren't available in your
> environment, brief a general subagent per block with the same lane rules and
> the `turbofy-blocks` + `turbofy-dynamic-fields` skills loaded.

### The unified flow

1. **Enter** — `Turbofy_app_init` (from scratch) or `Turbofy_app_pull`
   (existing). Confirm the app directory exists on disk.
2. **Schema** — define/modify `schema.ts` (inline, or delegate to
   `turbofy-schema-builder`). You usually do **not** push schema here — the final
   `Turbofy_app_push` reconciles it. Push early only if a builder needs live
   data. Capture the resulting **data contract** (table names, IDs, fields).
3. **Define block contracts** — before fanning out, spec each block slot: Name,
   `config` knobs + copy keys + locales, the `dynamicData` shape and which schema
   tables/fields it reads, sourceless-or-React, and target page/position. This is
   the sequential, high-leverage step that stops parallel builders from drifting.
   For a **delta**, label each block add / modify / delete.
4. **Build blocks in parallel** — delegate one `turbofy-block-builder` per
   **added or modified** block, each with its contract. **Deletes are not
   workers** — remove the `block-types/<Name>/` dir and unwire it yourself.
   Collect each builder's returned barrel line, import name, instance config,
   assumed schema deltas, and copy keys.
5. **Assemble** (orchestrator, sequential — the fan-in) — reconcile any schema
   deltas the builders surfaced; update the `block-types/index.ts` barrel; wire
   imports, `appBuilder.block(...)` placement, positions, and pages into
   `app.ts`. Keep `i18n.locales` a superset of every locale the builders used.
6. **Push once** — `Turbofy_app_push` with `dryRun: true`, fix any cross-block
   errors (re-delegating to the relevant builder if a single block is at fault),
   then the real push. This is a **hard barrier** — never run parallel or
   per-block pushes; the reconciler operates on the whole app from disk.

### Why one block can't be split, but many blocks can

Within a block, the React component depends on the block-type DSL, which often
depends on the dynamic-field code — so that vertical slice lives in **one**
`turbofy-block-builder` and runs in order. Across blocks there's no such
dependency once the schema + contracts are fixed, which is exactly the axis you
parallelize on.

---

## See also

- **`turbofy-platform`** — platform orientation, workspaces & environments, org/workspace discovery, the full MCP tool surface + core rules, the workspace schema workflow (`Turbofy_workspace_*`), and the data-builder DSL.
- **`turbofy-blocks`** — writing block-type runtime React components (`block-types/<Name>/index.tsx`): props, copies, UI/UX rules, client-side hooks.
- **`turbofy-dynamic-fields`** — server-side dynamic-field code (`defaultConfig`, `defaultDynamicData`, page `localizedConfig`): the `$$std` API, `$$args`, `$$self`, reserved `dynamicArgs` keys.
