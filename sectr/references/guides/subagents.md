# Subagents

A subagent is another agent your agent runs from a tool: an openai-agents
`as_tool()` agent, a LangChain agent your tool invokes, or a Pydantic AI agent
your tool delegates to. On Sectr, a subagent is just a tool.

**What works.**

- **Subagents are opaque.** The journal, and every client streaming the
  session, see the outer tool call and its result. The subagent's own
  messages, tool calls and tool results aren't journaled, so later turns don't
  see them either.
- **The subagent tool can be gated** like any other tool, in code or by
  `approval_tools`. Approving it runs the whole subagent; denying it
  runs none of it.
- Subagents without gated tools need nothing special.

**What isn't supported: a gated tool inside a subagent.** Sectr can't pause a
turn in the middle of a subagent and resume it there yet. So a tool inside a
subagent that's gated in code, or named in `approval_tools`, never runs.
Instead the agent is rejected when the runner starts, or the turn ends with an
`ERROR` whose code is `subagent_gated_tool`. To require approval for something
a subagent does, gate the subagent tool itself.

Each adapter's page shows a gated subagent tool in its framework, and that
framework's own limits:
[OpenAI Agents SDK](../sdk/adapters/openai-agents.md#subagents),
[LangGraph](../sdk/adapters/langgraph.md#subagents),
[Pydantic AI](../sdk/adapters/pydantic-ai.md#subagents).
