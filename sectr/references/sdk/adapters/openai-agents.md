# OpenAI Agents SDK adapter

For a walkthrough built on a runnable sample, see
[using the OpenAI Agents SDK](../../getting-started/openai-agents.md).

```python
from sectr import AgentApp
from sectr.adapters.openai_agents import openai_agents

app = AgentApp(name="support-agent")
app.add_route("/chat", openai_agents(agent))
```

- Requires `openai-agents` 0.22.0 through 0.23.1 (newer releases are added as
  they're tested). To install, see the [support matrix](../../../SKILL.md#supported-agent-frameworks).
- Signature: `openai_agents(agent, *, model=None, run_config=None, context=None)`
  - `context` is a **factory** called once per invocation with the turn's
    `SessionContext` (its `ctx.session_id` identifies the session), sync or
    `async`. Its value is the SDK's run context
    (`Runner.run_streamed(context=...)`): tools, hooks, guardrails and
    dynamic instructions read it as `RunContextWrapper.context`, and MCP tool
    filters see it too. A static value is `context=lambda ctx: value`.

## Behavior

- An approved tool runs as the SDK runs any tool, with its guardrails, your
  hooks, tracing, the run context and your `run_config`.
- Handoffs (`handoffs=[...]`): the conversation stays with the agent it was
  handed off to, as in the SDK. The handoff itself is journaled as a tool
  call and its result (the SDK's `transfer_to_<agent>` call and its output),
  and every later turn, an approval resume included, continues with the agent
  the conversation is with.
- MCP servers (`mcp_servers`) are connected for each invocation, on the
  agent and on its subagents.
- Reasoning is sent back to the model on later turns and approval resumes,
  as the SDK sends it from its own history: each response's reasoning items
  (encrypted content included) are journaled with its message.

## Limits to know about

- **Structured output isn't supported yet.** An `output_type` on the agent,
  or on an agent it hands off to, fails `openai_agents(agent)` with a
  `ValueError`. The agent's answer is journaled as text.
- **Some handoffs aren't supported yet**, and `openai_agents(agent)` rejects
  them with a `ValueError`: a handoff that rewrites the history the next
  agent sees (`input_filter` or `nest_handoff_history` on `handoff()`, or
  `handoff_input_filter` or `nest_handoff_history` in your `run_config`), and
  a `Handoff` you construct yourself instead of with `handoff()`.
- **Removing a handoff strands its sessions.** If you redeploy without a
  handoff a session went through, that session's next turn ends with an
  `ERROR` whose code is `openai_agents_unknown_handoff`.

### Subagents

A subagent is another agent your agent runs from a tool. On Sectr it's an
ordinary tool, and an opaque one: the journal, and every client streaming the
session, see the tool call and its result, but not the subagent's own messages
and tool calls, so later turns don't see those either.

To require approval for what a subagent does, gate its tool: approving it runs
the whole subagent, and denying it runs none of it. A gated tool *inside* a
subagent isn't supported yet, because Sectr can't pause a turn partway through
one (the cases are below).

An `escalate` tool that runs a `specialist` agent, gated as a whole:

```python
specialist = Agent(name="specialist", instructions="...", tools=[track_parcel])
agent = Agent(
    name="support",
    tools=[
        specialist.as_tool(
            tool_name="escalate",
            tool_description="Escalate a question to the shipping specialist.",
            needs_approval=True,
        ),
    ],
)
```

An agent you hand off to (`handoffs=[...]`) isn't opaque, unlike an
`as_tool()` subagent: its messages and tool calls are journaled like your
agent's. But neither may have a gated tool, at any depth. One gated in code
makes `openai_agents(agent)` raise a `ValueError` naming the tool and
`subagent_gated_tool`, so `sectr dev` and `sectr deploy` report it. One that
the deployment's `approval_tools` names ends each turn with an `ERROR` with
that code, before the model runs.

Where your own code starts the subagent or handles what it returns, the
adapter can't see its run:

- **A subagent your tool runs with `Runner.run` is
  invisible to the adapter.** A tool the policy names on it runs **without
  approval**, and a tool gated in code stops the inner run and hands its
  interruptions back to your tool, which decides what happens next; the turn
  doesn't fail. Use `as_tool()` instead.
- **Inside a subagent, a `Handoff` you build yourself
  hides its agent.** One made with `handoff(agent)` is seen; one constructed
  directly, with your own `on_invoke_handoff`, on an `as_tool()` agent (or an
  agent it hands off to) isn't, so a tool the policy names on its agent runs
  **without approval**. Use `handoff()`. (Your agent's own handoffs must use
  it: a hand-built one is rejected.)
