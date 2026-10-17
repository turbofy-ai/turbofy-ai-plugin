---
name: turbofy-flows
description: "Create, edit, or debug Turbofy automation flows: triggers, branches, loops, schedules, dynamic step parameters, secret references, cloud functions, and run logs through the hosted MCP. Validate and push flow.ts using flowBuilder. For database schemas use turbofy-platform."
---

# Turbofy Flows

A flow runs sequences of steps when a trigger matches, with conditional paths and loops where needed. Edit it as typed `flowBuilder` source in the hosted MCP session tree.

## Workflow

| Tool | Purpose |
|---|---|
| `flow_pull` | Materialize workspace flows and cloud function sources; return functions and secret metadata |
| `flow_init` | Create an empty remote flow and scaffold its source |
| `flow_push` | Compile referenced functions, validate, detect conflicts, and push one flow; dry-run by default |
| `flow_delete` | Delete a flow intentionally |

```text
workspaces/<environment>/<workspaceId>/flows/
  package.json
  tsconfig.json
  validate.ts
  <flowId>/
    flow.ts       # editable
    flow.base.json # merge baseline; never edit
  functions/<name>/
    function.ts  # cloudFunction declaration
    index.ts     # Node.js handler, or Dockerfile and its source context
    function.base.json # source baseline; never edit
```

For an existing flow:

1. `flow_pull`
2. Use `fs_read`/`fs_edit` on `<flowId>/flow.ts`.
3. Optionally run the shared validation command shown by the scaffold with `fs_exec`; poll long tasks with `fs_task_status`.
4. `flow_push` with the default `dryRun: true` and review validation, operations, and conflicts.
5. Apply with `dryRun: false`.

Push validates the edited flow against all remote flows and verifies referenced secret records and cloud functions. When the same flow changed remotely, pull again and reapply the intended edit.

## Cloud functions

Cloud functions live in the same workspace tree as flows. Create `functions/<name>/function.ts`:

```ts
import { cloudFunctionBuilder } from "@turbofy-ai/app-runtime/dsl";

export const cloudFunction = cloudFunctionBuilder.buildFunction({
  name: "transform-order",
  runtime: "NODEJS_22_X",
  entry: "index.ts",
});
```

Write `index.ts` with a named `handler` export using the Lambda Function URL event/response contract. Node.js 22 and 24 runtimes bundle TypeScript/JavaScript into a CommonJS deployment artifact. Put npm dependencies in the function's own `package.json` and generate `package-lock.json` with `npm install` there. Push uses `npm ci --ignore-scripts`; use Docker for native dependencies or install scripts. Imports must stay within the function directory and its dependencies.

Reference the function in `flow.ts`:

```ts
flowBuilder.step.cloudFunction("transform", {
  params: {
    cloudFunctionUrl: cloudFunctionBuilder.ref("transform-order"),
    order: flowBuilder.js("state.onOrderCreated"),
  },
}),
```

Function names start with a lowercase letter, contain lowercase letters, digits or hyphens, and are at most 48 characters. Existing static function URLs are also accepted if they exist in the workspace. Dynamic `cloudFunctionUrl` expressions cannot be verified and are rejected by push.

For Docker, declare `{ name: "transform-order", runtime: "DOCKER" }` and place a Lambda-compatible `Dockerfile` plus its text source files in that directory. Push packages the named build context and starts the workspace's CodeBuild pipeline. If the result reports `cloudFunctions[].state: "pending"`, the flow has not been saved. Retry after the build finishes; unchanged pending builds are not restarted. A failed build can be retried with another push, after inspecting/fixing the build failure.

`flow_push` deploys only functions referenced by the flow, skips unchanged sources, and defaults to a dry run with no remote uploads. Applying changed functions updates their shared deployment, so other flows using them see the update too. Source archives are private `Code` records; no new system type is required. `flow_pull` restores original sources and reports source-download errors separately. Legacy functions without stored sources appear in the inventory but cannot have their original source reconstructed. Runtime changes require a new function name.

## Runtime model

- A matching trigger seeds a `state` output map under the trigger key.
- Each step reads `state`, runs, stores its result under its step name, then advances.
- Steps run in array order within their sequence unless `next` chooses a later step in that sequence. Branches and loops own nested sequences; their rules are below.
- `skipIf` skips a step, including its contents for a branch or loop. `continueIf` stops the current sequence when falsy.
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

Ordinary factories follow `flowBuilder.step.<type>(name, { params, description?, next?, skipIf?, continueIf? })`. Control steps use the nested forms below.

| Category | Steps |
|---|---|
| Write | `createType`, `batchCreateType`, `updateType`, `deleteType` |
| Read | `type`, `batchGetType`, `listType`, `listTypeByParent` |
| Logic/integration | `logic`, `httpRequest`, `cloudFunction`, `notifyWebSocket`, `googleSearch`, `linkScraper`, `htmlToPdf`, `extractImageMetadata` |
| Control | `branch`, `forEach` |
| AI/media | `genericAI`, `elevenLabsTTS` |

The result of each step is stored at `state.<stepName>`. Consult the scaffold typings and validation errors for the exact parameter shape of the selected step.

`genericAI` operations include `generateText`, `generateObject`, `streamText`, `embed`, and `generateImage`. With `streamText`, downstream steps run for published stream chunks and the final result; each chunk contains the accumulated text. This is useful for updating one draft record throughout generation.

## Branches

Use `flowBuilder.step.branch(name, { branches: [{ when, steps }], description?, skipIf?, continueIf? })`. `when` accepts a boolean or `flowBuilder.js(...)` returning a boolean. A JavaScript condition here needs the `js` marker; a plain code string is invalid.

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Matching paths demo",
  triggers: { manual: flowBuilder.trigger.manual() },
  steps: [
    flowBuilder.step.logic("input", {
      params: { needsShipping: true, needsInvoice: true },
    }),
    flowBuilder.step.branch("route", {
      branches: [
        {
          when: flowBuilder.js("state.input.needsShipping === true"),
          steps: [
            flowBuilder.step.logic("ship", { params: { action: "ship" } }),
            flowBuilder.step.logic("shippingDone", {
              params: { action: flowBuilder.js("state.ship.action") },
            }),
          ],
        },
        {
          when: flowBuilder.js("state.input.needsInvoice === true"),
          steps: [
            flowBuilder.step.logic("invoice", { params: { action: "invoice" } }),
          ],
        },
      ],
    }),
  ],
});
```

- Every matching path runs concurrently, so both paths above run. Several conditions may match. No matches ends the current sequence.
- Each path must contain at least one step and receives its own copy of the accumulated state. Outputs from sibling paths are not shared.
- A branch must be the last step in its containing sequence. It has no `next` option or shared continuation after its paths. Put continuation steps inside each path.
- The branch output is `{ matchedSteps: string[] }`, containing the first step name of each matching path. It does not collect the paths' final outputs.

## For each

Use `flowBuilder.step.forEach(name, { items, steps, description?, next?, skipIf?, continueIf? })`. `items` is a static array or `flowBuilder.js(...)` returning an array. `steps` is the sequence run for each item; it may contain nested loops and branches.

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Nested map demo",
  triggers: { manual: flowBuilder.trigger.manual() },
  steps: [
    flowBuilder.step.forEach("loop0", {
      items: [1, 2, 3],
      steps: [
        flowBuilder.step.forEach("inner", {
          items: flowBuilder.js("[10, 100]"),
          steps: [
            flowBuilder.step.logic("multiply", {
              params: {
                value: flowBuilder.js("state.loop0.item * state.inner.item"),
                index: flowBuilder.js("state.inner.index"),
              },
            }),
          ],
        }),
        flowBuilder.step.logic("iterationResult", {
          params: {
            number: flowBuilder.js("state.loop0.item"),
            products: flowBuilder.js("state.inner.result"),
          },
        }),
      ],
    }),
    flowBuilder.step.logic("afterLoop", {
      params: { results: flowBuilder.js("state.loop0.result") },
    }),
  ],
});
```

- Inside an iteration, `state.<loopName>.item` is the current value and `.index` is its zero-based position. Each iteration has its own state; outer loop items remain available in nested loops by the outer loop name.
- Each loop runs up to 10 iterations concurrently. A step after the loop runs once, after its iterations succeed.
- After completion, `state.<loopName>` is `{ items, result }`. `result` has the input array's length and order, and each entry is the output of that iteration's last step, like JavaScript `map`. Body step outputs are local to the iteration; use the loop's `result` after it.
- An empty input produces `result: []` and continues. An empty body produces one `null` per input item; an iteration that returns no output also contributes `null`. If the last body step is a branch, its output is `{ matchedSteps }`, not an aggregation of path outputs.

### Structure and limits

- Step names are unique across the entire flow, including every path and nested loop. `next` links must stay within the same sequence; never link into another path, into a loop body from outside, or back to an earlier step.
- Author nested `branches` and `steps` in the DSL. The compiled JSON declaration keeps one flat `steps` map: branches point to path heads through `params.branches`, loops point to their body head through `params.bodyStep`, and a loop's `nextStep` points to its continuation. `flow_pull` reconstructs the nested DSL; `detachedSteps` is for preserving existing unconnected steps, not for authoring loop bodies or paths.
- A flow run permits **1,000 total loop iterations**, shared across all loops, nested loops, and matching paths. Outer iterations count too: a 10-item outer loop with a 10-item inner loop in every iteration uses 110 iterations.
- Static arrays over 1,000 items fail DSL/push validation. Dynamic arrays are checked after evaluation at runtime. Nested totals are enforced by a shared runtime counter as loops are reached; validation does not precompute every nested or dynamic expansion.
- A separate budget of **2,500 durable operations per execution** also applies. Steps, iteration/branch contexts, and retries consume it, so complex bodies can reach this budget before the iteration limit. A limit failure stops further work; calls already in flight may finish. Use `debug: true` and inspect `step-error` logs when investigating a limit failure.

## Image generation and editing

Use `genericAI` with `operation: "generateImage"`.

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

- missing targets or cycles through `next`, branch paths, or loop bodies
- shared steps or links crossing branch/loop sequence boundaries
- duplicate step names and oversized static loop inputs
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
