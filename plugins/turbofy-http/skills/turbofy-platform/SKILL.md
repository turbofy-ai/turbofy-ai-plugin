---
name: turbofy-platform
description: "Use as the first skill for any Turbofy work — especially when the user asks 'what is Turbofy', 'list my workspaces', 'set up my database', 'add a table or field', 'change my data model', 'add/edit/delete records', 'upload a file', or 'connect to Turbofy'. Also load at the start of a fresh session before calling any Turbofy MCP tool, to pick the right follow-on skill. Covers org/workspace discovery, schema JSON editing, record CRUD, file upload, and core MCP rules. Do NOT use for page layout, placing sections on pages, or building UI components — load turbofy-apps, turbofy-blocks, or turbofy-dynamic-fields instead."
disable-model-invocation: false
---

# Turbofy Platform

Orientation skill for the Turbofy platform. Load it first in a fresh session, for workspace/schema work, or when choosing which deeper skill to pull in. Companions: `turbofy-apps`, `turbofy-blocks`, `turbofy-dynamic-fields`, `turbofy-flows`.

## When to load this skill

| User wants… | Load |
|---|---|
| Pages, layout, homepage, translate the site | `turbofy-apps` |
| Build/style a section (navbar, hero, grid) | `turbofy-blocks` |
| Server data blank, fetch records, URL-based titles | `turbofy-dynamic-fields` |
| Automations / flows | `turbofy-flows` |
| Chatbot, AI assistant, conversational feature | `turbofy-chatbot` |
| Tables, schema, records, workspaces | stay here |

---

## 1) What is Turbofy?

Turbofy is a platform for building data-driven web applications without deploying backend or frontend infrastructure. The user defines a data schema, fills tables with records, and assembles UI by composing **building blocks** on **pages** inside an **app**. The runtime resolves dynamic content server-side and renders React block components in iframes on the client.

Everything is exposed through the **Turbofy HTTP MCP**. Agents work against remote JSON and a remote block build session — there is **no local workspace checkout** under `~/.turbofy`, and there is **no TypeScript DSL** (`schema.ts` / `app.ts` / `flow.ts` / `record.ts`).

### Mental model

```
Organization
└── Workspace                                 ← unit of isolation; has a data schema
    ├── Schema (tables, fields, enums)        ← JSON via workspace_get / workspace_schema_push
    ├── Records (rows in each table)          ← data_* tools
    ├── Flow(s)                               ← JSON via flow_list / flow_get / flow_upsert
    └── App(s)                                ← JSON via app_get / app_push / app_init (list via data_list cmsapp)
        ├── Pages + BuildingBlocks            ← edited in the app manifest, pushed with app_push
        ├── BuildingBlockTypes (metadata)     ← same manifest / app_push
        ├── Persistent Markdown documents     ← metadata via app_get; content via app_document_*
        └── React sources                     ← remote session via block_type_* tools
```

### Which skill covers what

| You are working on… | Load |
|---|---|
| Workspace/schema; org/workspace discovery; `workspace_*` / `data_*` / `file_upload` | this skill |
| Apps CMS model, `app_*` (including `app_push`), pages/blocks/localizations | `turbofy-apps` |
| Block React sources via `block_type_*` | `turbofy-blocks` |
| Server-side `@dynamic_field` JS (`defaultConfig`, `defaultDynamicData`, etc.) | `turbofy-dynamic-fields` |
| Flows (`flow_*` JSON declarations) | `turbofy-flows` |
| Chatbot / AI assistant (Thread + Message tables, reply flow, chat block — there is no chatbot API) | `turbofy-chatbot` |

---

## 2) Workspaces & MCP identity

A **workspace** is the unit of isolation. It owns a data schema, records, apps, and flows.

**Always pass both `orgId` and `workspaceId`.** The MCP does not track an "active workspace". Discover IDs with `list_organizations` and `list_workspaces`.

The bundled plugin points at the **production HTTP MCP** (`https://mcp.turbofy.com/mcp`).

There is **no local file root**. Do not create or edit files under `~/.turbofy`. Schema, apps, and flows are edited as JSON through MCP tools. Block React sources are edited inside a remote build session (`block_type_*` tools).

---

## 3) MCP tools at a glance

Tool names are unprefixed (`list_workspaces`, `workspace_get`, …). Hosts may show them as `mcp__turbofy-http__<tool>` (from this plugin's MCP server key).

**Discovery:**

- `list_organizations`
- `list_workspaces`

**Workspace schema:**

- `workspace_get` — full workspace + schema JSON (`{ types, enums, products, … }`). Use `schema.types[].id` as the table `ofType` for `data_*` / `$$std`.
- `workspace_schema_push` — replace schema with the given JSON (`dryRun` defaults to `true`)

**Data CRUD (every table, including system CMS tables):**

- `data_list`, `data_get`, `data_create`, `data_add_many`, `data_update`, `data_delete` (`confirm: true` required for delete)

**Files:**

- `file_upload` — upload text (`content`) or binary (`contentBase64`); creates a `filedocument` record. Optional `folderId` (prefer an app-named folder for app assets).

**Apps** (details in `turbofy-apps`):

- `app_get`, `app_init`, `app_push` (list apps with `data_list` + `ofType: "cmsapp"`)
- `app_document_get`, `app_document_put`, `app_document_delete` (`app_get.documents` replaces a separate list call)

**Block type sources** (details in `turbofy-blocks`):

- `block_type_open`, `block_type_fs_list`, `block_type_fs_read`, `block_type_fs_write`, `block_type_check`, `block_type_push`

**Flows** (details in `turbofy-flows`):

- `flow_list`, `flow_get`, `flow_upsert` (`dryRun` defaults to `true`)

---

## 4) Core MCP rules

**Pass `orgId` + `workspaceId` on every workspace-scoped call.**

**Read before write.** Fetch with `data_get` / `workspace_get` / `app_get` / `flow_get` before updating or deleting.

**Verify after write.** Re-fetch or re-list. Prefer `dryRun: true` on `workspace_schema_push`, `app_push`, and `flow_upsert` before applying (`dryRun: false`).

**Treat IDs correctly.**

- **Workspace table IDs** (`ofType`) are workspace-specific. Resolve via `workspace_get` → `schema.types[].id` (and `name` for humans).
- **System CMS table IDs** are stable and are **not** part of the workspace schema. When calling `data_*`, pass the literal string:

  | Entity | MCP `ofType` |
  |---|---|
  | App | `"cmsapp"` |
  | Page | `"cmspage"` |
  | BuildingBlockType | `"cmsbuildingblocktype"` |
  | BuildingBlock | `"cmsbuildingblock"` |
  | Localization | `"cmslocalization"` |
  | AppDocument | `"appdocument"` |
  | EntityLocalization | `"localization"` |
  | Api | `"cmsapi"` |
  | FileDocument | `"filedocument"` |
  | SlugMapping | `"slugmapping"` |
  | Code | `"code"` |
  | Secret | `"secret"` |

**Pagination.** `data_list` is capped; use returned `nextToken`.

**Legacy `@cms_*` tables.** Some older workspaces still embed CMS tables in the workspace schema. Prefer system CMS `ofType` strings above. If app tooling refuses to operate, the workspace may need migration — contact the workspace owner.

---

## 5) Workspace schema workflow

Schema is **JSON**, not a TypeScript DSL.

1. **Get** — `workspace_get` → use the returned `schema` object as the starting point.
2. **Edit** — mutate that JSON in memory (add/change/remove types, fields, enums).
3. **Dry-run** — `workspace_schema_push` with `dryRun: true` (default) and review the planned diff.
4. **Apply** — same call with `dryRun: false`.

### Critical rules

- **Types omitted from the pushed schema are DELETED.** Always start from `workspace_get` output; never push a partial types array.
- **Ids:** existing types/fields keep their `id`. New types/fields/enums should omit `id` (server assigns). To **rename**, keep the same `id` and change `name` — matching is by id, not name.
- **Parents.** `parents.first` / `parents.second` reference a parent table that must exist in `types`. The order of the `types` array doesn't matter — matching is by reference, not position.
- **Default fields.** Tables already have `id` (`@connector`), `createdAt`, `updatedAt` — do not invent duplicates when editing an existing table; new tables should include the usual connector/`createdAt`/`updatedAt` fields consistent with sibling tables in the same workspace.
- **Enums** live in `schema.enums` as `{ name, values: string[] }`. Field type for an enum field is the enum **name** (e.g. `"SignalStatus"`).

### Schema JSON shape (essentials)

```json
{
  "types": [
    {
      "id": "wGluy5",
      "name": "Signal",
      "directives": [],
      "publishingTypes": [],
      "fields": [
        { "id": "9U29sX", "name": "id", "type": "ID", "directives": ["@connector"] },
        { "id": "nOvMag", "name": "message", "type": "String", "directives": [] },
        { "id": "mkT0Ld", "name": "status", "type": "SignalStatus", "directives": [] }
      ],
      "parents": { "first": null, "second": null },
      "targets": [],
      "resolvers": []
    }
  ],
  "enums": [
    { "name": "SignalStatus", "values": ["New", "Acknowledged"] }
  ],
  "products": []
}
```

Common field `type` values: `String`, `ID`, `Integer`, `Float`, `Boolean`, `Email`, `Json`, `DateTime`, `Date`, `Url`, list variants as used by existing tables in that workspace, plus enum names.

Common field directives: `@connector`, `@sortby`, `@auth`, `@user_email`, `@user_password`, `@dynamic_field`, `@users` (table-level).

Preserve unrelated top-level schema keys returned by `workspace_get` (`products`, auth modes, region, etc.) unless you intentionally change them — push replaces the schema declaration; starting from a full get avoids accidental loss.

---

## 6) Generic data tools

Use `data_*` for:

- CRUD on workspace records (`ofType` = `schema.types[].id` from `workspace_get`).
- Inspecting system CMS entities when you need a single record outside an app manifest.
- Secrets discovery: `data_list` with `ofType: "secret"` (id + name only; values are never readable).

`file_upload` creates a `filedocument` usable in later `data_*` calls.

For **app structure** (pages, block instances, block-type metadata/copies/`defaultConfig`), prefer `app_get` → edit → `app_push` — see `turbofy-apps`. For **persistent app documentation**, discover metadata in `app_get.documents` and use `app_document_get` / `app_document_put` / `app_document_delete`; there is no `app_document_list`. For **block React sources**, use `block_type_*` — see `turbofy-blocks`. Do not round-trip app CMS entities via `data_*` when a specialized app tool covers the operation.

---

## See also

- **`turbofy-apps`** — app manifest, `app_push` reconciliation, localization.
- **`turbofy-blocks`** — remote `block_type_*` build session + React component rules.
- **`turbofy-dynamic-fields`** — `$$self` / `$$args` / `$$std`.
- **`turbofy-flows`** — flow JSON declarations via `flow_upsert`.
- **`turbofy-chatbot`** — the chatbot recipe: Thread/Message tables + reply flow + chat block. Turbofy has **no chatbot API**; `product_chatbot` is a legacy flag.
