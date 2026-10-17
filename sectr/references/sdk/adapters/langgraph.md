# LangGraph adapter

For a walkthrough built on a runnable sample, see
[using LangGraph](../../getting-started/langgraph.md).

```python
from langchain_core.messages import SystemMessage
from langgraph.graph import END, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode

from sectr import AgentApp
from sectr.adapters.langgraph import as_langgraph_tool, langgraph

tools = ToolNode([as_langgraph_tool(t) for t in TOOLS])  # TOOLS: sectr.tools.tool functions
# ... build a StateGraph(MessagesState) with an agent node and `tools` ...
graph = builder.compile()  # no checkpointer

app = AgentApp(name="support-agent")
app.add_route("/chat", langgraph(graph))
```

- Requires `langgraph>=1.2.11`, `langgraph-prebuilt>=1.1.0` and
  `langchain-core>=1.6.6`. To install, see the [support matrix](../../../SKILL.md#supported-agent-frameworks).
- Signatures:
  - `langgraph(graph, *, context=None) -> Handler` — wraps a compiled graph.
    Raises `ValueError` at construction (before the first turn) for
    unsupported graph shapes.
    - `context` is a **factory** called once per invocation with the turn's
      `SessionContext` (its `ctx.session_id` identifies the session), sync or
      `async`. Its value
      is the graph's runtime context (`astream(context=...)`): tools read it as
      `runtime.context` (`ToolRuntime`), and nodes and middleware as
      `Runtime.context`. A static value is `context=lambda ctx: value`. A
      resumed approved tool gets the same context.
  - `as_langgraph_tool(tool, *, needs_approval=False) -> BaseTool` — converts
    a [`sectr.tools.tool`](../tools.md) function for a `ToolNode`.
    The model-facing schema comes from the function signature (untyped
    parameters are `str`); the tool needs a docstring. Given a LangChain tool
    instead (any `BaseTool`: a `@tool`, an MCP tool, a community tool), it
    returns it gated when `needs_approval=True`, otherwise unchanged.
    `needs_approval=True` also gates a sectr tool.

## Behavior

Each turn streams the graph with `stream_mode=["messages",
"updates"]`: text and reasoning chunks become `MESSAGE_*` events, tool calls
`TOOL_CALL`, and tool-node messages `TOOL_RESULT`. A model message's
reasoning, as the graph stores it (signatures included), and its `name` (a
named agent's) are journaled with the message and sent back on later turns
and approval resumes. A gated tool
(`needs_approval=True`, `HumanInTheLoopMiddleware`'s `interrupt_on`, or
the deployment's
[`approval_tools`](../../../SKILL.md#marking-as-gated-in-deployment-config))
suspends the turn with `TOOL_APPROVAL_REQUIRED`. Other tool calls in the same model message run
first, and their results are recorded before the pause, so each runs once.
`approval_tools` also gates the `ToolNode`'s
other tools, such as plain LangChain tools and MCP tools from
`langchain-mcp-adapters`. On approval the graph resumes at its tool node and
runs the call exactly once, as on any turn: your `ToolNode`'s settings, tool
middleware, injected arguments, `Command` results and the edges out of the
tool node all apply. A denied call gets the standard rejection text without
running. Several gated calls in one model message become sequential
approvals, one per turn. See [approvals](../../../SKILL.md#defining-gated-tools).

## Supported graphs

ReAct-shaped: a single top-level `ToolNode` holding your
sectr tools: `langchain.agents.create_agent`, a `StateGraph(MessagesState)`
with an agent node, or the deprecated `create_react_agent`. These fail with a
clear error instead:

- a graph compiled with a checkpointer (the journal is the only state store);
- sectr tools in several `ToolNode`s (multi-agent and supervisor graphs), or
  in a `ToolNode` nested in a subgraph (the error names `subagent_gated_tool`:
  see [subagents](#subagents));
- a sectr tool that takes graph-injected arguments (`InjectedState`,
  `InjectedStore`, `ToolRuntime`);
- a gated sectr tool called from your own tool node instead of a `ToolNode`,
  a tool named in `approval_tools` in a second top-level `ToolNode`,
  or a gated call in a graph whose tool node is reached from more than one
  node (these fail at the first gated call, before anyone is asked to
  approve);
- structured output, which isn't supported yet: a `response_format` on
  `create_agent` or `create_react_agent`. (A node of your own that calls
  `with_structured_output` can't be detected; its result isn't journaled as
  the answer.)

## How the adapter works

LangGraph's own human-in-the-loop (`interrupt()`
with `Command(resume=...)`) needs a checkpointer, which would be a second
source of truth next to the journal, so the adapter doesn't use one:

- **Every turn is a fresh `graph.astream`**, its message state rebuilt from the
  journaled transcript: the journal is the checkpoint.
- **A pause is an exception.** A gated sectr tool raises `ApprovalRequired`, a
  `BaseException`, so LangGraph's tool error handling can't swallow it; the
  adapter's wrapper around the tool node raises it for any other gated call
  (`interrupt_on`, `as_langgraph_tool(..., needs_approval=True)`,
  `approval_tools`). The adapter journals the model's tool call and ends the
  turn with `TOOL_APPROVAL_REQUIRED`.
- **A resume re-enters the graph at its tool node.** The adapter rebuilds the
  paused state from the journal (with a throwaway in-memory checkpointer, for
  that one turn only), and the graph's own tool node runs the approved call
  or answers it with the rejection text; the model generates only the
  continuation.

## Limits to know about

- **Gate tools, don't call `interrupt()`.** LangGraph's native
  `langgraph.types.interrupt()` can't be resumed without a checkpointer, so it
  fails loudly (a runtime error; `sectr inspect` also warns when it finds the
  import). Gate tools with `needs_approval=True`, or with LangChain's
  `HumanInTheLoopMiddleware` on `create_agent`: Sectr honors its
  `interrupt_on` with its own pause, so the middleware never calls
  `interrupt()`. Each entry must allow both approve and reject (`True`
  does); `edit` and `respond` decisions aren't offered, and an entry's
  `when` predicate applies. Middleware on a subagent isn't honored: a tool
  it gates ends the turn with `subagent_gated_tool`, without running. Native
  `interrupt()`, multi-agent graphs, and checkpointed resume are planned.
- **Conversation state only.** Non-message state channels don't survive from
  one turn to the next. Keep state in the conversation — for example a
  `todo_write` tool whose journaled call *is* the write.

### Subagents

A subagent is another agent your agent runs from a tool. On Sectr it's an
ordinary tool, and an opaque one: the journal, and every client streaming the
session, see the tool call and its result, but not the subagent's own messages
and tool calls, so later turns don't see those either.

To require approval for what a subagent does, gate its tool: approving it runs
the whole subagent, and denying it runs none of it. A gated tool *inside* a
subagent isn't supported yet, because Sectr can't pause a turn partway through
one: such a tool never runs. Instead the agent is rejected when the runner
starts, or the turn ends with an `ERROR` whose code is `subagent_gated_tool`.

An `escalate` tool that runs a `specialist` agent, gated as a whole:

```python
specialist = create_agent(model, tools=[as_langgraph_tool(track_parcel)], name="specialist")

@tool(needs_approval=True)  # sectr.tools.tool
async def escalate(question: str) -> str:
    """Escalate a question to the shipping specialist."""
    result = await specialist.ainvoke({"messages": [{"role": "user", "content": question}]})
    return result["messages"][-1].text

graph = create_agent(model, tools=[as_langgraph_tool(escalate)])
```

A LangGraph subgraph used as a node is treated the same way: sectr tools in it
are rejected when the runner starts, and a gated call to any other tool in it
(by the policy, or by the subgraph's own `HumanInTheLoopMiddleware`) ends the
turn with `subagent_gated_tool`.

Where your own code starts the subagent or handles what it returns, the
adapter can't see its run:

- **Keep your graph's config when you pass one to a subagent.**
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
