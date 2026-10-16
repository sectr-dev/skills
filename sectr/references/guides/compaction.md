# Compaction

Every turn re-invokes your handler with the **full transcript** — that's
what makes runners stateless and sessions portable. Conversations that outgrow
the model's context need compaction, and sectr handles it without breaking
either property.

## The mechanism

Your code (or your framework) computes the compacted context — sectr journals
it as a `TranscriptCompacted` event carrying the **full post-compaction
canonical messages**. From that point the transcript is:

- everything after the snapshot, **plus** the snapshot itself standing in for
  everything before it.

The snapshot is closed context by definition: [`ctx.messages`](../sdk/context.md)
positionally resets its fold at it — user turns derive from
`INVOCATION_STARTED`s after the snapshot, assistant messages from the
post-snapshot `MESSAGE_*` events, and so on. Identity without snapshots
holds (the fold needs no state), so crash recovery and approval resumes
derive the same context deterministically from the journal.

## Who computes it

**Frameworks with built-in memory management** (langgraph) compute
compaction naturally; the adapter journals the result. **Hand-rolled
agents** can emit `TranscriptCompacted` themselves:

```python
yield TranscriptCompacted(messages=compact(ctx.messages))
```

After that point, `ctx.messages` in the **next** turn folds through the
snapshot — the conversation continues with the compacted history.

## Guidance

- **Compact at a turn boundary when you can** (end of a turn, before the
  next user message): the snapshot then covers a complete exchange and no
  in-flight streaming needs special handling.
- **The snapshot must be canonical messages** — the same OpenAI-style shape
  `ctx.messages` produces — because the fold restarts *from* it.
- Don't try to journal a diff: the event carries the full context so that
  replay, resume, and rehydration need no history beyond it.

> **Why journal it at all**: The journal is the only source of truth: if compaction happened only in
> model memory, a resumed turn would rebuild the full transcript and blow the
> context anyway. `TranscriptCompacted` makes "the agent's context now" a
> durable fact every component agrees on — dashboards show it, resumes fold
> through it.
