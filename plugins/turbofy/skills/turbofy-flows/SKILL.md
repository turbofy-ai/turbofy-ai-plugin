---
name: turbofy-flows
description: Use when building or modifying Turbofy flows (workspace automations) via the Turbofy MCP — creating a flow with Turbofy_flow_init, pulling flows locally with Turbofy_flow_pull, editing flow.ts with the flowBuilder DSL (triggers, steps, dynamic js params, secrets), validating (circular-flow rules, npm run validate), or pushing with Turbofy_flow_push. Loads the flow runtime mental model (trigger → linked list of steps over SNS+Lambda), the file layout (~/.turbofy/workspaces/<env>/<wsId>/flows/<flowId>/), signatures for all trigger variants and all 19 step types, the `state` output context for js() params, the secret() rules, and the recursion/validation rules. For workspace/schema work load `turbofy-platform`; for apps load `turbofy-apps`.
---

# Turbofy Flows

This skill covers building and modifying Turbofy flows — event-driven automations that run in a workspace. It documents the `Turbofy_flow_*` workflow, the `flowBuilder` DSL, all trigger and step signatures, dynamic parameters, secrets, and the validation rules.

For platform-level concerns (org/workspace discovery, environments, the data schema, core MCP rules), see `turbofy-platform`.

---

## 1) What are flows?

A flow is a workspace automation: **triggers** that decide when it runs, and a **linked list of steps** executed in order. Flows are stored as a JSON declaration and executed server-side via SNS topics + Lambda functions — one step at a time, each step publishing its output and the name of the next step.

Execution model (what actually happens at runtime):

- A DynamoDB stream event (record INSERT / MODIFY / REMOVE) is matched against every flow's `TABLE_DATA_CHANGE` triggers — by `tableId` (case-insensitive) and operation, then by the optional JS `condition` evaluated against the trigger data.
- On match, execution starts at `startStep` with an **output map** seeded as `{ [triggerKey]: triggerData }`. For MODIFY the trigger data is `{ old, new }`; for INSERT/REMOVE it is the record itself. Manual triggers seed `{}`; schedule triggers seed `{ scheduledTime }` (the ISO timestamp of the scheduled firing time).
- Each step resolves its params (static values, `js()` code, secrets), runs, and appends its result to the output map under its own step name. Then execution moves to the step's `nextStep` — strictly linear, no branching.
- `skipIf` (skip this step, keep going) and `continueIf` (stop the whole flow when falsy) are JS conditions evaluated against the output map.
- Flow runs and per-step logs are recorded as FlowRun / FlowRunLog records; `debug: true` enables verbose step logging. `disabled: true` turns the flow off.

**Steps are a linked list, not a graph.** A step must never point back at itself or an earlier step, and a flow must never (transitively) retrigger itself — validation enforces this (section 7).

## 2) Workflow & file layout

Tools:

| Tool | Purpose |
|---|---|
| `Turbofy_flow_pull` | Pull **all** flows of a workspace to local files. Also returns the workspace's secret records (id + name). |
| `Turbofy_flow_init` | Create a new empty flow remotely, then pull all flows. |
| `Turbofy_flow_push` | Push **one** flow (`flowId`). `dryRun` defaults to `true` — always review the dry run before applying with `dryRun: false`. |

File layout:

```
~/.turbofy/workspaces/<environment>/<workspaceId>/flows/
├── package.json        # shared — depends on @turbofy-ai/app-runtime
├── tsconfig.json       # shared
├── validate.ts         # shared — validates ALL flows: npm run validate
└── <flowId>/
    ├── flow.ts         # the editable flowBuilder source (exports `flow`)
    └── flow.base.json  # pull baseline — never edit; used for diff/conflict detection
```

The loop: `Turbofy_flow_pull` → edit `<flowId>/flow.ts` → `npm run validate` (inside the flows dir, optional but cheap) → `Turbofy_flow_push { flowId }` (dry run) → review changes/validation → `Turbofy_flow_push { flowId, dryRun: false }`.

Rules:

- `flow.ts` must `export const flow = flowBuilder.buildFlow({...})`.
- Never edit `flow.base.json`. If push reports a **conflict** (remote changed since pull), run `Turbofy_flow_pull` and re-apply your edits.
- Push validates the flow against **all** remote flows (cross-flow recursion), checks that referenced secrets exist, and blocks on errors even with `dryRun: false`. Warnings never block.

## 3) flowBuilder reference

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Order fulfillment",          // flow record name (not part of the declaration)
  triggers: {
    onOrderCreated: flowBuilder.trigger.tableDataChange({
      tableId: "order",               // table name, matched case-insensitively
      operation: "INSERT",            // "INSERT" | "MODIFY" | "REMOVE"
      condition: "state.new !== undefined || state.status === 'paid'", // optional JS, evaluated against the trigger data (NOT the output map)
    }),
    manual: flowBuilder.trigger.manual(),      // fired from the dashboard / flowtrigger record
    nightly: flowBuilder.trigger.schedule({    // recurring schedule (see "Trigger variants")
      cron: "0 9 * * ? *",                     // daily at 09:00
      timezone: "Europe/Berlin",
    }),
  },
  steps: [
    // executed in array order; each step's nextStep = the following entry
    flowBuilder.step.httpRequest("callWebhook", {
      description: "Notify ERP",
      params: {
        httpMethod: "POST",
        url: "https://erp.example.com/hook",
        headers: { Authorization: flowBuilder.secret("<secret record id>") },
        body: flowBuilder.js("JSON.stringify(state.onOrderCreated)"),
      },
      skipIf: "state.onOrderCreated.test === true",       // JS: truthy → skip this step, continue flow
      continueIf: "state.callWebhook.type === 'success'", // JS: falsy → stop the flow after this step
      // next: "someLaterStep" — optional explicit override of the array order
    }),
    flowBuilder.step.createType("logResult", {
      params: {
        ofType: "webhooklog",
        fields: flowBuilder.js("({ status: state.callWebhook.status })"),
      },
    }),
  ],
  debug: false,
  disabled: false,
});
```

- **Trigger key = output key.** The key you choose in `triggers` (e.g. `onOrderCreated`) is the key under which the trigger data appears in `state`.
- **Step name = output key.** The first argument of every step factory is both the step's identity and its key in `state` after it runs.
- **Chaining**: array order defines execution order. `next: "<stepName>"` overrides it; `next: null` ends the flow early. `detachedSteps: [...]` exists only for round-tripping steps unreachable from `startStep` — don't use it for new flows.
- Conditions (`skipIf`, `continueIf`, trigger `condition`) are **plain JS code strings** (js is the only supported logic language; json-logic is legacy and not supported by the DSL).

### Trigger variants

Three trigger factories. The key you give a trigger in `triggers` is its output key in `state`; `condition` is a plain-JS string evaluated against the trigger data (not the output map).

- `flowBuilder.trigger.tableDataChange({ tableId, operation, condition? })` — fires on a record change. `operation` is `"INSERT" | "MODIFY" | "REMOVE"`. Trigger data: the record (INSERT/REMOVE) or `{ old, new }` (MODIFY).
- `flowBuilder.trigger.manual()` — fired from the dashboard or a `flowtrigger` record. Trigger data: `{}`.
- `flowBuilder.trigger.schedule(opts?)` — runs the flow on a recurring schedule via AWS EventBridge Scheduler (see below).

#### schedule(opts?)

```ts
flowBuilder.trigger.schedule({
  // exactly one of cron / rate — never both:
  cron?: string;              // AWS 6-field cron (see below)
  rate?: { value: number; unit: "minutes" | "hours" | "days" }; // value = positive integer
  timezone?: string;          // IANA tz e.g. "Europe/Berlin"; ONLY valid with cron; defaults to UTC
  startDate?: string;         // ISO 8601 date/datetime; does not fire before it (date-only = midnight UTC)
  endDate?: string;           // ISO 8601 date/datetime; stops firing after it; must be after startDate
  condition?: string;         // same JS condition support as other triggers
})
```

Trigger data is `{ scheduledTime: string }` — the ISO timestamp of the *scheduled* (not actual) firing time; reference it via `state.<triggerKey>.scheduledTime`.

**AWS 6-field cron** — fields are `minutes hours day-of-month month day-of-week year` (NOT the standard 5-field crontab):

- Exactly one of day-of-month / day-of-week must be `?` when the other is specified.
- Day-of-week is `1`–`7` (1 = Sunday) or `SUN`–`SAT`; month is `1`–`12` or `JAN`–`DEC`.
- Examples: daily 09:00 → `"0 9 * * ? *"` · every Monday noon → `"0 12 ? * MON *"` · 1st & 15th at 08:30 → `"30 8 1,15 * ? *"`.
- For simple fixed intervals prefer `rate`: every 10 minutes → `rate: { value: 10, unit: "minutes" }`.

**Validation** (`npm run validate` / push):

- Error: both `cron` and `rate` set; malformed cron; `timezone` set without `cron`, or an unknown tz; `rate.value` not a positive integer; `startDate`/`endDate` that don't parse as ISO dates; `endDate` ≤ `startDate`.
- Warning (round-trips fine but never fires): a `schedule()` with neither `cron` nor `rate`.

**Runtime & lifecycle:**

- Schedules reconcile **automatically** on push (DynamoDB stream → EventBridge Scheduler) — no extra MCP/CLI steps. Disabling the flow disables its schedules; deleting the flow or the trigger deletes them; a schedule whose `endDate` has passed is removed and never fires again.
- Requires `@turbofy-ai/app-runtime` **> 0.0.8** in the pulled flows project for the new typings (`startDate`/`endDate`, etc.). If validate/build complains about unknown schedule options, bump this dependency.

**Web-editor interplay:** the visual schedule builder can only represent a subset of cron — a single minute, an explicit list of hours, and either any-day / specific weekdays / specific days-of-month (months and year must be `*`). Expressions outside that subset (ranges like `9-17`, steps like `0/5`, `L`/`W`/`#` markers, `*` hours, month/year restrictions) still validate and run, but appear in the UI as a read-only "advanced schedule". Prefer `rate` for simple intervals and builder-representable crons (e.g. `"30 8 ? * MON,FRI *"`) when you can, so users can keep editing the schedule visually.

## 4) Dynamic params: js() and the state context

Any param value (at any nesting level of an object) can be one of:

- a **static value** — stored verbatim in the declaration,
- `flowBuilder.js("<code>")` — JS executed server-side at step-resolution time; the last expression is the value. `state` is the accumulated output map keyed by trigger/step names: `state.onOrderCreated` (trigger data — `{ old, new }` for MODIFY triggers), `state.callWebhook` (a previous step's result).
- `flowBuilder.secret("<secret record id>")` — see section 6. Only for credential-holding string params.

Notes:

- To return an object literal from `js()`, wrap it in parentheses: `"({ a: 1 })"`.
- Markers inside **arrays** are not supported (the runtime cannot address paths inside arrays) — wrap the whole array: `flowBuilder.js("[state.a.id, state.b.id]")`.
- Static **objects and arrays** are fine as params; the builder emits the right config so they pass through verbatim.

## 5) Step catalog (all 19 step types)

Every factory has the signature `flowBuilder.step.<type>(name, { description?, params, next?, skipIf?, continueIf? })`. Params below are the static shapes; any leaf can be `js()` instead. The step's **result** is what lands in `state.<stepName>`.

**Write steps** (create/modify/delete workspace records — these can trigger other flows' `TABLE_DATA_CHANGE` triggers, see section 7):

| Step | Fires | Params | Result |
|---|---|---|---|
| `createType` | INSERT | `{ ofType, fields?, id?, namespace?, directives?, connections?, dynamicArgs? }` | created record `{ id, createdAt, updatedAt, ofType, fields, ... }` |
| `batchCreateType` | INSERT | `{ ofType, namespace, input: [{ id, fields?, directives?, connections? }], dynamicArgs? }` | record map keyed by id |
| `updateType` | MODIFY | `{ id, ofType, fields?, connections?, namespace?, directives?, dynamicArgs? }` | updated record |
| `deleteType` | REMOVE | `{ id, ofType, namespace?, dynamicArgs? }` | deleted record |

**Read steps:**

| Step | Params | Result |
|---|---|---|
| `type` | `{ id, ofType, namespace?, dynamicArgs? }` | record |
| `batchGetType` | `{ ids: string[], ofType, namespace?, dynamicArgs? }` | record map keyed by id |
| `listType` | `{ ofType, limit?, nextToken?, sortOrder?, namespace?, dynamicArgs? }` | `{ items: record[], nextToken }` |
| `listTypeByParent` | `{ ofType, parentType, parentId, parentKey: "FIRST"\|"SECOND", limit?, nextToken?, sortOrder?, namespace?, dynamicArgs? }` | `{ items: record[], nextToken }` |

**Logic & integration steps:**

| Step | Params | Result |
|---|---|---|
| `logic` | free-form record — each key is typically a `js()` expression | the resolved params object (use it to compute/reshape values for later steps) |
| `httpRequest` | GET/DELETE: `{ httpMethod, url, parameters?, headers?, contentMediaType? }`; POST/PUT/PATCH additionally `{ body }` (string) | `{ type: "success", status?, headers?, body?, statusText? }` \| `{ type: "error", content }` |
| `cloudFunction` | `{ cloudFunctionUrl, ...anything }` — invokes a Turbofy cloud function with the params | same shape as `httpRequest` result |
| `notifyWebSocket` | `{ record?: { id?, ofType?, namespace?, connections? }, operation?: "INSERT"\|"UPDATE"\|"DELETE" }` | `{}` — pushes the record event to subscribed WebSocket clients |
| `googleSearch` | `{ searchTerms, apiKey?, searchEngineId?, lang?, numOfResults?, searchType?: "image", siteSearch?, siteSearchFilter?, dateRestrict? }` | array of `{ title, link, snippet, ... }` (or image results) |
| `linkScraper` | `{ websiteUrl, selectors?, resultType?: "text"\|"html", numOfResults? }` | `{ [selector]: string[] }` |
| `htmlToPdf` | `{ htmlString, outputFilename?, format?: "a4"\|..., margin?: { top?, right?, bottom?, left? } }` | `{ type: "success", url, key, mimeType }` \| `{ type: "error", value? }` |
| `extractImageMetadata` | `{ url? }` | EXIF/IPTC metadata record (`Image Width`, `Image Height`, `FileType`, ...) |

**AI steps:**

| Step | Params | Result |
|---|---|---|
| `genericAI` | discriminated by `operation`: `"generateObject"` `{ apiKey, model: { provider: "openai"\|"anthropic"\|"mistral"\|"cohere"\|"google", model }, prompt, schema, temperature?, providerOptions? }` · `"generateText"`/`"streamText"` `{ apiKey, model, prompt? or messages?, system?, tools, toolChoice?, temperature? }` · `"embed"` `{ apiKey, model, value }` | `generateText` → `{ type: "result", value: { text, toolCalls } }` · `generateObject` → `{ type: "objectResult", value }` · `streamText` → repeated `{ type: "streamChunk", value: "<accumulated text>" }` then `{ type: "streamResult", value: { text, toolCalls } }` · `embed` → `{ type: "embeddingResult", value }` · errors → `{ type: "error", error }` |
| `openAIImageGeneration` | `{ model: "dall-e-3"\|"dall-e-2", prompt, size?, quality?, style?, n?, outputFilename?, apiKey? }` | `{ type: "result", url, key, mimeType }` \| `{ type: "error", value }` |
| `elevenLabsTTS` | `{ text, model?, voiceId?, stream?, outputFormat?, apiKey?, stability?, similarityBoost?, style? }` | `{ type: "success", url, key, mimeType, durationSeconds, characterStartTimes }` \| `{ type: "stream", ... }` \| `{ type: "error", value? }` |

`apiKey` params (`genericAI`, `openAIImageGeneration`, `elevenLabsTTS`, `googleSearch`) are credentials: they **must** be `flowBuilder.secret(...)` (or `js()`), never a plain string — the build fails otherwise.

For `streamText`, every step **after** the AI step runs once per published `streamChunk` (throttled; `value` is the accumulated text so far) and once more for the final `streamResult` — use this to stream into a record via `updateType`. The canonical chatbot pattern (create draft message → `streamText` → `updateType` per chunk) is in `turbofy-chatbot`.

## 6) Secrets

Secrets are workspace records of type `secret` holding only a **name**; the value lives in SSM and is never readable through any API.

- **Discover** secrets with `Turbofy_data_list { ofType: "secret" }` (returns id + name), or from the `secrets` array in the `Turbofy_flow_pull` result. To create one, the user must add it in the dashboard (Workspace → Secrets) — the value cannot be set via MCP.
- **Reference** a secret with `flowBuilder.secret("<secret record id>")` — the id is a UUID. The declaration stores only the id; the runtime resolves the value just-in-time.
- **Never** paste a secret value (API key, token, password) into `flow.ts` or any param. `secret()` rejects non-UUID input, and push errors on secret ids that don't exist in the workspace.

## 7) Validation & recursion rules

`buildFlow()` validates on compile; `Turbofy_flow_push` additionally validates across all remote flows; `npm run validate` (in the flows dir) validates all local flows. **Errors block push; warnings don't.**

Errors:

- `startStep`/`nextStep` referencing a missing step; a step revisited by the chain (self- or backward-pointing `next`) — steps must form a linked list.
- **Self-retrigger**: a write step whose `ofType` matches one of the flow's own `TABLE_DATA_CHANGE` triggers (same table + matching operation: create→INSERT, update→MODIFY, delete→REMOVE). With a trigger `condition` this downgrades to a warning — make sure the condition provably terminates the recursion (e.g. the write sets a flag the condition excludes).
- **Cross-flow cycles**: flow A writes a table that triggers flow B, whose writes (transitively) trigger flow A. The error spells out the full chain. Fix by breaking the cycle: write to a different table, narrow the trigger operation, or disable one link.
- Secret params that don't hold a record id; credential params passed as plain strings; secret ids that don't exist in the workspace.

Warnings:

- Steps unreachable from `startStep`; `startStep: null` with declared steps.
- Write steps with a **dynamic** `ofType` and any `cloudFunction` step — recursion can't be statically verified there; re-check these manually.
- Step params that don't match the step's schema (best-effort check; dynamic params are exempt).

## See also

- `turbofy-platform` — org/workspace discovery, environments, core MCP rules, schema workflow.
- `turbofy-dynamic-fields` — the other server-side JS surface ($$std API for dynamic fields); note flows use `state` (output map), not `$$std`.
- `turbofy-chatbot` — the chatbot recipe built on flows (Message INSERT trigger → `genericAI` → assistant Message write). Turbofy has **no chatbot API**.
