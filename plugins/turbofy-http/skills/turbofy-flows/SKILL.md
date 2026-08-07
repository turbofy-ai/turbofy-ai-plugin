---
name: turbofy-flows
description: "Use when building or modifying Turbofy automations: initializing or pulling flows, editing flow.ts with flowBuilder, validating triggers/steps/dynamic params/secrets, pushing with a dry run, or deleting a flow. Covers flow_init/flow_pull/flow_push/flow_delete and the hosted flows session tree. For workspace schema load turbofy-platform."
---

# Turbofy Flows

A flow is a set of triggers and a linked list of server-side steps. Edit it as typed `flowBuilder` source in the hosted MCP session tree.

## Workflow

| Tool | Purpose |
|---|---|
| `flow_pull` | Materialize all workspace flows and return available secret metadata |
| `flow_init` | Create an empty remote flow and scaffold its source |
| `flow_push` | Compile, validate, three-way merge, and push one flow; dry-run by default |
| `flow_delete` | Delete a flow intentionally |

```text
workspaces/<environment>/<workspaceId>/flows/
  package.json
  tsconfig.json
  validate.ts
  <flowId>/
    flow.ts       # editable
    flow.base.json # merge baseline; never edit
```

For an existing flow:

1. `flow_pull`
2. Use `fs_read`/`fs_edit` on `<flowId>/flow.ts`.
3. Optionally run the shared validation command shown by the scaffold with `fs_exec`; poll long tasks with `fs_task_status`.
4. `flow_push` with the default `dryRun: true` and review validation, operations, and conflicts.
5. Apply with `dryRun: false`.

Push validates the edited flow against all remote flows and verifies referenced secret records. When the same flow changed remotely, pull again and reapply the intended edit.

## Runtime model

- A matching trigger seeds a `state` output map under the trigger key.
- Each step reads `state`, runs, stores its result under its step name, then advances.
- Steps form a linked list. Array order supplies the default `next`; `next` may override it.
- `skipIf` skips one step and continues. `continueIf` stops the flow when falsy.
- `debug: true` adds verbose logs; `disabled: true` prevents execution.

Trigger data:

- table INSERT/REMOVE: the record
- table MODIFY: `{ old, new }`
- manual: `{}`
- schedule: `{ scheduledTime }`

## DSL

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Order fulfillment",
  triggers: {
    onOrderCreated: flowBuilder.trigger.tableDataChange({
      tableId: "<table-id>",
      operation: "INSERT",
      condition: "state.status === 'paid'",
    }),
    manual: flowBuilder.trigger.manual(),
  },
  steps: [
    flowBuilder.step.httpRequest("notifyErp", {
      params: {
        httpMethod: "POST",
        url: "https://example.com/orders",
        headers: { Authorization: flowBuilder.secret("<secret-record-id>") },
        body: flowBuilder.js("JSON.stringify(state.onOrderCreated)"),
      },
    }),
    flowBuilder.step.createType("writeLog", {
      params: {
        ofType: "<log-table-id>",
        fields: flowBuilder.js("({ status: state.notifyErp.status })"),
      },
    }),
  ],
  debug: false,
  disabled: false,
});
```

- Trigger keys and step names become keys in `state`.
- Table triggers use table ids and operation `INSERT`, `MODIFY`, or `REMOVE`.
- Conditions are JavaScript strings. Trigger conditions see trigger data; step expressions see the accumulated output map.
- `next: null` ends early. Avoid `detachedSteps` in new flows.

## Schedules

```ts
nightly: flowBuilder.trigger.schedule({
  cron: "0 9 * * ? *",
  timezone: "Europe/Warsaw",
}),

everyTenMinutes: flowBuilder.trigger.schedule({
  rate: { value: 10, unit: "minutes" },
}),
```

Use exactly one of `cron` or `rate`. Turbofy uses AWS six-field cron: minutes, hours, day-of-month, month, day-of-week, year. `timezone` applies only to cron. Optional `startDate` and `endDate` accept ISO dates/datetimes.

Schedules reconcile automatically on push and are disabled or removed with their flow/trigger lifecycle.

## Dynamic params and secrets

- Static values are stored as written.
- `flowBuilder.js("<expression>")` evaluates against `state` when the step runs.
- `flowBuilder.secret("<secret-record-id>")` resolves a managed secret just in time.
- To return an object literal from `js`, wrap it: `flowBuilder.js("({ id: state.x.id })")`.
- Markers inside arrays are not addressable; wrap the whole array in one `js(...)` expression.
- Discover secret ids with `data_list { ofType: "secret" }` or the `flow_pull` result. Values are created and managed in the dashboard and must never appear in source.

Credential fields for AI and integration steps must use `secret(...)` or `js(...)`, never literal keys.

## Step catalog

Every factory follows `flowBuilder.step.<type>(name, { params, description?, next?, skipIf?, continueIf? })`.

| Category | Steps |
|---|---|
| Write | `createType`, `batchCreateType`, `updateType`, `deleteType` |
| Read | `type`, `batchGetType`, `listType`, `listTypeByParent` |
| Logic/integration | `logic`, `httpRequest`, `cloudFunction`, `notifyWebSocket`, `googleSearch`, `linkScraper`, `htmlToPdf`, `extractImageMetadata` |
| AI/media | `genericAI`, `openAIImageGeneration`, `elevenLabsTTS` |

The result of each step is stored at `state.<stepName>`. Consult the scaffold typings and validation errors for the exact parameter shape of the selected step.

`genericAI` operations include `generateText`, `generateObject`, `streamText`, and `embed`. With `streamText`, downstream steps run for published stream chunks and the final result; each chunk contains the accumulated text. This is useful for updating one draft record throughout generation.

## Recursion and validation

Errors block push:

- missing or cyclic `next` targets
- a table write that unconditionally retriggers the same flow
- cross-flow write/trigger cycles
- invalid step params, credentials, or secret ids

A self-retrigger with a condition becomes a warning because the validator cannot prove the condition terminates. Ensure the condition excludes records written by the flow, for example a Message INSERT flow that only accepts `role === 'user'` and writes `role: 'assistant'`.

Dynamic `ofType` values and cloud functions limit static recursion analysis; review those paths manually.

## See also

- `turbofy-platform` — workspace discovery, schema, table ids, secret metadata
- `turbofy-dynamic-fields` — app `$$std` runtime; flows use `state` instead
- `turbofy-chatbot` — streaming Message flow recipe
