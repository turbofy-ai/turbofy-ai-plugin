---
name: turbofy-apps
description: "Use when building or editing a Turbofy website/app — creating an app, listing/inspecting apps, adding/removing/reordering pages and sections, site settings, private pages or authentication, translating content, or fixing URLs/slugs. Triggers: 'build my app', 'add a page', 'update the homepage', 'change the layout', 'add a header to every page', 'translate to German', 'fix this URL', 'reorder sections', 'make this page private', 'add login', 'require authentication', 'edit my Turbofy app'. Load BEFORE calling any app_* MCP tool. Covers the app JSON manifest and app_get → edit → app_push. Do NOT use for database schema changes or writing React UI code — load turbofy-platform or turbofy-blocks instead."
disable-model-invocation: false
---

# Turbofy Apps

Building and modifying Turbofy apps via the HTTP MCP — JSON manifests and the `app_get` → edit → `app_push` loop.

For workspaces/schema/data rules see `turbofy-platform`. For React block sources see `turbofy-blocks`. For `@dynamic_field` JS see `turbofy-dynamic-fields`.

## When to load this skill

Load when the user wants to **change or inspect site structure / content placement** — e.g. "add a contact page", "put the hero above the footer", "translate the site to French".

---

## 1) What are Apps?

Apps assemble web UIs from a data schema and building blocks.

- **In-dashboard** — default. Runs inside Turbofy with direct workspace data access.
- **Published / standalone** — public site. Supports **private (auth-gated) pages** via end-user authentication.

### Data model

Core Apps CMS tables (`App`, `Page`, `BuildingBlockType`, `BuildingBlock`, `Localization`, …) are **system types** — not in the workspace schema. Use stable MCP `ofType` strings when calling `data_*` (see § 2). Prefer the app manifest + `app_push` for structure edits.

In dynamic-field code, workspace tables are referenced **by table id** (`$$std.getRecord(<tableId>, …)`). Resolve IDs via `workspace_get` → `schema.types[].id`. For system CMS tables in dynamic fields, use `"cmspage"`, `"cmslocalization"`, etc.

#### App

Top-level container. Key fields: `id`, `name`, `settings` (includes `i18n`, optional `auth`). Has many Pages.

#### Page

A route. Key fields:

- `id`, `name`
- `slugConfig` (`Json`) — `{ "<lang>": { "slug": "/path/[param]" }, "paramsCollectionMap": { "<param>": { "ofType": "<table-ID>", "slugField": "<field>" } } }`. For dynamic pages `paramsCollectionMap` is required; **`ofType` must be the table ID**.
- `localizedConfig` (optional string on the manifest) — `@dynamic_field` JS for page metadata.
- Optional `visibility` — `"public"` (default) | `"authenticated"` | `"group"` | `"user"`. `"user"` is the common “requires login” value.
- `blocks` — ordered instances (`id`, `blockTypeId`, `position`, optional `config` / `dynamicData` / `localizations`)

#### BuildingBlockType

Category of section. On the manifest:

- `id`, `name` (required)
- `defaultConfig` / `defaultDynamicData` — server-side JS strings; see `turbofy-dynamic-fields`
- `localizations` — `{ [lang]: { [copyKey]: string } }` (required on push)
- `sourceCodeUrl` / `compiledCodeUrl` — published artifacts (updated by `block_type_push`, not `app_push`)

#### BuildingBlock

Instance on a page: `id`, `blockTypeId`, `position` (multiples of 10; optional hierarchy like `20/10`), optional per-instance `config` / `dynamicData` / `localizations`.

#### Localization IDs (runtime)

| Pattern | Purpose |
|---|---|
| `{lang}_blocktype_{blockTypeId}` | Block type copies → `config.copies` |
| `{lang}_block_{blockId}` | Per-instance overrides |
| `{lang}_page_{pageId}` | Page-level content |

`app_push` materializes dictionaries from the manifest’s `localizations` fields — edit the manifest, don’t hand-write localization rows for structure work.

---

## 2) MCP tools

Pass **`orgId` + `workspaceId`** on every call (see `turbofy-platform`).

| Tool | Purpose |
|---|---|
| `data_list` (`ofType: "cmsapp"`) | List apps in the workspace (ids + names + settings) |
| `app_get` | Full JSON manifest (edit this, then push) |
| `app_init` | Create app + default Home page; returns the new manifest |
| `app_push` | Validate + reconcile pages / blocks / block types / copies from a modified manifest. **`dryRun` defaults to `true`**. Does **not** compile React sources — use `block_type_push` for that. |

There is **no** local app directory and **no** `app_pull`.

### System CMS `ofType` map (for occasional `data_*`)

| Entity | `ofType` |
|---|---|
| App | `"cmsapp"` |
| Page | `"cmspage"` |
| BuildingBlockType | `"cmsbuildingblocktype"` |
| BuildingBlock | `"cmsbuildingblock"` |
| Localization | `"cmslocalization"` |
| FileDocument / Image | `"filedocument"` |
| SlugMapping | `"slugmapping"` |

---

## 3) Primary workflow: `app_get` → edit → `app_push`

0. **List** (if needed) — `data_list` with `ofType: "cmsapp"` → pick `id` as `appId`.
1. **Get** — `app_get` → keep the full returned object as your working manifest (`workspace`, `app`, `pages`, `blockTypes`, `counts`).
2. **Edit** in memory:
   - Add/remove/reorder pages and blocks
   - Change `slugConfig`, `settings.i18n`, `settings.auth`, visibility
   - Add/update block types (`name`, `defaultConfig`, `defaultDynamicData`, `localizations`)
   - Per-block instance overrides (`config`, `dynamicData`, `localizations`)
3. **Dry-run** — `app_push` with `{ orgId, workspaceId, appId, manifest, dryRun: true }` (default). Review `summary` / `operations` / `validationIssues`.
4. **Apply** — same call with `dryRun: false`.
5. **Confirm** — `app_get` again.

### Critical `app_push` rules

- **Start from a full `app_get`.** The reconciler diffs the manifest against remote state. Entities **omitted** from `pages` / `blocks` / `blockTypes` are **deleted**.
- `appId` must match `manifest.app.id`.
- Validates slug type refs, block→blockType refs, and dynamic-field JS; applies **copies guards** on `defaultConfig` / instance `config` (you supply the user JS; the platform wraps/merges `copies`).
- Prefer **unwrapped** user `defaultConfig` when editing, e.g. `return ({ signalTableId: "wGluy5" });` — do not hand-maintain the `/* @graphapi-io/wrap:bt-copies-* */` wrapper; push re-applies it.
- **React source compile/upload is separate:** after adding a new block type (or changing `index.tsx`), use `block_type_open` → `block_type_fs_*` → `block_type_check` → `block_type_push` (`turbofy-blocks`).
- Always dry-run before apply.

### Create an app

1. `app_init` with `{ orgId, workspaceId, name }`
2. Continue with `app_get` / `app_push` / `block_type_*` as needed

### Schema for the app

Schema is workspace-scoped: `workspace_get` → edit JSON → `workspace_schema_push` (`turbofy-platform`). Not part of the app manifest.

### Localization

- App locales: `app.settings.i18n` = `{ locales: string[], default: string }`.
- Block type copies: `blockTypes[].localizations` — keep a key for every locale in `i18n.locales` (empty string placeholders OK).
- Instance overrides: `pages[].blocks[].localizations` only when needed.
- At runtime, `config.copies` is the merged dictionary (active locale over `i18n.default`; instance over type). React must read strings from `config.copies` only.

### Page visibility & auth

Set on the manifest (then `app_push`):

- Page `visibility`: `"public"` | `"authenticated"` | `"group"` | `"user"`.
- App `settings.auth`: `{ enabled?, allowSignup?, signupFields?, redirectPageId?, loginPageId? }`.

Login/signup pages must remain publicly reachable. Login/Signup system blocks call `/api/auth/login` and `/api/auth/signup`.

---

## 4) Parallel build roles

| Role | Owns | Must not |
|---|---|---|
| Orchestrator | `orgId`/`workspaceId`/`appId`; `app_init`/`app_get`/`app_push`; schema via `workspace_*`; wiring pages/blocks | edit every block’s React sources itself when delegating |
| `turbofy-block-builder` | one block type’s remote session + its manifest slice (config/copies) when assigned | other blocks; final app-wide push unless told |
| `turbofy-schema-builder` | schema JSON via `workspace_get` / `workspace_schema_push` | pages/blocks |

Typical multi-section build: orchestrator `app_get` → schema if needed → parallel block builders (React + intended config/copies) → orchestrator merges into one manifest → `app_push` dry-run/apply → each builder (or orchestrator) runs `block_type_push` for sources.

---

## See also

- **`turbofy-platform`** — discovery, schema JSON, `data_*`, CMS ofType table.
- **`turbofy-blocks`** — `block_type_*` session + React rules.
- **`turbofy-dynamic-fields`** — `$$std` / `$$args` for `defaultConfig` / `defaultDynamicData`.
