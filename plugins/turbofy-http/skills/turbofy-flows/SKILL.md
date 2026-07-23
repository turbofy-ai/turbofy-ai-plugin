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
- On match, start at `startStep` with output map `{ [triggerKey]: triggerData }`. MODIFY → `{ old, new }`; INSERT/REMOVE → the record; MANUAL → `{}`; SCHEDULE → `{ scheduledTime }` (ISO timestamp of the scheduled firing time).
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

### Trigger variants

- `TABLE_DATA_CHANGE` — `{ variant, name, tableId, operation: "INSERT"|"MODIFY"|"REMOVE", condition? }`. Trigger data: the record (INSERT/REMOVE) or `{ old, new }` (MODIFY).
- `MANUAL` — `{ variant, name }`. Trigger data: `{}`.
- `SCHEDULE` — recurring run via AWS EventBridge Scheduler (see below).

#### SCHEDULE trigger

```json
{
  "variant": "SCHEDULE",
  "name": "nightly",
  "cron": "0 9 * * ? *",
  "timezone": "Europe/Berlin"
}
```

Fields (copy the exact shape from an existing `flow_get` / confirm via dry-run):

- Exactly **one of** `cron` or `rate` — never both.
  - `cron`: AWS **6-field** cron (see below).
  - `rate`: `{ "value": <positive integer>, "unit": "minutes" | "hours" | "days" }` — for simple fixed intervals, e.g. every 10 minutes → `{ "value": 10, "unit": "minutes" }`.
- `timezone`: IANA tz (e.g. `"Europe/Berlin"`), **only valid with `cron`**; defaults to UTC.
- `startDate` / `endDate`: ISO 8601 date or datetime. Won't fire before `startDate` (date-only = midnight UTC); stops after `endDate`, which must be after `startDate`.
- `condition`: optional JS string, same as other triggers.

Trigger data is `{ scheduledTime }` — the ISO timestamp of the *scheduled* (not actual) firing time; reference it via `state.<triggerKey>.scheduledTime`.

**AWS 6-field cron** — fields are `minutes hours day-of-month month day-of-week year` (NOT the standard 5-field crontab):

- Exactly one of day-of-month / day-of-week must be `?` when the other is specified.
- Day-of-week is `1`–`7` (1 = Sunday) or `SUN`–`SAT`; month is `1`–`12` or `JAN`–`DEC`.
- Examples: daily 09:00 → `"0 9 * * ? *"` · every Monday noon → `"0 12 ? * MON *"` · 1st & 15th at 08:30 → `"30 8 1,15 * ? *"`.

**Lifecycle:** schedules reconcile automatically on upsert (DynamoDB stream → EventBridge Scheduler) — no extra MCP steps. Disabling the flow disables its schedules; deleting the flow or trigger deletes them; a schedule past its `endDate` is removed and never fires again.

**Web-editor interplay:** the visual schedule builder only represents a subset of cron — a single minute, an explicit list of hours, and either any-day / specific weekdays / specific days-of-month (months and year must be `*`). Expressions outside that subset (ranges `9-17`, steps `0/5`, `L`/`W`/`#` markers, `*` hours, month/year restrictions) still validate and run, but show as a read-only "advanced schedule". Prefer `rate` for simple intervals and builder-representable crons (e.g. `"30 8 ? * MON,FRI *"`) so users can keep editing visually.

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
- **Schedule triggers:** both `cron` and `rate` set; malformed cron; `timezone` without `cron`, or an unknown tz; `rate.value` not a positive integer; unparseable `startDate`/`endDate`; `endDate` ≤ `startDate`. A `SCHEDULE` trigger with neither `cron` nor `rate` is only a **warning** — it round-trips but never fires.

Always dry-run first.

---

## See also

- `turbofy-platform` — discovery, `workspace_get`, `data_*`, schema.
- `turbofy-dynamic-fields` — different server JS surface (`$$std`); flows use `state`, not `$$std`.
