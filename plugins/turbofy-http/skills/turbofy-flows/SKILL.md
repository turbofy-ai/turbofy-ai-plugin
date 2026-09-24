---
name: turbofy-flows
description: "Create, edit, or debug Turbofy automation flows: triggers, schedules, dynamic step parameters, secret references, and run logs through the hosted MCP. Validate and push flow.ts using flowBuilder. For database schemas use turbofy-platform."
---

# Turbofy Flows

A flow runs an ordered sequence of steps when a trigger matches. Edit it as typed `flowBuilder` source in the hosted MCP session tree.

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
- Steps run in array order unless `next` chooses a later step. Do not point back to an earlier step.
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

Use exactly one of `cron` or `rate`. Turbofy uses six-field cron: minutes, hours, day-of-month, month, day-of-week, year. `timezone` applies only to cron. Optional `startDate` and `endDate` accept ISO dates/datetimes.

Schedules update automatically on push. Disabling a flow disables its schedules; removing a trigger removes its schedule. Advanced cron expressions can run even when the visual editor displays them as read-only. Exactly one of day-of-month/day-of-week must be `?` when the other is specified; use `rate` for simple intervals.

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

`genericAI` operations include `generateText`, `generateObject`, `streamText`, `embed`, and `generateImage`. With `streamText`, downstream steps run for published stream chunks and the final result; each chunk contains the accumulated text. This is useful for updating one draft record throughout generation.

## Image generation and editing

Use `genericAI` with `operation: "generateImage"`. Prefer it over the legacy `openAIImageGeneration` step, which only supports DALL-E.

```ts
flowBuilder.step.genericAI("heroImage", {
  params: {
    operation: "generateImage",
    apiKey: flowBuilder.secret("<secret-record-id>"),
    model: { provider: "openai", model: "gpt-image-2.5-flare" },
    prompt: flowBuilder.js("`Product photo of ${state.onProductCreated.name} on white`"),
    size: "1024x1024",
    outputFilename: flowBuilder.js("`${state.onProductCreated.id}-hero.png`"),
  },
}),
flowBuilder.step.updateType("saveImage", {
  params: {
    ofType: "<product-table-id>",
    id: flowBuilder.js("state.onProductCreated.id"),
    fields: flowBuilder.js("({ imageUrl: state.heroImage.value.images[0].url })"),
  },
}),
```

Providers and representative models (`model` accepts any id the provider serves):

| Provider | Models |
|---|---|
| `openai` | `gpt-image-2.5-sunburst` (precision, editing), `gpt-image-2.5-flare` (fast), `gpt-image-2` |
| `google` | `gemini-3-pro-image-preview`, `gemini-3.1-flash-image-preview`, `imagen-4.0-generate-001` |
| `xai` | `grok-imagine-image`, `grok-imagine-image-quality` |
| `fal` | `fal-ai/flux-pro/kontext/max`, `fal-ai/flux-pro/v1.1-ultra`, `fal-ai/qwen-image`, `fal-ai/recraft/v3/text-to-image` |
| `deepinfra` | `black-forest-labs/FLUX.1-Kontext-pro`, `black-forest-labs/FLUX-1.1-pro`, `stabilityai/sd3.5` |
| `replicate` | `black-forest-labs/flux-2-pro`, `black-forest-labs/flux-fill-pro`, `recraft-ai/recraft-v3` |
| `fireworks` | `accounts/fireworks/models/flux-kontext-max`, `accounts/fireworks/models/flux-1-dev-fp8` |
| `luma` | `photon-1`, `photon-flash-1` |
| `togetherai` | `black-forest-labs/FLUX.1-kontext-max`, `black-forest-labs/FLUX.1.1-pro` |
| `blackForestLabs` | `flux-kontext-max`, `flux-pro-1.1-ultra`, `flux-pro-1.0-fill` |

Parameters:

- `prompt` — required.
- `images` — array of image URLs or data URLs. Turns the call into an edit or reference-image call. Workspace file URLs (`state.<step>.value.images[0].url`, `filedocument.url`) work directly.
- `mask` — mask image URL for inpainting. Honoured by OpenAI, Google, Fal, DeepInfra, Replicate, Together.ai, and Black Forest Labs; xAI, Fireworks, and Luma ignore it and report a warning.
- `n` — number of images (default 1).
- `size` (`"1024x1024"`) or `aspectRatio` (`"16:9"`) — use one; support varies per model.
- `seed` — for reproducible output where the provider supports it.
- `outputFilename` — stored file name; with `n > 1` the files get `-1`, `-2`, ... suffixes. Defaults to a UUID.
- `providerOptions` — provider-specific options keyed by provider id, for example `{ openai: { quality: "high" } }`.

Result at `state.<stepName>`:

```ts
{ type: "imageResult", value: { images: [{ url, key, mimeType }], warnings: string[] } }
// or
{ type: "error", error: string }
```

Generated files are uploaded to the workspace bucket and registered as `filedocument` records. Guard downstream steps with `continueIf: "state.<stepName>.type === 'imageResult'"`.

Editing example — apply a change to an existing image:

```ts
flowBuilder.step.genericAI("recolor", {
  params: {
    operation: "generateImage",
    apiKey: flowBuilder.secret("<secret-record-id>"),
    model: { provider: "blackForestLabs", model: "flux-kontext-max" },
    prompt: "Make the background a soft gradient blue, keep the product unchanged",
    images: flowBuilder.js("[state.onProductCreated.imageUrl]"),
  },
}),
```

## Run logs

Every trigger match creates a `flowrun` record in the workspace. With `debug: true` each run also writes `flowrunlog` records as it progresses. Both are system tables: read them with the data tools, never write them.

1. Set `debug: true`, push, and fire the flow. A manual trigger fires on `data_create { ofType: "flowtrigger", item: { flowId: "<flowId>" } }`; a table trigger fires on the matching record write.
2. `data_list { ofType: "flowrun", sortOrder: "DESC" }` — newest run first. Each run carries `flowId`, `status`, and `createdAt`.
3. `data_list { ofType: "flowrunlog", sortOrder: "DESC" }` — newest logs first. Keep the entries whose `flowRunId` matches the run; the tools have no parent filter.

Each log's `content` is an object with a `type` and a `name` (the trigger key or step name):

| `type` | Payload |
|---|---|
| `trigger-matched` | `data` — the trigger data |
| `step-started` | `rawInput` — the `state` the step sees, `step` — its definition |
| `step-params-resolved` | `resolvedParams` — params after `js` and `secret` resolution; secrets show as `[REDACTED]` |
| `step-completed` | `output` — what was stored at `state.<name>` |
| `step-skipped` / `step-discontinued` | the step was skipped by `skipIf` or stopped by `continueIf` |
| `step-error` | `error` — message and stack |

A run that ends at `step-started` with no `step-completed` or `step-error` is still executing or hit the step timeout. The console shows the same logs per step under the step's Test tab. Runs and logs are visible to workspace members only, not to app users. Logs are kept indefinitely, so turn `debug` off once the flow behaves.

## Recursion and validation

Errors block push:

- missing or cyclic `next` targets
- a table write that unconditionally retriggers the same flow
- cross-flow write/trigger cycles
- invalid credentials or secret ids

A self-retrigger with a condition becomes a warning because the validator cannot prove the condition terminates. Ensure the condition excludes records written by the flow, for example a Message INSERT flow that only accepts `role === 'user'` and writes `role: 'assistant'`.

Also review warnings about step parameter shapes, dynamic table ids, and cloud-function writes. Passing validation does not prove those paths are correct.

## Dashboard card appearance

The flow card description, icon, and background color are stored on the flow declaration and show everywhere that flow is opened. Set them on `buildFlow` (UI-only; the runtime ignores them):

```ts
export const flow = flowBuilder.buildFlow({
  description: "Sync new orders to the CRM", // card subtitle, max 160 characters; omit for none
  color: "green", // grey | sand | purple | violet | green | red | blue | pink | orange, or "#rrggbb"
  icon: "mail", // omit for the default bot mark
  // ...triggers, steps
});
```

Icon ids: `bot`, `sparkles`, `mail`, `shopping-bag`, `calendar`, `user`, `users`, `file-text`, `globe`, `zap`, `heart`, `message-square`, `database`, `workflow`, `bell`, `image`, `credit-card`, `map`, `settings`, `sliders`, `star`, `truck`, `weather`, `fitness`, `marketing`, `sales`, `slides`, `project-management`.

Pull the flow, set `description`, `color`, and/or `icon`, then push. Do not drop other declaration fields. `description` must be at most 160 characters.

## See also

- `turbofy-platform` — workspace discovery, schema, table ids, secret metadata
- `turbofy-dynamic-fields` — app `$$std` runtime; flows use `state` instead
- `turbofy-chatbot` — streaming Message flow recipe
