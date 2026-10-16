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

In each framework, an `escalate` tool that runs a `specialist` agent, gated as
a whole:

```python
# OpenAI Agents SDK
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

```python
# LangGraph
specialist = create_agent(model, tools=[as_langgraph_tool(track_parcel)], name="specialist")

@tool(needs_approval=True)  # sectr.tools.tool
async def escalate(question: str) -> str:
    """Escalate a question to the shipping specialist."""
    result = await specialist.ainvoke({"messages": [{"role": "user", "content": question}]})
    return result["messages"][-1].text

graph = create_agent(model, tools=[as_langgraph_tool(escalate)])
```

```python
# Pydantic AI
specialist = Agent(model, instructions="...", tools=[track_parcel])

async def escalate(ctx: RunContext, question: str) -> str:
    """Escalate a question to the shipping specialist."""
    result = await specialist.run(question, usage=ctx.usage)
    return result.output

agent = Agent(model, tools=[Tool(escalate, requires_approval=True)])
```

An openai-agents agent you hand off to (`handoffs=[...]`) follows the same
rule: a gated tool on it isn't supported. So does a LangGraph subgraph used as
a node: sectr tools in it are rejected when the runner starts, and a gated call
to any other tool in it (by the policy, or by the subgraph's own
`HumanInTheLoopMiddleware`) ends the turn with `subagent_gated_tool`.

**Limits to know about.** In these shapes, the adapter can't see the subagent's
run, because your own code starts it or handles what it returns:

- **Pydantic AI: `approval_tools` doesn't reach a delegate's tools.**
  A tool the policy names on an agent your tool runs (`specialist.run(...)`)
  runs **without approval**. Gate the delegating tool instead.
- **Pydantic AI: a delegate that allows `DeferredToolRequests` doesn't fail
  the turn.** A delegate with `output_type=[str, DeferredToolRequests]` hands
  a gated tool's requests back to your tool instead of raising. The gated tool
  doesn't run, but the turn carries on with whatever your tool returns. Gate
  the delegating tool instead.
- **OpenAI Agents SDK: a subagent your tool runs with `Runner.run` is
  invisible to the adapter.** A tool the policy names on it runs **without
  approval**, and a tool gated in code stops the inner run and hands its
  interruptions back to your tool, which decides what happens next; the turn
  doesn't fail. Use `as_tool()` instead.
- **OpenAI Agents SDK: inside a subagent, a `Handoff` you build yourself
  hides its agent.** One made with `handoff(agent)` is seen; one constructed
  directly, with your own `on_invoke_handoff`, on an `as_tool()` agent (or an
  agent it hands off to) isn't, so a tool the policy names on its agent runs
  **without approval**. Use `handoff()`. (Your agent's own handoffs must use
  it: a hand-built one is rejected.)
- **LangGraph: keep your graph's config when you pass one to a subagent.**
  LangChain passes your graph's config on to `specialist.ainvoke(...)` by
  itself, and the adapter uses it to tell the subagent's run from your graph's.
  A config you pass merges with it, except `configurable`, which replaces it
  whole. To add your own values, extend the inherited ones:

  ```python
  from langgraph.config import get_config

  configurable = {**get_config()["configurable"], "thread_id": thread_id}
  result = await specialist.ainvoke(messages, {"configurable": configurable})
  ```

  Without them, a gated tool inside the subagent still doesn't run, but the
  turn's `ERROR` is `langgraph_unmatched_approval` instead of
  `subagent_gated_tool`. The same happens if a task started before the turn,
  such as a background worker, runs the subagent.
