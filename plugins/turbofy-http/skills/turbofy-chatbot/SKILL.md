---
name: turbofy-chatbot
description: "Use when building a chatbot, AI assistant, support chat, or other conversational feature in a Turbofy app. A chatbot is composed from Thread/Message tables, a flow triggered by Message INSERT that calls genericAI, and a React chat block; there is no separate chatbot API. Covers the typed schema and flow sources, streaming replies, WebSocket delivery, and UI contract."
---

# Turbofy Chatbot

Turbofy chat is assembled from ordinary platform primitives:

| Piece | Implementation |
|---|---|
| Storage | Workspace `Thread` and `Message` tables |
| Reply logic | A flow triggered by user Message INSERT |
| UI | A block that creates Message records and subscribes to changes |

There is no chat endpoint or chatbot SDK. Sending means creating a Message record; receiving means observing the assistant record written by the flow.

## Architecture

```text
Chat block                  Workspace data                    Flow
useCreateType ── INSERT ──> Message(role=user) ─────────────> table trigger
                                                                  │ history
                                                                  │ genericAI
useWsSubscription <─────── Message(role=assistant) <──────── create/update
```

Workspace record changes are published to authorized WebSocket subscribers automatically; this pattern does not need a `notifyWebSocket` step.

## Schema

Use `workspace_pull`, edit `workspaces/<environment>/<workspaceId>/schema.ts`, and apply with `workspace_push` after a dry run:

```ts
const RoleEnum = builder.enumType("Role", ["user", "assistant"]);

const ThreadTable = builder.table(
  "Thread",
  { title: builder.fields.string() },
  {},
);

const MessageTable = builder.table(
  "Message",
  {
    content: builder.fields.string(),
    role: builder.fields.enum(RoleEnum),
    isComplete: builder.fields.boolean(),
  },
  { firstParent: ThreadTable },
);
```

Add both tables and the enum to `builder.build(...)`. The parent creates a `threadId` relationship field. Confirm emitted ids and field names in `schema.ts` or with `table_list`; flows and hooks use table ids, not names.

For user-owned threads, make the user table the Thread's parent. The authenticated user's record id matches `user.sub` from `useCurrentUser()`.

## Streaming reply flow

Call `flow_init`, then edit the generated `flows/<flowId>/flow.ts` and push it with `flow_push` dry-run/apply. The safe streaming sequence is:

1. Load recent history.
2. Create an empty assistant draft.
3. Run `genericAI` with `operation: "streamText"`.
4. Update the same draft after every accumulated chunk and final result.

```ts
import { flowBuilder } from "@turbofy-ai/app-runtime/dsl";

export const flow = flowBuilder.buildFlow({
  name: "Chatbot reply",
  triggers: {
    onUserMessage: flowBuilder.trigger.tableDataChange({
      tableId: "<message-table-id>",
      operation: "INSERT",
      condition: "state.role === 'user' && !!state.threadId",
    }),
  },
  steps: [
    flowBuilder.step.listTypeByParent("getHistory", {
      params: {
        ofType: "<message-table-id>",
        parentType: "<thread-table-id>",
        parentKey: "FIRST",
        parentId: flowBuilder.js("state.onUserMessage.threadId"),
        sortOrder: "DESC",
        limit: 30,
      },
    }),
    flowBuilder.step.createType("createDraft", {
      params: {
        ofType: "<message-table-id>",
        fields: flowBuilder.js(
          "({ threadId: state.onUserMessage.threadId, role: 'assistant', content: '', isComplete: false })",
        ),
        dynamicArgs: flowBuilder.js(
          "({ ownerSub: state.onUserMessage.pbac0sub })",
        ),
      },
    }),
    flowBuilder.step.genericAI("generateReply", {
      params: {
        operation: "streamText",
        apiKey: flowBuilder.secret("<secret-record-id>"),
        model: { provider: "openai", model: "gpt-4o" },
        system: "You are a helpful assistant.",
        tools: {},
        messages: flowBuilder.js(
          "state.getHistory.items.map((m) => ({ role: m.role, content: m.content })).reverse()",
        ),
      },
    }),
    flowBuilder.step.updateType("updateDraft", {
      params: {
        id: flowBuilder.js("state.createDraft.id"),
        ofType: "<message-table-id>",
        fields: flowBuilder.js(
          "({ content: state.generateReply.type === 'streamChunk' ? state.generateReply.value : (state.generateReply.type === 'streamResult' ? state.generateReply.value.text : 'Sorry — something went wrong.'), isComplete: state.generateReply.type === 'streamResult' || state.generateReply.type === 'error' })",
        ),
      },
    }),
  ],
});
```

The trigger condition is mandatory: the flow writes to the table that triggers it. Its assistant records must not match the user-message condition. Push validation warns about the guarded self-retrigger; verify that the condition actually terminates it.

Stream chunks contain the accumulated reply, not deltas. The update step overwrites `content` and marks the final result complete. Store provider keys as workspace secrets and reference their record ids with `flowBuilder.secret(...)`.

The trigger payload includes `pbac0sub` from the user message. Forward it as `dynamicArgs.ownerSub` when creating the assistant draft so the reply belongs to the same end user. Without this, the elevated flow write has no end-user owner, and PBAC prevents the client from listing the reply or receiving its WebSocket events.

## Chat block

Pull the app, then edit `block-types/Chat/record.ts` and `index.tsx` through `fs_*`. Keep table ids in `defaultConfig` and UI strings in `localizations`; run `block_type_check` and publish through `app_push`.

The component needs:

- `useCreateType` to lazily create a Thread and add user Messages
- `useListTypesByParent` to load the selected Thread's Messages
- `useWsSubscription` for Message `INSERT`/`UPDATE`, followed by `refetch()` or direct event reconciliation
- loading, empty, error, sending, and streaming states from `config.copies`
- Markdown rendering for assistant output; keep user input as plain text

```tsx
const { mutateAsync: createMessage } = useCreateType({ ofType: config.messageTableId });
const { data, refetch } = useListTypesByParent(
  config.messageTableId,
  config.threadTableId,
  threadId ?? "",
);

useWsSubscription(
  { ofType: config.messageTableId, operations: ["INSERT", "UPDATE"] },
  (event) => {
    const record = event.record as { threadId?: string };
    if (threadId && record.threadId === threadId) refetch();
  },
);
```

Sort messages chronologically for display. Show a typing indicator after the user message and while the assistant draft has `isComplete === false`. For smooth rendering, animate visible text toward the latest accumulated server value rather than replaying completed messages.

WebSocket delivery requires an authenticated user. Place the chat on a protected page and configure read access so each user can observe only their own Thread/Message records. Public pages can write and receive server replies in storage, but will not receive authenticated WebSocket events.

## Checklist

- Schema declarations retain existing ids and include Thread, Message, and Role in `builder.build(...)`.
- Flow trigger accepts only user messages and references exact table ids.
- History is limited, fetched newest-first, then reversed for the model.
- AI credentials use `secret(...)`; no values are committed to source.
- Assistant creation forwards `pbac0sub` through `dynamicArgs.ownerSub`.
- `streamText` updates one assistant draft; chunks are treated as accumulated text.
- The block subscribes to server writes and handles access-controlled WebSocket delivery.
- `workspace_push`, `flow_push`, and `app_push` are dry-run before apply.

## See also

- `turbofy-platform` — schema DSL, table ids, secrets
- `turbofy-flows` — full flow DSL and validation
- `turbofy-blocks` — React hooks, copies, and UI rules
- `turbofy-apps` — app tree, page visibility, and auth
