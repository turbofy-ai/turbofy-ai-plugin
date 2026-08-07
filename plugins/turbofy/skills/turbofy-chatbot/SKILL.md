---
name: turbofy-chatbot
description: "Use when building a chatbot, AI assistant, or any conversational feature in a Turbofy workspace/app — 'add a chatbot', 'AI support chat', 'chat with my data', 'assistant widget' — or when a workspace contains Thread/Message tables or a product_chatbot flag and the task touches chat. CRITICAL: Turbofy has NO chatbot API, endpoint, or SDK. A chatbot is assembled from three ordinary primitives: Thread/Message tables (schema), a flow triggered on Message INSERT that calls genericAI and writes the assistant reply, and a chat UI block. product_chatbot is a legacy flag — ignore it. Loads the full recipe: schema, flow (incl. streaming), chat block, and the WebSocket delivery model. Companions: turbofy-platform (schema), turbofy-flows (DSL details), turbofy-blocks (UI rules)."
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

Add to `schema.ts` (workflow: `Turbofy_workspace_pull` → edit → `Turbofy_workspace_push`; see `turbofy-platform`):

```ts
const RoleEnum = builder.enumType("Role", ["user", "assistant"]);

const ThreadTable = builder.table(
  "Thread",
  {
    title: builder.fields.string(),
  },
  {
    // Optional: parent to the @users table so each signed-in user owns their threads
    firstParent: UserTable,
  },
);

const MessageTable = builder.table(
  "Message",
  {
    content: builder.fields.string(),
    role: builder.fields.enum(RoleEnum),
    isComplete: builder.fields.boolean(), // only needed for the streaming variant
  },
  {
    firstParent: ThreadTable,
  },
);
```

Notes:

- The parent link puts a `threadId` field on `Message` records (child field is the camelCase parent name + `Id`). If you're reusing existing tables, confirm the actual field name via `Turbofy_table_list`.
- For a logged-in chatbot: the signed-in user's record ID in the `@users` table equals `user.sub` from `useCurrentUser()` (`@/lib/auth`) — use it as the Thread's user parent value.
- `content` as `string()` is enough for plain text. Use `json()` if you later need structured content (tool calls, attachments).
- An optional `Agent` table (`name`, `systemPrompt` fields) lets users edit the prompt as data instead of hardcoding it in the flow.

## 4) Flow — the bot's brain

Workflow: `Turbofy_flow_init` → edit `flow.ts` → `Turbofy_flow_push` (see `turbofy-flows` for the full DSL).

**Chatbots must use `operation: "streamText"`** — not `generateText`. Waiting for a full reply and then dumping it feels broken; streaming updates the assistant message as tokens arrive.

`streamText` publishes `{ type: "streamChunk", value: "<accumulated text so far>" }` repeatedly (throttled) and finally `{ type: "streamResult", value: { text, toolCalls } }` (or `{ type: "error", error }`). **Every step after the AI step runs once per publish** — that is how streaming reaches the client: create an empty assistant draft *before* the AI step, then `updateType` it after; each update is a record MODIFY that the WebSocket pushes to the block.

Step chain: `getHistory` → `createDraft` → `generateReply` (`streamText`) → `updateDraft`:

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Chatbot reply",
  triggers: {
    onUserMessage: flowBuilder.trigger.tableDataChange({
      tableId: "<messageTableId>",
      operation: "INSERT",
      // MANDATORY — see "Why the condition is mandatory" below
      condition: "state.role === 'user' && !!state.threadId",
    }),
  },
  steps: [
    flowBuilder.step.listTypeByParent("getHistory", {
      params: {
        ofType: "<messageTableId>",
        parentType: "<threadTableId>",
        parentKey: "FIRST",
        sortOrder: "DESC",
        limit: 30,
        parentId: flowBuilder.js("state.onUserMessage.threadId"),
      },
    }),
    flowBuilder.step.createType("createDraft", {
      params: {
        ofType: "<messageTableId>",
        fields: flowBuilder.js(
          "({ threadId: state.onUserMessage.threadId, role: 'assistant', content: '', isComplete: false })",
        ),
        // REQUIRED — attribute the draft to the user who sent the trigger message
        dynamicArgs: flowBuilder.js(
          "({ ownerSub: state.onUserMessage.pbac0sub })",
        ),
      },
    }),
    flowBuilder.step.genericAI("generateReply", {
      params: {
        operation: "streamText", // REQUIRED for chatbots — do not use generateText
        apiKey: flowBuilder.secret("<secret record id>"), // never a plain string
        model: { provider: "openai", model: "gpt-4o" },
        system: "You are a helpful assistant.",
        tools: {}, // required by the params schema even when unused
        // DESC + reverse() → chronological order for the LLM
        messages: flowBuilder.js(
          "state.getHistory.items.map((m) => ({ role: m.role, content: m.content })).reverse()",
        ),
      },
    }),
    flowBuilder.step.updateType("updateDraft", {
      params: {
        id: flowBuilder.js("state.createDraft.id"),
        ofType: "<messageTableId>",
        fields: flowBuilder.js(
          "({ content: state.generateReply.type === 'streamChunk' ? state.generateReply.value : (state.generateReply.type === 'streamResult' ? state.generateReply.value.text : 'Sorry — something went wrong.'), isComplete: state.generateReply.type === 'streamResult' || state.generateReply.type === 'error' })",
        ),
      },
    }),
  ],
});
```

Chunk values are the **accumulated full text**, not deltas — each update simply overwrites `content`. `isComplete` flips to `true` on the final `streamResult` (or `error`); the block uses it to stop its streaming cursor.

**Ownership forwarding:** the TABLE_DATA_CHANGE trigger payload includes `pbac0sub` from the inserted user message. Pass it as `dynamicArgs.ownerSub` on `createDraft` so the assistant message is owned by the same end user. Without this, the flow writes as elevated IAM with no end-user owner — the client cannot list the reply or receive its WebSocket events under PBAC.

### Why the trigger condition is mandatory

The flow **writes to the same Message table that triggers it** — an infinite loop waiting to happen. The `condition: "state.role === 'user'"` is what terminates the recursion: the flow's own writes set `role: 'assistant'`, which never matches. Push validation flags this pattern as a self-retrigger **error** without a condition and downgrades it to a **warning** with one — the validator only checks that a condition exists, so make sure yours actually excludes the flow's own writes.

### Provider toggle (multiple models)

`apiKey` must be a static `flowBuilder.secret(...)` marker, so a single step cannot pick a secret at runtime. To let users switch providers (e.g. OpenAI ↔ Anthropic), add a `provider` enum field to Message (set by the UI), then declare **one `genericAI` `streamText` step per provider with mutually exclusive `skipIf` conditions**:

```ts
flowBuilder.step.genericAI("generateOpenAI", {
  skipIf: "state.onUserMessage.provider === 'anthropic'",
  params: { operation: "streamText", /* openai model + openai secret */ },
}),
flowBuilder.step.genericAI("generateAnthropic", {
  skipIf: "state.onUserMessage.provider !== 'anthropic'",
  params: { operation: "streamText", /* anthropic model + anthropic secret */ },
}),
// updateDraft then reads whichever step ran (the skipped one is undefined in state)
```

## 5) Chat UI block

An ordinary block (`block-types/Chat/index.tsx` — all rules from `turbofy-blocks` apply: copies, Tailwind, shadcn, loading/empty states). Data flow essentials:

```tsx
import { useState } from "react";
import { useCreateType, useListTypesByParent, useWsSubscription } from "@/api";
import type { IWsEvent } from "@/api";
import type { IBuildingBlockProps } from "@/lib/types";

interface IConfig {
  messageTableId: string; // put table IDs into defaultConfig in record.ts
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
- Table IDs, never table names — inject them via `defaultConfig` in `record.ts` (e.g. `({ messageTableId: "${MessageTable.id}", threadTableId: "${ThreadTable.id}" })`).

## 6) Gotchas

- **No chatbot API** — if you catch yourself looking for chat endpoints or docs, re-read section 1.
- **Trigger condition** — omitting `role === 'user'` (or writing assistant messages that still match it) creates an infinite flow loop. Validation warns, not blocks, when any condition exists.
- **`apiKey` must be `flowBuilder.secret(...)`** — the LLM key lives in a workspace secret (dashboard → Workspace → Secrets), never in `flow.ts`.
- **History ordering** — `listTypeByParent` with `sortOrder: "DESC"` + `limit` gives the most recent window; `.reverse()` it before handing to the LLM.
- **Always use `streamText` for chatbots** — `generateText` waits for the full reply and feels broken in a chat UI.
- **Stream chunks are accumulated text**, not deltas.
- **WS events are access-controlled** — end users only receive events for records their read-access policy allows, so scope Message/Thread auth accordingly if threads must be private per user.
- **Forward `pbac0sub` → `ownerSub` on assistant creates** — `createDraft` must set `dynamicArgs: flowBuilder.js("({ ownerSub: state.onUserMessage.pbac0sub })")`. Otherwise the reply is not owned by the user and will not show up via list/WS under PBAC.
- **Public pages don't receive WS events** — the WebSocket requires a valid end-user token, so chatbots currently only work on private (authenticated) pages. A chat block on a public page will store and answer messages, but the reply only shows up after a reload.

## See also

- `turbofy-platform` — schema workflow (`Turbofy_workspace_pull/push`), data-builder DSL, table IDs.
- `turbofy-flows` — full flowBuilder DSL, all step signatures, secrets, validation/recursion rules.
- `turbofy-blocks` — block component rules, `@/api` hooks, `useWsSubscription`, UI/accessibility standards.
- `turbofy-apps` — placing the chat block on a page, auth settings for login-gated chat.
