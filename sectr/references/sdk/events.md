# Events

`sectr.events` mirrors the platform's event schema. Handlers yield
**`RunnerEvent`** subclasses; the platform wraps each into an **`Envelope`**
(a journal row: `id`, `session_id`, `seq`, `created_at`, …) when it journals
the stream — user code never assigns ids or sequence numbers.

## The vocabulary

| Event                  | Emitted when                | Key fields                                                                          |
| ---------------------- | --------------------------- | ----------------------------------------------------------------------------------- |
| `MessageStart`         | an assistant message begins | `message_id`, `role`                                                                |
| `MessageDelta`         | streaming content           | `message_id`, `delta` (`{"type": "text", "text": …}` or `{"type": "reasoning", …}`) |
| `MessageEnd`           | the message is complete     | `message_id`, `framework_state?`                                                    |
| `ToolCall`             | the agent requests a tool   | `tool_call_id`, `name`, `arguments`                                                 |
| `ToolResult`           | the tool returned           | `tool_call_id`, `result`                                                            |
| `ToolApprovalRequired` | a gated call awaits a human | `tool_call_id`, `name`, `arguments`, `reason?` — **must be the final event**        |
| `Error`                | the turn failed             | `code`, `message`                                                                   |
| `TranscriptCompacted`  | context was compacted       | `messages` — see [compaction](../guides/compaction.md)                              |

`framework_state` is for framework adapters: what an adapter needs to send
that one model response back as its framework stored it (signed reasoning,
for example), as `{"adapter": …, "v": …, "data": {…}}`. Only the adapter
that wrote it reads it, and the platform drops one over 256 KiB. Hand-written handlers
leave it out.

There is **no** "turn end" event: exhausting the generator ends the turn —
the platform appends `NO_MORE_ACTIONS` itself. Emitting it from user code is
an error the adapter catches.

Platform-only types (`SESSION_CREATED`, `INVOCATION_STARTED`,
`TOOL_APPROVAL_GRANTED`/`DENIED`, `SESSION_ERROR`) have no constructors in
the SDK — runners never emit them; they appear inside rehydrated transcripts
as `Envelope`s, which is why [`ctx.messages`](context.md)
can fold them into the conversation.

## Shapes

**Streaming a message:**

```python
yield MessageStart(role="assistant", message_id=msg_id)
yield MessageDelta(message_id=msg_id, delta={"type": "text", "text": chunk})
yield MessageEnd(message_id=msg_id)
```

```python
yield ToolCall(tool_call_id=frag["id"], name=frag["name"], arguments=arguments)
result = await ctx.execute(frag["name"], call_id=frag["id"], **arguments)
yield ToolResult(tool_call_id=frag["id"], result=result)
```

```python
yield Error(code="model_unavailable", message="upstream 503 after 3 retries")
```

`ToolApprovalRequired` is normally **not** emitted by hand: the adapter
catches `ApprovalRequired` escaping your handler and emits it as the final
frame (finality is adapter-enforced). Emit it yourself only when replicating
that machinery deliberately.

All snippets are from the framework-free example (`examples/raw/` in the
repo) — the examples double as the SDK's conformance fixtures, so the docs
can't drift from what the platform actually accepts.
