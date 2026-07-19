---
name: turbofy-platform
description: "Use as the first skill for any Turbofy work — especially when the user asks 'what is Turbofy', 'list my workspaces', 'set up my database', 'add a table or field', 'change my data model', 'add/edit/delete records', 'upload a file', or 'connect to Turbofy'. Also load at the start of a fresh session before calling any Turbofy MCP tool, to pick the right follow-on skill. Covers org/workspace discovery, environments, schema editing, record CRUD, file upload, and core MCP rules. Do NOT use for page layout, placing sections on pages, or building UI components — load turbofy-apps, turbofy-blocks, or turbofy-dynamic-fields instead."
disable-model-invocation: false
---

# Turbofy Platform

This is the orientation skill for the Turbofy platform. Load it first when you walk into a fresh session, when you're working at the workspace/schema level (not inside a specific app), or when you need to decide which deeper skill to pull in. The three companion skills — `turbofy-apps`, `turbofy-blocks`, `turbofy-dynamic-fields` — each own a deeper slice of the platform.

## When to load this skill

Load when the user asks about **workspaces, databases, tables, records, or getting started with Turbofy** — e.g. "list my workspaces", "add a Products table", "upload a file". Do not wait for them to mention `schema.ts`, MCP tools, or Turbofy internals.

| User wants… | Load instead |
|---|---|
| Pages, layout, homepage, translate the site | `turbofy-apps` |
| Build/style a section (navbar, hero, grid) | `turbofy-blocks` |
| Server data blank, fetch records, URL-based titles | `turbofy-dynamic-fields` |
| Tables, schema, records, workspaces | stay here |

---

## 1) What is Turbofy?

Turbofy is a platform for building data-driven web applications inside a dashboard, without deploying any backend or frontend infrastructure. The user defines a data schema, fills the tables with records, and assembles UI by composing **building blocks** on **pages** inside an **app**. The runtime resolves dynamic content server-side and renders React block components in iframes on the client.

Everything is exposed through the **Turbofy MCP** (`@turbofy-ai/mcp`), which lets an agent discover workspaces, pull schemas and apps to disk, edit them as code, and push the changes back.

### Mental model

```
Organization
└── Workspace                                 ← unit of isolation; has a data schema
    ├── Schema (tables, fields, enums)        ← schema.ts in the workspace root
    ├── Records (rows in each table)          ← managed via Turbofy_data_*
    └── App(s)                                ← apps/<appId>/ under the workspace root
        ├── Pages                             ← routes; defined in app.ts
        │   └── BuildingBlocks                ← instances of a BuildingBlockType on a page
        ├── BuildingBlockTypes                ← block-types/<Name>/record.ts (manifest)
        │   └── Runtime React component       ← block-types/<Name>/index.tsx
        └── @dynamic_field code               ← JS strings run server-side in QuickJS
            (defaultConfig, defaultDynamicData, localizedConfig, …)
```

### Which skill covers what

| You are working on…                                                                         | Load                      |
| ------------------------------------------------------------------------------------------- | ------------------------- |
| Workspace/schema-level work; orientation; org/workspace discovery; `Turbofy_workspace_*`    | this skill                |
| Apps CMS data model, `Turbofy_app_*` workflow, app file layout, localization, block instances | `turbofy-apps`            |
| Writing a block type's runtime React component (`block-types/<Name>/index.tsx`)             | `turbofy-blocks`          |
| Writing server-side `@dynamic_field` JS (`defaultConfig`, `defaultDynamicData`, etc.)       | `turbofy-dynamic-fields`  |
| Flows (workspace automations): `Turbofy_flow_*` workflow, flowBuilder DSL, triggers/steps    | `turbofy-flows`           |

Multiple skills can be active at once — pull in whichever match the work in flight.

---

## 2) Workspaces & environments

### Workspaces

A **workspace** is the unit of isolation. It owns a data schema, records, and any apps built on top. Every MCP tool that touches workspace state takes a `workspaceId` argument — there is no "active workspace" tracked by the MCP. Discover IDs with `Turbofy_list_workspaces` (and `Turbofy_list_organizations` for the org scope).

### Environments

Turbofy runs in two environments: **`prod`** and **`alpha`**. The bundled MCP is currently pinned to `prod`. A future MCP release will honor a `TURBOFY_ENV=alpha` env var and surface a second `turbofy-alpha` server alongside `turbofy`.

When both are available, use `turbofy` for production work and `turbofy-alpha` for the alpha stage. Both expose the same tool surface (`Turbofy_*`).

### Local file root

The MCP materializes workspace state under a single, predictable root:

```
~/.turbofy/workspaces/<environment>/<workspaceId>/
  schema.ts                 # workspace data schema (DSL) — owned by this skill
  schema.base.json          # diff baseline for Turbofy_workspace_push (do not edit)
  package.json              # scaffolded so the directory is self-contained
  tsconfig.json
  apps/
    <appId>/                # one directory per app — owned by turbofy-apps
      app.ts
      app.base.json
      block-types/
        <Name>/
          record.ts         # block type manifest
          index.tsx         # runtime React component (optional)
```

`<environment>` is `prod` or `alpha` depending on which MCP is active. `<workspaceId>` and `<appId>` are reported by `Turbofy_list_workspaces` and `Turbofy_app_*`.

The `apps/<appId>/` subtree and everything in it is documented by `turbofy-apps`. The `schema.ts` + `schema.base.json` pair at the workspace root is owned by this skill (see § 5).

---

## 3) MCP tools at a glance

The `turbofy` (and future `turbofy-alpha`) MCP exposes a single `Turbofy_*` tool surface. Grouped by concern:

**Discovery (org & workspace level):**

- `Turbofy_list_organizations`
- `Turbofy_list_workspaces`

**Workspace schema:**

- `Turbofy_workspace_pull` — write the current schema to `~/.turbofy/workspaces/<env>/<wsId>/` as `schema.ts` + `schema.base.json`
- `Turbofy_workspace_push` — typecheck, compile, validate, and push the local `schema.ts`
- `Turbofy_table_list` — list workspace tables with field names/types and **table IDs** (workspace-specific)

**Generic data CRUD (works on every table, including the system CMS tables):**

- `Turbofy_data_list`, `Turbofy_data_get`, `Turbofy_data_create`, `Turbofy_data_add_many`, `Turbofy_data_update`, `Turbofy_data_delete`

**Files:**

- `Turbofy_file_upload` — upload a file and create a FileDocument record

**Apps (unified workspace — covered by `turbofy-apps`):**

- `Turbofy_app_init`, `Turbofy_app_pull`, `Turbofy_app_push`

**Flows (covered by `turbofy-flows`):**

- `Turbofy_flow_init`, `Turbofy_flow_pull`, `Turbofy_flow_push`

---

## 4) Core MCP rules

These apply to every tool call against the Turbofy MCP.

**Pass the workspace ID explicitly.** No "active workspace" is tracked. Every tool that needs a workspace accepts a `workspaceId` argument. Within a pulled app directory the workspace is implicit from the directory path; everywhere else, discover IDs via `Turbofy_list_workspaces`.

**Read before write.** Before updating or deleting a record, fetch it (`Turbofy_data_get`) to confirm IDs and current state. Before creating related records, list the parent/collection (`Turbofy_data_list` with the right `ofType`) to avoid duplicates.

**Verify after write.** After create/update/delete, fetch/list again to confirm the intended state. Before `Turbofy_app_push` or `Turbofy_workspace_push`, call it with `dryRun: true` (where supported) to preview the diff.

**Treat IDs correctly.**

- **Workspace-schema table IDs** are workspace-specific. Resolve them via `Turbofy_table_list`, or read them from `schema.ts` (each table builder call carries an `{ id: "..." }` option, and new tables expose `SomeTable.id`).
- **System CMS table IDs** are stable and never appear in `Turbofy_table_list` output. Two ways to reference them depending on context:
  - **Inside the workspace/app DSL** (TypeScript: `schema.ts`, `app.ts`, `record.ts`) — import and use the enum, e.g. `CmsOfTypeEnum.Page`.
  - **When calling MCP tools directly** (e.g. `Turbofy_data_list`, `Turbofy_data_get`) — pass the literal string value. The enum cannot be used here.

  | Entity              | DSL enum                          | MCP string value         |
  | ------------------- | --------------------------------- | ------------------------ |
  | App                 | `CmsOfTypeEnum.App`               | `"cmsapp"`               |
  | Page                | `CmsOfTypeEnum.Page`              | `"cmspage"`              |
  | BuildingBlockType   | `CmsOfTypeEnum.BuildingBlockType` | `"cmsbuildingblocktype"` |
  | BuildingBlock       | `CmsOfTypeEnum.BuildingBlock`     | `"cmsbuildingblock"`     |
  | Localization        | `CmsOfTypeEnum.Localization`      | `"cmslocalization"`      |
  | EntityLocalization  | `CmsOfTypeEnum.EntityLocalization`| `"localization"`         |
  | Api                 | `CmsOfTypeEnum.Api`               | `"cmsapi"`               |
  | FileDocument        | `CmsOfTypeEnum.FileDocument`      | `"filedocument"`         |
  | SlugMapping         | `CmsOfTypeEnum.SlugMapping`       | `"slugmapping"`          |
  | Code                | `CmsOfTypeEnum.Code`              | `"code"`                 |

**Know common limits.** `Turbofy_data_list` is capped (commonly 100 per request). Paginate with the returned `nextToken`.

**Legacy `@cms_*` tables.** If a workspace schema still uses legacy `@cms_*` directive tables, app tooling refuses to operate until the workspace is migrated to system CMS types. Contact the workspace owner to migrate before continuing.

---

## 5) Workspace schema workflow

Modifying the workspace's data schema is a three-step loop:

1. **Pull** — `Turbofy_workspace_pull` writes the current schema to `~/.turbofy/workspaces/<env>/<wsId>/` as `schema.ts` (the editable DSL) and `schema.base.json` (the diff baseline — do not edit by hand).
2. **Edit** — modify `schema.ts` using the data-builder DSL (see § 6).
3. **Push** — `Turbofy_workspace_push` typechecks, compiles, validates, and pushes.

If the push fails, fix the reported errors in `schema.ts` and call push again.

When you're editing schema as part of broader app work, `Turbofy_app_push` reconciles schema changes as a side-effect of the app push. For pure schema work — adding tables, fields, enums, parents — use `Turbofy_workspace_push` directly.

---

## 6) Data-builder DSL reference

`schema.ts` imports and uses the data builder:

```ts
import { dataBuilder as builder } from "@graphapi-io/dsl-builders";
```

### Enums

```ts
const StatusEnum = builder.enumType("Status", ["Active", "Inactive"]);
```

### Tables

```ts
const ProjectTable = builder.table(
  "Project",
  {
    name: builder.fields.string(),
    description: builder.fields.string(),
    status: builder.fields.enum(StatusEnum),
  },
  {
    directives: ["@required_oncreate", "@sortby"],
    publishingTypes: ["Queries", "Mutations"],
  },
);
```

### Fields

Scalar: `string()`, `integer()`, `float()`, `boolean()`, `id()`, `email()`, `phone()`, `url()`, `date()`, `dateTime()`, `time()`, `timestamp()`, `json()`, `ipAddress()`

List: `listString()`, `listInteger()`, `listFloat()`, `listBoolean()`, `listId()`

Special: `enum(enumDeclaration)`, `dynamicField()`

All field methods accept optional opts:

```ts
builder.fields.string({ label: "Full Name", directives: ["@auth"] });
```

### Parent-child relationships

```ts
const TaskTable = builder.table(
  "Task",
  {
    title: builder.fields.string(),
  },
  {
    firstParent: ProjectTable,
  },
);
```

Up to two parents are supported via `firstParent` and `secondParent`.

### Build call

Every schema file must end with `builder.build()`:

```ts
export const schema = builder.build({
  enums: [StatusEnum],
  types: [ProjectTable, TaskTable],
  products: [],
});
```

### Critical schema rules

- **Do NOT provide IDs for new types, fields, or enums.** The builder auto-generates 6-character unique IDs. Existing types in the decompiled file already have `{ id: "..." }` — keep those as-is. Only omit IDs for things you are adding.
- **Declare parents before children.** Tables used as `firstParent` or `secondParent` must appear earlier in the file.
- **Default fields are automatic.** Every table automatically gets `id` (with `@connector`), `createdAt`, and `updatedAt`. Do not add these manually.
- **Include all types in the build call.** Every table and enum variable must appear in the `builder.build()` arrays, otherwise it will be silently dropped.

---

## 7) Generic data tools

`Turbofy_data_*` is the generic CRUD surface across **every** table — both workspace schema tables and system CMS tables. Use it for:

- One-off reads/writes against workspace records.
- Things the workspace files don't own — arbitrary `Localization` rows, `SlugMapping` entries, `FileDocument` records.
- Inspecting app entities from the outside — pass `ofType` as the **literal string value** for system CMS tables (e.g. `"cmspage"`, `"cmsbuildingblock"`, `"cmslocalization"`, `"filedocument"`, `"slugmapping"`). See the mapping table in § 4 — `CmsOfTypeEnum.X` only works inside the DSL, not in MCP tool arguments.

For app entities that are owned by the workspace files (pages, block types, block instances, block-type copies), prefer the `Turbofy_app_*` workflow — see `turbofy-apps`. Writing those directly via `Turbofy_data_*` works but the next `Turbofy_app_pull` / `Turbofy_app_push` will round-trip from / overwrite based on whatever is on disk.

`Turbofy_file_upload` uploads a file and creates a backing `FileDocument` record. The returned IDs are valid `ofType: "filedocument"` references for subsequent `Turbofy_data_*` calls.

---

## See also

- **`turbofy-apps`** — Apps CMS data model (App, Page, BuildingBlockType, BuildingBlock, Localization, Image, SlugMapping), the `Turbofy_app_*` workflow, `apps/<appId>/` file layout, localization workflow, block instance editing, macros.
- **`turbofy-blocks`** — writing block-type runtime React components (`block-types/<Name>/index.tsx`): props, `config.copies`, client-side `@/api` hooks, `@/navigation`, UI/UX/accessibility rules.
- **`turbofy-dynamic-fields`** — server-side `@dynamic_field` JavaScript: `$$self`, `$$args`, the `$$std` API, reserved `dynamicArgs` keys, debugging.
