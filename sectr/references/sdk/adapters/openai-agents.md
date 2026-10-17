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
  - `context` is a **factory** called once per invocation with the
    [`SessionContext`](../context.md), sync or `async`. Its value
    is the SDK's run context (`Runner.run_streamed(context=...)`): tools,
    hooks, guardrails and dynamic instructions read it as
    `RunContextWrapper.context`, and MCP tool filters see it too. A static
    value is `context=lambda ctx: value`.
- An approved tool runs as the SDK runs any tool, with its guardrails, your
  hooks, tracing, the run context and your `run_config`.
- Structured output isn't supported yet: an `output_type` on the agent, or on
  an agent it hands off to, fails `openai_agents(agent)` with a `ValueError`.
  The agent's answer is journaled as text.
- Handoffs (`handoffs=[...]`): the conversation stays with the agent it was
  handed off to, as in the SDK. The handoff itself is journaled as a tool
  call and its result (the SDK's `transfer_to_<agent>` call and its output),
  and every later turn, an approval resume included, continues with the agent
  the conversation is with. If you redeploy without a handoff a session went
  through, that session's next turn ends with an `ERROR` whose code is
  `openai_agents_unknown_handoff`.
  - Not supported yet, and rejected by `openai_agents(agent)` with a
    `ValueError`: a handoff that rewrites the history the next agent sees
    (`input_filter` or `nest_handoff_history` on `handoff()`, or
    `handoff_input_filter` or `nest_handoff_history` in your `run_config`),
    and a `Handoff` you construct yourself instead of with `handoff()`.
- Subagents (`as_tool()`) are opaque tools. An agent it hands off to isn't
  opaque: its messages and tool calls are journaled like your agent's.
  Neither may have a gated tool, at any depth. One gated in code makes
  `openai_agents(agent)` raise a `ValueError` naming the tool and
  `subagent_gated_tool`, so `sectr dev` and `sectr deploy` report it. One
  that the deployment's
  [`approval_tools`](../../../SKILL.md#marking-as-gated-in-deployment-config)
  names ends each turn with an `ERROR` with that code, before the model
  runs. See [subagents](../../guides/subagents.md).
- MCP servers (`mcp_servers`) are connected for each invocation, on the
  agent and on its subagents.
- Reasoning is sent back to the model on later turns and approval resumes,
  as the SDK sends it from its own history: each response's reasoning items
  (encrypted content included) are journaled with its message.
