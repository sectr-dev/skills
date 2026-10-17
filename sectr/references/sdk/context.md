# SessionContext

`sectr.SessionContext` is the single argument of every handler. One
`SessionContext` = one invocation: the rehydrated transcript plus the
polymorphic reason that started it.

## The reason — input and decision

The reason is polymorphic; these sugars raise `TypeError` on the wrong kind
so misuse fails loudly:

- **`ctx.input`** — the user's message payload. Only on turns whose reason
  is `user_message`.
- **`ctx.decision`** — the approval decision, `{"approved": bool, "approver"?:
  str}`. Only on turns whose reason is `approval_decision`.

> **Note**: The input is opaque JSON: the console sends `{"message": …}`, your API
> clients may send anything. It lands in the journal inside the invocation's
> `INVOCATION_STARTED` and comes back out through `ctx.input` unchanged.

## `ctx.messages` — the conversation fold

The canonical transcript view: OpenAI-style message dicts covering the full
multi-turn conversation. Framework adapters project this into their native
state; hand-written loops feed it straight to a provider.

- **The current user message is already in `ctx.messages`**: the transcript
  includes this turn's start. Don't append it again.
- **Assistant messages** carry their text in `content` and any reasoning in
  `reasoning_content`. Keys starting with `_sectr_` are for framework
  adapters: drop them before sending messages to a provider yourself.
- **Tool calls** are OpenAI `tool_calls` on the assistant message that made
  them, and **tool results** are `{role: "tool"}` messages.
- **On an approval resume,** the conversation ends with the unanswered tool
  call, as the model left it: append its result.
- **After a compaction,** the conversation starts from the snapshot (see
  [compaction](no-framework.md#compaction)).

## Approval resume helpers

- **`ctx.pending_tool_call`** — the unanswered tool call on an
  approval-resume turn: `{name, arguments, tool_call_id, ...}` (or `None`).
- **`ctx.execute(name, call_id=..., **arguments)`** — run a registered tool.
  On a gated tool during a `user_message` turn it raises `ApprovalRequired`
  (the platform journals the approval request and suspends the turn). On an
  approval-resume turn the decision is already in hand, so executing the
  pending tool proceeds.
- **`ctx.request_approval(name, arguments)`** — explicit sugar that always
  raises `ApprovalRequired`: for gating an action no single tool expresses.

## Transcript shape

`ctx.transcript` is the raw list of `sectr.events.Envelope`s (journal rows)
— `ctx.messages` is a view over it. `ctx.session_id` / `ctx.invocation_id`
identify the session and this invocation.

> **Tip**: Treat the transcript as read-only. State you want to survive across turns
> should live **in the conversation itself** (journaled tool results, e.g. a
> `todo_write` tool whose result the next turn reads back) — that's what makes
> runners stateless and sessions portable.
