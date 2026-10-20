# No framework

Without a framework adapter, your agent is one async generator per agent. It
receives a [`SessionContext`](context.md) (the conversation so
far, and why this turn started), calls your model, and yields events. Sectr
journals them, streams them to clients, and rebuilds the conversation from
them on the next turn. Tools come from [`sectr.tools`](../tools.md).

## The handler

```python
from collections.abc import AsyncIterator

from sectr import AgentApp, SessionContext
from sectr.events import RunnerEvent

app = AgentApp(name="support-agent", env=["OPENROUTER_API_KEY"])


@app.agent_route("/chat")
async def chat(ctx: SessionContext) -> AsyncIterator[RunnerEvent]:
    ...
```

- **Every turn calls the handler afresh**, with the whole conversation in
  `ctx.messages`: OpenAI-style message dicts, ready to send to a model.
  **The current user message is already in it**, so don't append it again.
- **The turn ends when the generator is exhausted.** Sectr ends the turn
  itself; there's no turn-end event to yield.
- **Create model clients lazily**, inside the handler or on first use: Sectr
  imports your module before any secret is set (see
  [your first agent](../../../SKILL.md#serving-the-agent)).

## A turn

The loop: send the conversation to the model, stream its message, run the
tools it asks for, and go again until it stops calling tools.

```python
import json
import uuid

from sectr.events import MessageDelta, MessageEnd, MessageStart, ToolCall, ToolResult
from tools import TOOLS  # sectr.tools functions


@app.agent_route("/chat")
async def chat(ctx: SessionContext) -> AsyncIterator[RunnerEvent]:
    messages = [{"role": "system", "content": SYSTEM}, *ctx.messages]
    while True:
        stream = await client().chat.completions.create(
            model=MODEL,
            messages=messages,
            tools=[t.openai_schema for t in TOOLS],
            stream=True,
        )
        msg_id = uuid.uuid4().hex
        yield MessageStart(role="assistant", message_id=msg_id)
        text, calls = "", {}
        async for chunk in stream:
            delta = chunk.choices[0].delta
            if delta.content:
                text += delta.content
                yield MessageDelta(message_id=msg_id, delta={"type": "text", "text": delta.content})
            ...  # accumulate delta.tool_calls into `calls`
        yield MessageEnd(message_id=msg_id)

        if not calls:
            return  # the turn is over

        messages.append(assistant_message(text, calls))
        for call in calls.values():
            arguments = json.loads(call["arguments"] or "{}")
            yield ToolCall(tool_call_id=call["id"], name=call["name"], arguments=arguments)
            # A gated tool raises ApprovalRequired here: the turn suspends.
            result = await ctx.execute(call["name"], call_id=call["id"], **arguments)
            yield ToolResult(tool_call_id=call["id"], result=result)
            messages.append(tool_message(call["id"], result))
```

## The events you yield

| Event                 | Yield it when               | Key fields                                                                          |
| --------------------- | --------------------------- | ----------------------------------------------------------------------------------- |
| `MessageStart`        | an assistant message begins | `message_id`, `role`                                                                |
| `MessageDelta`        | streaming its content       | `message_id`, `delta` (`{"type": "text", "text": …}` or `{"type": "reasoning", …}`) |
| `MessageEnd`          | the message is complete     | `message_id`                                                                        |
| `ToolCall`            | the model calls a tool      | `tool_call_id`, `name`, `arguments`                                                 |
| `ToolResult`          | the tool returned           | `tool_call_id`, `result`                                                            |
| `Error`               | the turn failed             | `code`, `message`                                                                   |
| `TranscriptCompacted` | you compacted the context   | `messages` (see [compaction](#compaction))                                          |

All of them are in `sectr.events`. Sectr gives each one its id and position
in the session; you never assign either.

```python
yield Error(code="model_unavailable", message="upstream 503 after 3 retries")
```

`ToolApprovalRequired` exists too, but you don't yield it: when a gated tool
raises `ApprovalRequired` out of your handler, Sectr ends the turn with it.

## Tools and approvals

Define tools with [`sectr.tools`](../tools.md), and run them with
`ctx.execute(name, call_id=..., **arguments)`. On a gated tool, `ctx.execute`
raises `ApprovalRequired`: let it propagate, and the turn suspends until
someone approves or denies the call (see
[approvals](../../../SKILL.md#no-framework)).

To ask for approval for something no single tool expresses, call
`await ctx.request_approval(name, arguments, reason=...)`: it raises
`ApprovalRequired` the same way, and the `reason` is shown with the pending
approval.

## Resuming after an approval

The decision starts a new turn, whose reason is the decision. Your handler
runs again: check `ctx.reason`, then run the pending tool (or record the
denial) before continuing the loop:

```python
if ctx.reason["kind"] == "approval_decision":
    pending = ctx.pending_tool_call
    result = (
        await ctx.execute(pending["name"], call_id=pending["tool_call_id"], **pending["arguments"])
        if ctx.decision["approved"]
        else {"error": "denied"}
    )
    yield ToolResult(tool_call_id=pending["tool_call_id"], result=result)
    messages.append(tool_message(pending["tool_call_id"], result))
```

`ctx.pending_tool_call` is the call that's waiting, and `ctx.decision` is
`{"approved": bool, "approver"?: str}`. The resumed turn runs like any other.
One decision per turn: if the model called several gated tools, each is
decided on its own turn.

## Compaction

Every turn gets the full conversation. When it outgrows the model's context,
compact it yourself: yield a `TranscriptCompacted` carrying the **full**
compacted conversation, as canonical messages (the same shape as
`ctx.messages`):

```python
yield TranscriptCompacted(messages=compact(ctx.messages))
```

From then on, `ctx.messages` starts from that snapshot: it stands in for
everything before it, and later turns build on it.

- **Compact at a turn boundary when you can** (the end of a turn, before the
  next user message), so the snapshot covers a complete exchange.
- **Carry the whole context, not a diff**: later turns, resumes and replays
  start from the snapshot alone.

Framework adapters don't journal compaction: a framework's own memory
management or summarization runs on the rebuilt conversation each turn, and
isn't recorded as a compaction.
