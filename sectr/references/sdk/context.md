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

- **User turns** come from `INVOCATION_STARTED` envelopes with a
  `user_message` reason — and the transcript already includes *this*
  invocation's start event, so **the current user message is already in
  `ctx.messages`**. Don't append it again.
- **Assistant messages** are rebuilt from `MESSAGE_START`/`DELTA`/`END`
  (text deltas merged into `content`, reasoning deltas into
  `reasoning_content`). Keys starting with `_sectr_` are for framework
  adapters (a `MESSAGE_END`'s `framework_state` lands under
  `_sectr_framework_state`): drop them before sending messages to a
  provider yourself.
- **Tool calls** attach to the assistant message just before them as
  OpenAI `tool_calls`. A call with no message of its own (a tool-only model
  response) gets a new, empty assistant message, so it never lands on an
  earlier turn's reply. Parallel calls share one message. **Tool results**
  become `{role: "tool"}` messages.
- **An unanswered tool call** at the end (the approval pause) is preserved
  exactly as the LLM left it — the resumed turn appends the tool result
  message.
- **An interrupted tool call** — one with no result that the conversation
  moved past (the turn crashed or the tool raised, then the user wrote
  again) — gets a synthesized tool message, *"This tool call did not
  complete; no result was recorded."*, so the model never sees a call
  without an answer. Derived from the journal, never persisted. Such a call
  can no longer be approved: `ctx.pending_tool_call` stops reporting it.
- **Compaction snapshots** (`TRANSCRIPT_COMPACTED`) positionally reset the
  fold — see [compaction](../guides/compaction.md).
- **Crash stitching**: a message interrupted by a session error is continued
  with a synthesized "please continue" user message, derived from the
  journal — never persisted.

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
