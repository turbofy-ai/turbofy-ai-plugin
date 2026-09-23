---
name: turbofy-platform
description: "Discover Turbofy organizations and workspaces, edit database schemas, manage records, upload files, and read or publish files the user attached in chat, through the hosted MCP. Also use for platform orientation and selecting an app, block, or flow skill."
---

# Turbofy Platform

Turbofy combines a typed data platform, app builder, automation flows, and a React block runtime. The HTTP MCP keeps editable projects in a persistent remote session tree; use MCP `fs_*` tools to inspect and edit it.

## Start here

1. Reuse known organization/workspace ids from the user or current task. Otherwise call `list_organizations`.
2. Call `list_workspaces` with the selected `orgId` when the workspace is unknown.
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
  files/<fileId>          # scratch space for file_pull and file_push, never saved
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
| Files | `file_upload`, `file_upload_intent`, `file_pull`, `file_push`, `file_view`, `file_set_visibility` |

Use `dryRun: true` before mutating pushes. It is the default for `workspace_push`, `app_push`, and `flow_push`.

## Schema workflow

1. `workspace_pull` materializes `schema.ts` plus the merge baseline.
2. Read and edit `schema.ts` through `fs_*`.
3. `workspace_push` validates and previews schema changes. Review the dry run.
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
| Shared module | `cmssharedmodule` |
| Localization | `cmslocalization` |
| File document | `filedocument` |
| Slug mapping | `slugmapping` |
| Secret metadata | `secret` |

Prefer app files and `app_push` for app-owned entities. Use `data_*` for ordinary records and targeted inspection. Paginate `data_list` with its returned `nextToken`. Pass `parentType` and `parentId` to `data_list` to get only the children of one record, for example the files in an Assets folder.

## Files

`file_upload` creates a `FileDocument` and accepts exactly one source:

- `content` for small UTF-8 text
- `contentBase64` for small binary payloads already available to the caller
- `sourceUrl` for a public HTTPS resource; the service downloads it directly and uploads it to storage, so it need not pass through the agent sandbox. URL fetching is capped at 25 MiB.

Supply the MIME type when known; a URL response MIME type can be used otherwise. The default folder is workspace-scoped. The returned record id can be stored anywhere that expects `ofType: "filedocument"`.

Use `file_upload_intent` when the bytes are too large or should travel directly from a client to storage. It creates the `FileDocument` and returns a presigned `PUT` request, including the URL, method, and required `Content-Type`. Upload the bytes exactly as instructed; the upload URL is temporary and should not be stored as the asset URL.

Use `file_pull` to read a workspace file yourself. It copies the `FileDocument`'s content into the session tree at `files/<fileId>` and returns that path; inspect it with `fs_read`, or parse it with `fs_exec` (node and npm are available, so install a parser such as a PowerPoint or Excel reader when needed). The copy is scratch space: it is not saved and the Assets file is unchanged. `file_pull` only reaches workspace Assets; bring internet files in with `file_upload` and `sourceUrl` first.

Use `file_push` to save a file you created or edited in the session tree, such as an edited PDF, a generated chart or a converted image. It creates a new `FileDocument` from the sandbox path and returns it; the bytes go from the sandbox straight to storage, so never read them into a message or upload them with `file_upload` content. It never overwrites an existing file, so pushing an edited version leaves the original untouched; tell the user which file is new. New files are `PRIVATE` unless you pass `accessControl: "PUBLIC"`. Pass `folderId` to keep a result next to its source, for example the same `Chats` folder.

Use `file_view` to look at an image yourself: it returns the image as content you can see, from Assets (`fileId`) or from the session tree (`path`). It takes PNG, JPEG, GIF and WebP up to 3.75 MB; convert other formats in the sandbox and view the result by path. Views cost context, so look at what the task needs rather than every file in a folder.

PDFs are read in the sandbox, because not every model accepts them: `file_pull` the file and extract its text with `pdfjs-dist`. When the layout matters, render the pages you need to PNG with `pdf-to-img` and `file_view` them.

`file_set_visibility` makes a `FileDocument` `PUBLIC` or `PRIVATE`. A private file has no loadable URL, so an app can only show a public one. It returns the updated record, including the file's current `url`.

### Using the user's Assets

Assets folders are `filedocumentfolder` records; the workspace root folder's id is the `workspaceId`. When the user points at a folder ("use the files in branding"):

1. `data_list` with `ofType: "filedocumentfolder"` to find the folder by `name`.
2. `data_list` with `ofType: "filedocument"`, `parentType: "filedocumentfolder"` and the folder id to list its files.
3. `file_view` the images you need to understand (logo colors); `file_pull` PDFs such as brand guidelines, text and other formats, and read them in the sandbox.
4. `file_set_visibility` with `PUBLIC` on the files the app will show, then reference the record or its `url`.

### Chat attachments

Files a user attaches in the Turbofy chat are private `FileDocument` records under Assets > Chats > the conversation title. Each one appears in the user's message as a line like:

```
[Attached file: deck.pptx, Assets file id abc12345-deck.pptx, private; not shown inline, read it in the sandbox with file_pull]
```

- Images and text files are already in the message; text over 8k characters is cut, and `file_pull` reads the rest. PDFs are not, read them in the sandbox as described above.
- For anything marked "not shown inline", call `file_pull` with the file id before answering questions about its content.
- To edit an attachment, `file_pull` it, change the copy with `fs_exec` (for PDFs, `pdf-lib` handles forms, stamps, page edits and merges; rewriting existing text in place is not practical), then `file_push` the result.
- To use an attachment in an app, such as a logo, call `file_set_visibility` with `PUBLIC`, then reference the record or its public `url`. Make a file public only when the user clearly means it for publication; a logo is, a screenshot of internal data is not. Ask when unsure.

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
