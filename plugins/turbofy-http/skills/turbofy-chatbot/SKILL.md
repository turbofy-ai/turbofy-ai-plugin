---
name: turbofy-chatbot
description: "Use when building a chatbot, AI assistant, or any conversational feature in a Turbofy workspace/app — 'add a chatbot', 'AI support chat', 'chat with my data', 'assistant widget' — or when a workspace contains Thread/Message tables or a product_chatbot flag and the task touches chat. CRITICAL: Turbofy has NO chatbot API, endpoint, or SDK. A chatbot is assembled from three ordinary primitives: Thread/Message tables (schema), a flow triggered on Message INSERT that calls genericAI and writes the assistant reply, and a chat UI block. product_chatbot is a legacy flag — ignore it. Loads the full recipe: schema, flow JSON (incl. streaming), chat block, and the WebSocket delivery model. Companions: turbofy-platform (schema), turbofy-flows (declaration details), turbofy-blocks (UI rules)."
disable-model-invocation: false
---

# Turbofy Chatbot (tables + flow + UI block)

How to build a chatbot on Turbofy. Read section 1 first — it corrects a common false assumption.

## 1) There is no chatbot API

**Turbofy has no chatbot API, no chat endpoint, and no chat SDK.** Do not search for chatbot documentation, do not invent HTTP calls against a guessed interface, and do not report being blocked on "missing chatbot docs". Everything a chatbot needs is composed from primitives you already have:

| Piece | What it is | Skill |
|---|---|---|
| **Tables** | `Thread` + `Message` (+ optional `Agent` for prompt config) — plain workspace tables | `turbofy-platform` |
| **Flow** | Triggered on `Message` INSERT (`role === 'user'`), calls `genericAI`, writes the assistant `Message` | `turbofy-flows` |
| **UI block** | Sends by **creating a Message record**; receives via WebSocket events on the Message table | `turbofy-blocks` |

Two things that look like a chatbot API but aren't:

- **`product_chatbot`** (in the schema's `products` array) is a **legacy flag from a retired dashboard template**. It is outdated and has no runtime effect relevant to app building. Never gate work on it, never add it to new schemas, and never treat its presence as evidence of a chat API.
- **Existing `Thread` / `Message` tables** are ordinary workspace tables — usually left over from that same template or from a previous chatbot build. Reuse or extend them like any other tables.

## 2) Architecture — the message loop

```
Chat block (client)                Workspace data                     Flow (server)
──────────────────                 ──────────────                     ─────────────
useCreateType ────── INSERT ─────▶ Message { role: "user" } ────────▶ TABLE_DATA_CHANGE trigger
                                                                       (condition: role === 'user')
                                                                        │ listTypeByParent → history
                                                                        │ genericAI → completion
useWsSubscription ◀── WS event ─── Message { role: "assistant" } ◀──── createType / updateType
  └─ refetch()
```

- **Sending a message = creating a record.** **Receiving a reply = the flow writing a record.**
- Every record INSERT/MODIFY/REMOVE on a workspace table is **automatically pushed to subscribed WebSocket clients** (respecting the end-user read-access policy). The chat block subscribes with `useWsSubscription` — no `notifyWebSocket` flow step is needed for this pattern. The WebSocket requires a logged-in user with a valid token, so this only works on private (authenticated) pages — see the Gotchas.

## 3) Schema

Edit the schema JSON (`workspace_get` → mutate → `workspace_schema_push`, dry-run first; see `turbofy-platform`). Add:

- **Enum** `Role` with values `["user", "assistant"]`.
- **Thread** table — optionally `parents.first` → the `@users` table so each signed-in user owns their threads.
- **Message** table — fields `content` (`String`), `role` (`Role`), `isComplete` (`Boolean`, only needed for streaming); `parents.first` → Thread.

Notes:

- The parent link puts a `threadId` field on `Message` records (child field is the camelCase parent name + `Id`). If you're reusing existing tables, confirm the actual field name in `workspace_get` output.
- For a logged-in chatbot: the signed-in user's record ID in the `@users` table equals `user.sub` from `useCurrentUser()` (`@/lib/auth`) — use it as the Thread's user parent value.
- `content` as `String` is enough for plain text. Use `Json` if you later need structured content (tool calls, attachments).
- An optional `Agent` table (`name`, `systemPrompt`) lets users edit the prompt as data instead of hardcoding it in the flow.

## 4) Flow — the bot's brain

Create via `flow_upsert` (dry-run first; see `turbofy-flows` for the declaration format).

**Chatbots must use `"operation": "streamText"`** — not `generateText`. Waiting for a full reply and then dumping it feels broken; streaming updates the assistant message as tokens arrive.

`streamText` publishes `{ type: "streamChunk", value: "<accumulated text so far>" }` repeatedly (throttled) and finally `{ type: "streamResult", value: { text, toolCalls } }` (or `{ type: "error", error }`). **Every step after the AI step runs once per publish** — that is how streaming reaches the client: create an empty assistant draft *before* the AI step, then `updateType` it after; each update is a record MODIFY that the WebSocket pushes to the block.

Step chain: `getHistory` → `createDraft` → `generateReply` (`streamText`) → `updateDraft`:

```json
{
  "triggers": {
    "onUserMessage": {
      "variant": "TABLE_DATA_CHANGE",
      "name": "onUserMessage",
      "tableId": "<messageTableId>",
      "operation": "INSERT",
      "condition": "state.role === 'user' && !!state.threadId"
    }
  },
  "startStep": "getHistory",
  "steps": {
    "getHistory": {
      "id": "getHistory",
      "name": "getHistory",
      "type": "listTypeByParent",
      "nextStep": "createDraft",
      "params": {
        "ofType": "<messageTableId>",
        "parentType": "<threadTableId>",
        "parentKey": "FIRST",
        "sortOrder": "DESC",
        "limit": 30,
        "parentId": "state.onUserMessage.threadId"
      },
      "paramsConfig": {
        "parentId": { "language": "js", "type": "logic" }
      }
    },
    "createDraft": {
      "id": "createDraft",
      "name": "createDraft",
      "type": "createType",
      "nextStep": "generateReply",
      "params": {
        "ofType": "<messageTableId>",
        "fields": "({ threadId: state.onUserMessage.threadId, role: 'assistant', content: '', isComplete: false })"
      },
      "paramsConfig": { "fields": { "language": "js", "type": "logic" } }
    },
    "generateReply": {
      "id": "generateReply",
      "name": "generateReply",
      "type": "genericAI",
      "nextStep": "updateDraft",
      "params": {
        "operation": "streamText",
        "apiKey": "<secret record uuid>",
        "model": { "provider": "openai", "model": "gpt-4o" },
        "system": "You are a helpful assistant.",
        "tools": {},
        "messages": "state.getHistory.items.map((m) => ({ role: m.role, content: m.content })).reverse()"
      },
      "paramsConfig": {
        "apiKey": { "type": "secret" },
        "messages": { "language": "js", "type": "logic" }
      }
    },
    "updateDraft": {
      "id": "updateDraft",
      "name": "updateDraft",
      "type": "updateType",
      "nextStep": null,
      "params": {
        "id": "state.createDraft.id",
        "ofType": "<messageTableId>",
        "fields": "({ content: state.generateReply.type === 'streamChunk' ? state.generateReply.value : (state.generateReply.type === 'streamResult' ? state.generateReply.value.text : 'Sorry — something went wrong.'), isComplete: state.generateReply.type === 'streamResult' || state.generateReply.type === 'error' })"
      },
      "paramsConfig": {
        "id": { "language": "js", "type": "logic" },
        "fields": { "language": "js", "type": "logic" }
      }
    }
  },
  "debug": false,
  "disabled": false
}
```

Key details:

- **`apiKey` is a secret reference**: the param value is the secret **record UUID** (`data_list` with `ofType: "secret"`), marked in `paramsConfig` as `{ "type": "secret" }`. Never a plaintext key. Users create secrets in the dashboard.
- History comes back newest-first (`DESC` + `limit` = the recent window); `.reverse()` restores chronological order for the LLM.
- Chunk values are the **accumulated full text**, not deltas — each update simply overwrites `content`. `isComplete` flips to `true` on the final `streamResult` (or `error`); the block uses it to stop its streaming cursor.

### Why the trigger condition is mandatory

The flow **writes to the same Message table that triggers it** — an infinite loop waiting to happen. The `"condition": "state.role === 'user'"` is what terminates the recursion: the flow's own writes set `role: 'assistant'`, which never matches. Upsert validation flags this pattern as a self-retrigger **error** without a condition and downgrades it to a **warning** with one — the validator only checks that a condition exists, so make sure yours actually excludes the flow's own writes.

### Provider toggle (multiple models)

`apiKey` must resolve from a static secret reference (`"paramsConfig": { "apiKey": { "type": "secret" } }`), so a single step cannot pick a secret at runtime. To let users switch providers (e.g. OpenAI ↔ Anthropic), add a `provider` enum field to Message (set by the UI), then declare **one `genericAI` `streamText` step per provider with mutually exclusive `skipIf` conditions** (e.g. `"skipIf": "state.onUserMessage.provider === 'anthropic'"` on the OpenAI step and the negation on the Anthropic step, chained via `nextStep`). The `updateDraft` step then reads whichever step ran — the skipped step is `undefined` in `state`.

## 5) Chat UI block

An ordinary block edited through the remote build session (`block_type_open` → `block_type_fs_write` → `block_type_check` → `block_type_push`; all rules from `turbofy-blocks` apply: copies, Tailwind, shadcn, loading/empty states). Data flow essentials:

```tsx
import { useState } from "react";
import { useCreateType, useListTypesByParent, useWsSubscription } from "@/api";
import type { IWsEvent } from "@/api";
import type { IBuildingBlockProps } from "@/lib/types";

interface IConfig {
  messageTableId: string; // put table IDs into the block type's defaultConfig
  threadTableId: string;
  copies?: { placeholder?: string; send?: string; empty?: string; typing?: string };
}

interface IMessage {
  id: string;
  role: string;
  content: string;
  isComplete?: boolean;
  createdAt: string;
}

export const BuildingBlock = ({ config }: IBuildingBlockProps<IConfig>) => {
  const copies = config?.copies;
  const [threadId, setThreadId] = useState<string | null>(null);
  const [input, setInput] = useState("");

  const { mutateAsync: createThread } = useCreateType({ ofType: config.threadTableId });
  const { mutateAsync: createMessage, isPending: isSending } = useCreateType({
    ofType: config.messageTableId,
  });

  // Reactive to client-side mutations; server-side (flow) writes arrive via the
  // WebSocket subscription below → refetch().
  const { data, refetch } = useListTypesByParent(
    config.messageTableId,
    config.threadTableId,
    threadId ?? "", // no thread yet → empty result
  );

  // Fast path: WebSocket events for the flow's server-side writes.
  useWsSubscription(
    { ofType: config.messageTableId, operations: ["INSERT", "UPDATE"] },
    (event: IWsEvent) => {
      const record = event.record as { threadId?: string };
      if (threadId && record.threadId === threadId) refetch();
    },
  );

  const messages = ((data ?? []) as unknown as IMessage[])
    .slice()
    .sort((a, b) => a.createdAt.localeCompare(b.createdAt));
  const awaitingReply =
    messages.length > 0 &&
    (messages[messages.length - 1].role === "user" ||
      messages[messages.length - 1].isComplete === false);

  const handleSend = async () => {
    const content = input.trim();
    if (!content || isSending) return;
    let currentThreadId = threadId;
    if (!currentThreadId) {
      const thread = await createThread({ fields: {} });
      currentThreadId = thread.id as string;
      setThreadId(currentThreadId);
    }
    await createMessage({
      fields: { threadId: currentThreadId, role: "user", content },
    });
    setInput("");
  };

  // render: message list (align by role), typing indicator while awaitingReply,
  // input + send button — all labels from config.copies
  // ...
};
```

Key points:

- **The client hooks are only reactive to client-side mutations.** The flow's writes happen server-side, so the block must pair `useListTypesByParent` with `useWsSubscription` + `refetch()`. For extra-smooth streaming you can read `event.record` directly instead of refetching.
- **Chatbots only work on private pages right now.** The WebSocket connection requires a logged-in user with a valid token, so WS events are not delivered on public pages — the assistant reply won't appear until a reload. Place the chat block on an authenticated page (`visibility: "user"`, see `turbofy-apps` for auth settings); do not build a chatbot on a public page.
- Lazily create the `Thread` on first send (as above). To persist conversations for logged-in users, parent the thread to the user (`useCurrentUser().user.sub` is the user record ID) and load the latest thread on mount instead.
- Show a typing indicator while waiting for the assistant draft (`last message.role === "user"`). Once the draft exists, render its growing `content` as Markdown.
- **Render assistant replies as Markdown** — LLMs return Markdown (`**bold**`, lists, code fences). Use `react-markdown` + `remark-gfm` (available via `@turbofy-ai/app-runtime`). Keep user messages as plain text.
- **Client typewriter on top of `streamText`** — WS/`updateType` chunks arrive in bursts and feel clunky if rendered as-is. Keep a `visibleLength` that typewrites toward the latest `content` target (not a full replay of finished history — mount already-complete messages at full length). Keep a cursor while `isComplete === false` or while still catching up after the last chunk.
- Table IDs, never table names — inject them via the block type's `defaultConfig` (e.g. `({ messageTableId: "<messageTableId>", threadTableId: "<threadTableId>" })`).

## 6) Gotchas

- **No chatbot API** — if you catch yourself looking for chat endpoints or docs, re-read section 1.
- **Trigger condition** — omitting `role === 'user'` (or writing assistant messages that still match it) creates an infinite flow loop. Validation warns, not blocks, when any condition exists.
- **`apiKey` must be a secret marker** — value = secret record UUID, `paramsConfig` entry = `{ "type": "secret" }`. Never plaintext.
- **History ordering** — `listTypeByParent` with `"sortOrder": "DESC"` + `limit` gives the most recent window; `.reverse()` it before handing to the LLM.
- **Always use `streamText` for chatbots** — `generateText` waits for the full reply and feels broken in a chat UI.
- **Stream chunks are accumulated text**, not deltas.
- **WS events are access-controlled** — end users only receive events for records their read-access policy allows, so scope Message/Thread auth accordingly if threads must be private per user.
- **Public pages don't receive WS events** — the WebSocket requires a valid end-user token, so chatbots currently only work on private (authenticated) pages. A chat block on a public page will store and answer messages, but the reply only shows up after a reload.

## See also

- `turbofy-platform` — schema JSON workflow (`workspace_get` / `workspace_schema_push`), table IDs.
- `turbofy-flows` — full declaration shape, `paramsConfig` markers, secrets, validation/recursion rules.
- `turbofy-blocks` — block build session, `@/api` hooks, `useWsSubscription`, UI/accessibility standards.
- `turbofy-apps` — placing the chat block on a page, auth settings for login-gated chat.
