---
name: turbofy-flows
description: "Use when building or modifying Turbofy flows (workspace automations) via the Turbofy MCP — listing flows, fetching a JSON declaration, or creating/updating with flow_upsert (dryRun). Covers the runtime mental model (trigger → linked list of steps), JSON declaration shape (triggers, startStep, steps, paramsConfig for js/secret markers), step types, secrets, and recursion/validation rules. No local flow.ts / flowBuilder DSL. For workspace/schema work load turbofy-platform; for apps load turbofy-apps."
disable-model-invocation: false
---

# Turbofy Flows

Event-driven workspace automations via the HTTP MCP. Declarations are **JSON** — there is no local `flow.ts`, no `flowBuilder` DSL, and no `~/.turbofy` checkout.

For org/workspace discovery and schema see `turbofy-platform`.

---

## 1) What are flows?

A flow is: **triggers** (when it runs) + a **linked list of steps** (what runs). Stored as a JSON declaration; executed server-side (SNS + Lambda) one step at a time.

Runtime:

- DynamoDB stream events match `TABLE_DATA_CHANGE` triggers by `tableId` (case-insensitive) + operation, then optional JS `condition` against trigger data.
- On match, start at `startStep` with output map `{ [triggerKey]: triggerData }`. MODIFY → `{ old, new }`; INSERT/REMOVE → the record; MANUAL → `{}`.
- Each step resolves params (static / JS / secret), runs, stores result under its step name, then follows `nextStep`.
- Per-step skip/continue conditions are JS against the output map.
- `debug: true` → verbose logs; `disabled: true` → off.

**Steps are a linked list, not a graph.** No cycles; no transitive self-retrigger across flows (validated on upsert).

---

## 2) MCP workflow

| Tool | Purpose |
|---|---|
| `flow_list` | Summaries: `flowId`, name, trigger/step counts, `disabled`, `createdAt` / `updatedAt` |
| `flow_get` | Full declaration JSON |
| `flow_upsert` | Create or update from declaration. **`dryRun` defaults to `true`**. Omit `flowId` to create (pass `name`). Set `dryRun: false` to apply. |

Always: `flow_get` (or start fresh) → edit declaration → `flow_upsert` dry-run → review → `flow_upsert` with `dryRun: false`.

---

## 3) Declaration shape

```json
{
  "triggers": {
    "onSignal": {
      "variant": "TABLE_DATA_CHANGE",
      "name": "onSignal",
      "tableId": "wgluy5",
      "operation": "INSERT"
    }
  },
  "startStep": "ackSignal",
  "steps": {
    "ackSignal": {
      "id": "ackSignal",
      "name": "ackSignal",
      "type": "updateType",
      "description": "Acknowledge the signal",
      "nextStep": "logActivity",
      "params": {
        "id": "state.onSignal.id",
        "ofType": "wgluy5",
        "fields": "({ status: 'Acknowledged', ackNote: 'auto' })"
      },
      "paramsConfig": {
        "id": { "language": "js", "type": "logic" },
        "fields": { "language": "js", "type": "logic" }
      },
      "skipCondition": "false;",
      "skipConditionLanguage": "js",
      "continueCondition": "true;",
      "continueConditionLanguage": "js"
    },
    "logActivity": {
      "id": "logActivity",
      "name": "logActivity",
      "type": "createType",
      "nextStep": null,
      "params": {
        "ofType": "ppkiqq",
        "fields": "({ text: state.onSignal.message, signalId: state.onSignal.id })"
      },
      "paramsConfig": {
        "fields": { "language": "js", "type": "logic" }
      },
      "skipCondition": "false;",
      "skipConditionLanguage": "js",
      "continueCondition": "true;",
      "continueConditionLanguage": "js"
    }
  },
  "debug": true,
  "disabled": false
}
```

Notes:

- **Trigger key = `state` key.** Step **`name`/`id` = `state` key** after the step runs.
- Prefer starting from `flow_get` and editing, so `paramsConfig` markers stay correct.
- Dynamic JS params: put the code string in `params` and mark the path in `paramsConfig` with `{ "language": "js", "type": "logic" }`.
- Object literals from JS must be wrapped: `"({ a: 1 })"`.
- `tableId` / `ofType` are often normalized to **lowercase** in stored declarations — match what `flow_get` returns; compare case-insensitively to `workspace_get` → `schema.types[].id`.
- Trigger variants are **uppercase**: `TABLE_DATA_CHANGE`, `MANUAL`, `SCHEDULE` — lowercase values won't match at runtime (see existing flows / dry-run errors for exact fields).

### `flow_upsert` call

```json
{
  "orgId": "…",
  "workspaceId": "…",
  "flowId": "…",
  "name": "Signal Autoresponder",
  "declaration": { "triggers": {}, "startStep": "…", "steps": {} },
  "dryRun": true
}
```

Omit `flowId` when creating; `name` is required on create.

---

## 4) Step catalog

Write steps (can fire other flows' `TABLE_DATA_CHANGE` triggers):

| `type` | Fires | Typical `params` |
|---|---|---|
| `createType` | INSERT | `{ ofType, fields?, id?, … }` |
| `batchCreateType` | INSERT | `{ ofType, namespace, input: […] }` |
| `updateType` | MODIFY | `{ id, ofType, fields?, … }` |
| `deleteType` | REMOVE | `{ id, ofType, … }` |

Read steps: `type`, `batchGetType`, `listType`, `listTypeByParent`.

Logic & integration: `logic`, `httpRequest`, `cloudFunction`, `notifyWebSocket`, `googleSearch`, `linkScraper`, `htmlToPdf`, `extractImageMetadata`.

AI: `genericAI`, `openAIImageGeneration`, `elevenLabsTTS`.

Credential params (`apiKey`, auth headers, etc.) should use **secret references**, never plaintext. The validator only enforces this for `apiKey` on `genericAI` / `googleSearch` / `openAIImageGeneration` / `elevenLabsTTS` — a plaintext auth header on `httpRequest` passes validation, so keep it secret-marked by convention. Prefer copying the secret marker shape from an existing `flow_get` in the workspace, or use the dry-run validator.

---

## 5) Secrets

- Discover: `data_list` with `ofType: "secret"` (id + name only).
- Values live in SSM — never readable via MCP. Users create secrets in the dashboard.
- Never paste API keys into declarations.

---

## 6) Validation & recursion

`flow_upsert` validates the declaration (including cross-flow cycles and secret refs). **Errors block apply; warnings do not.**

Errors include:

- Missing `startStep` / `nextStep` targets; cycles in the step chain.
- **Self-retrigger:** a write step whose `ofType` matches the flow's own `TABLE_DATA_CHANGE` trigger (same table + matching operation). Any non-empty trigger `condition` downgrades this to a warning (presence is checked, not whether it actually stops recursion — make sure it does).
- **Cross-flow cycles** between flows.
- Invalid / missing secret ids; plaintext values in the enforced `apiKey` params.

Always dry-run first.

---

## See also

- `turbofy-platform` — discovery, `workspace_get`, `data_*`, schema.
- `turbofy-dynamic-fields` — different server JS surface (`$$std`); flows use `state`, not `$$std`.
