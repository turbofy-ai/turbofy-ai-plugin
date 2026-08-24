---
name: turbofy-platform
description: "Use first for Turbofy work, especially organization/workspace discovery, database schema changes, record CRUD, file uploads, or choosing the right follow-on skill. Covers the hosted MCP session tree, workspace_pull → fs_* edit → workspace_push, the schema DSL, data tools, and file_upload/file_upload_intent. For app structure or UI load turbofy-apps or turbofy-blocks."
---

# Turbofy Platform

Turbofy combines a typed data platform, app builder, automation flows, and a React block runtime. The HTTP MCP keeps editable projects in a persistent remote session tree; use MCP `fs_*` tools to inspect and edit it.

## Start here

1. `list_organizations`
2. `list_workspaces` with the selected `orgId`
3. Keep `orgId` and `workspaceId` on every scoped call.
4. Use `table_list` for a quick schema/table overview, or pull the workspace when editing schema.

The active deployment determines `<environment>`. Tool results return the authoritative path; do not guess it.

## Session filesystem

Workspace projects live at:

```text
workspaces/<environment>/<workspaceId>/
  schema.ts
  schema.base.json
  package.json
  tsconfig.json
  apps/<appId>/...
  flows/<flowId>/...
```

Use only the MCP filesystem tools:

- `fs_list`, `fs_read`, `fs_search` to inspect
- `fs_edit` for exact replacements in existing files
- `fs_write` for new files or deliberate full rewrites
- `fs_exec` for project commands; long commands can return a task id
- `fs_task_status` to poll asynchronous work

The tree persists across MCP session restarts. Never edit server baselines such as `schema.base.json`, `app.base.json`, `flow.base.json`, or `.base/` contents.

## Tool map

| Concern | Tools |
|---|---|
| Discovery | `list_organizations`, `list_workspaces`, `table_list` |
| Schema | `workspace_pull`, `workspace_push` |
| Apps | `app_init`, `app_pull`, `app_push` |
| Blocks | `block_type_open`, `block_type_check`, shared `fs_*` |
| Flows | `flow_init`, `flow_pull`, `flow_push`, `flow_delete` |
| Records | `data_list`, `data_get`, `data_create`, `data_add_many`, `data_update`, `data_delete` |
| Files | `file_upload`, `file_upload_intent` |

Use `dryRun: true` before mutating pushes. It is the default for `workspace_push`, `app_push`, and `flow_push`.

## Schema workflow

1. `workspace_pull` materializes `schema.ts` plus the merge baseline.
2. Read and edit `schema.ts` through `fs_*`.
3. `workspace_push` compiles, validates, and computes a three-way merge. Review the dry run.
4. Repeat with `dryRun: false` to apply.

If remote and session edits overlap, push reports conflicts and applies nothing. Pull the current remote version and reapply the intended changes.

When schema changes are part of app work, `app_push` also reconciles the app's `schema.ts`. Use `workspace_push` for schema-only work.

### Data-builder DSL

```ts
import { dataBuilder as builder } from "@graphapi-io/dsl-builders";

const StatusEnum = builder.enumType("Status", ["Active", "Inactive"]);

const ProjectTable = builder.table(
  "Project",
  {
    name: builder.fields.string(),
    status: builder.fields.enum(StatusEnum),
  },
  { directives: ["@required_oncreate"] },
);

const TaskTable = builder.table(
  "Task",
  { title: builder.fields.string() },
  { firstParent: ProjectTable },
);

export const schema = builder.build({
  enums: [StatusEnum],
  types: [ProjectTable, TaskTable],
  products: [],
});
```

Field factories include `string`, `integer`, `float`, `boolean`, `id`, `email`, `phone`, `url`, `date`, `dateTime`, `time`, `timestamp`, `json`, `ipAddress`, list variants, `enum`, `dynamicField`, and `localizedString`.

### Searchable tables

Add `"@fts_searchable"` to a table's `directives` to index it for
`$$std.queryRecords` / `$$std.searchRecords` (see `turbofy-dynamic-fields`)
and the REST `query`/`search` routes. Optional `searchConfig` tunes
indexing per field:

```ts
const PartTable = builder.table(
  "Part",
  {
    name: builder.fields.string(),
    attributes: builder.fields.json(),
    title: builder.fields.localizedString({ locales: ["en", "de"] }),
  },
  {
    directives: ["@fts_searchable"],
    searchConfig: {
      fields: {
        attributes: { indexKeys: true, keyTypes: { voltage: "number" } },
      },
    },
  },
);
```

- Json fields with `indexKeys: true` become filterable by dotted key paths
  (`attributes.voltage`); `keyTypes` pins a key's value type
  (`"text" | "number" | "boolean"`).
- Localized string fields index every declared locale under
  `field.<locale>` — filterable and full-text searchable per language.
- Creating, updating, or deleting a record in a searchable table costs 5
  credits instead of 1.

Rules:

- Preserve ids emitted for existing declarations; omit ids for new types, fields, and enums.
- Declare parents before children. A table can use `firstParent` and `secondParent`.
- Do not add automatic `id`, `createdAt`, or `updatedAt` fields.
- Include every table and enum in `builder.build(...)`; omission means deletion.

## Record CRUD

Generic `data_*` tools work across workspace tables and system CMS tables. Use the exact table id/`ofType`; runtime code must not substitute display names.

Common system `ofType` values:

| Entity | `ofType` |
|---|---|
| App | `cmsapp` |
| Page | `cmspage` |
| Building block type | `cmsbuildingblocktype` |
| Building block | `cmsbuildingblock` |
| Localization | `cmslocalization` |
| File document | `filedocument` |
| Slug mapping | `slugmapping` |
| Secret metadata | `secret` |

Prefer app files and `app_push` for app-owned entities. Use `data_*` for ordinary records and targeted inspection. Paginate `data_list` with its returned `nextToken`.

## Files

`file_upload` creates a `FileDocument` and accepts exactly one source:

- `content` for small UTF-8 text
- `contentBase64` for small binary payloads already available to the caller
- `sourceUrl` for a public HTTPS resource; the service downloads it directly and uploads it to storage, so it need not pass through the agent sandbox. URL fetching is capped at 25 MiB.

Supply the MIME type when known; a URL response MIME type can be used otherwise. The default folder is workspace-scoped. The returned record id can be stored anywhere that expects `ofType: "filedocument"`.

Use `file_upload_intent` when the bytes are too large or should travel directly from a client to storage. It creates the `FileDocument` and returns a presigned `PUT` request, including the URL, method, and required `Content-Type`. Upload the bytes exactly as instructed; the upload URL is temporary and should not be stored as the asset URL.

## Core rules

- Treat ids as opaque and preserve them across edits.
- Read before write; never push partial declarations.
- Never expose secret values. Flows reference secret record ids; secret values are managed in the dashboard.
- When dependency setup is asynchronous, inspect the returned state and retry validation after completion.

## Follow-on skills

- `turbofy-apps` — pages, app settings, block placement, docs, localization, auth
- `turbofy-blocks` — React block source and block validation
- `turbofy-dynamic-fields` — server-side `$$std` data logic
- `turbofy-flows` — automation declarations and flow runtime
- `turbofy-chatbot` — Thread/Message + flow + UI recipe
