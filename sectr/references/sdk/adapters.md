# Framework adapters

An adapter wraps the agent you define using a 3rd party agent framework, so that it
runs out of the box on Sectr. Your agent code stays 100% framework code - zero platform
concepts leak in.

## OpenAI Agents SDK

```python
from sectr import AgentApp
from sectr.adapters.openai_agents import openai_agents

app = AgentApp(name="support-agent")
app.add_route("/chat", openai_agents(agent))
```

- Installation: `uv add sectr[openai-agents]` (supports `openai-agents` 0.22.0
  through 0.23.1; newer releases are added as they're tested)
- Signature: `openai_agents(agent, *, model=None, run_config=None, context=None)`
  - `context` is a **factory** called once per invocation with the
    [`SessionContext`](context.md), sync or `async`. Its value
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
  [`approval_tools`](../../SKILL.md#marking-as-gated-in-deployment-config)
  names ends each turn with an `ERROR` with that code, before the model
  runs. See [subagents](#subagents).
- MCP servers (`mcp_servers`) are connected for each invocation, on the
  agent and on its subagents.
- Reasoning is sent back to the model on later turns and approval resumes,
  as the SDK sends it from its own history: each response's reasoning items
  (encrypted content included) are journaled with its message.

See the [first-agent guide](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) for the full walkthroug).

## LangGraph

For a walkthrough built on a runnable sample, see
[using LangGraph](../getting-started/langgraph.md).

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

- Installation: `uv add sectr[langgraph]` (requires `langgraph>=1.2.11`, `langgraph-prebuilt>=1.1.0`, `langchain-core>=1.6.6`)
- Signatures:
  - `langgraph(graph, *, context=None) -> Handler` — wraps a compiled graph.
    Raises `ValueError` at construction (before the first turn) for
    unsupported graph shapes.
    - `context` is a **factory** called once per invocation with the
      [`SessionContext`](context.md), sync or `async`. Its value
      is the graph's runtime context (`astream(context=...)`): tools read it as
      `runtime.context` (`ToolRuntime`), and nodes and middleware as
      `Runtime.context`. A static value is `context=lambda ctx: value`. A
      resumed approved tool gets the same context.
  - `as_langgraph_tool(tool, *, needs_approval=False) -> BaseTool` — converts
    a [`sectr.tools.tool`](tools.md) function for a `ToolNode`.
    The model-facing schema comes from the function signature (untyped
    parameters are `str`); the tool needs a docstring. Given a LangChain tool
    instead (any `BaseTool`: a `@tool`, an MCP tool, a community tool), it
    returns it gated when `needs_approval=True`, otherwise unchanged.
    `needs_approval=True` also gates a sectr tool.

**Behavior.** Each turn streams the graph with `stream_mode=["messages",
"updates"]`: text and reasoning chunks become `MESSAGE_*` events, tool calls
`TOOL_CALL`, and tool-node messages `TOOL_RESULT`. A model message's
reasoning, as the graph stores it (signatures included), and its `name` (a
named agent's) are journaled with the message and sent back on later turns
and approval resumes. A gated tool
(`needs_approval=True`, `HumanInTheLoopMiddleware`'s `interrupt_on`, or
the deployment's
[`approval_tools`](../../SKILL.md#marking-as-gated-in-deployment-config))
suspends the turn with `TOOL_APPROVAL_REQUIRED`. Other tool calls in the same model message run
first, and their results are recorded before the pause, so each runs once.
`approval_tools` also gates the `ToolNode`'s
other tools, such as plain LangChain tools and MCP tools from
`langchain-mcp-adapters`. On approval the graph resumes at its tool node and
runs the call exactly once, as on any turn: your `ToolNode`'s settings, tool
middleware, injected arguments, `Command` results and the edges out of the
tool node all apply. A denied call gets the standard rejection text without
running. Several gated calls in one model message become sequential
approvals, one per turn. See [approvals](../../SKILL.md#defining-gated-tools).

**Supported graphs.** ReAct-shaped: a single top-level `ToolNode` holding your
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

**Limits to know about.**

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
- **Subagents** are opaque tools, and a gated tool inside one isn't
  supported. See [subagents](#subagents).

## Pydantic AI

For a walkthrough built on a runnable sample, see
[using Pydantic AI](../getting-started/pydantic-ai.md).

```python
from pydantic_ai import Agent, Tool

from sectr import AgentApp
from sectr.adapters.pydantic_ai import pydantic_ai

agent = Agent(model, tools=[lookup_order, Tool(refund, requires_approval=True)])

app = AgentApp(name="support-agent")
app.add_route("/chat", pydantic_ai(agent))
```

- Installation: `uv add "sectr[pydantic-ai]" "pydantic-ai-slim[openai]"` (requires
  `pydantic-ai-slim>=2.54`; the extra pulls in no model provider, so add the one
  you use).
- Signature: `pydantic_ai(agent, *, model=None, deps=None, model_settings=None, usage_limits=None) -> Handler`
  - `model`, `model_settings` and `usage_limits` pass through to `agent.iter`;
    `model` is for deterministic fakes.
  - `deps` is a **factory** called once per invocation with the
    [`SessionContext`](context.md), sync or `async`. A static value
    is `deps=lambda ctx: value`. A resumed approved tool gets the same `deps`.
  - Raises `TypeError` at construction for anything but a plain
    `pydantic_ai.Agent`: durable-execution wrappers (`TemporalAgent`,
    `DBOSAgent`, `PrefectAgent`) own a second durable store, which conflicts
    with the journal being the checkpoint.

**Behavior.** Each turn runs `agent.iter(...)` on a `message_history` rebuilt
from the journal, and streams the run: text and reasoning become `MESSAGE_*`
events (one start/end bracket per model response), tool calls `TOOL_CALL`
(journaled once, after the message they belong to), and tool results
`TOOL_RESULT`. A response's thinking (signatures included) is journaled with
its message and sent back on later turns and approval resumes, to the
provider that issued it, as Pydantic AI does natively. A gated tool (`requires_approval=True`, a tool that raises
`ApprovalRequired`, or the deployment's
[`approval_tools`](../../SKILL.md#marking-as-gated-in-deployment-config))
suspends the turn with `TOOL_APPROVAL_REQUIRED`. On approval, Pydantic AI resumes from the history
with `DeferredToolResults` and runs the tool itself, once; when denied it
returns the standard rejection text and the tool never runs. System prompts
(`system_prompt=`, static or dynamic) are restored on every turn;
`instructions=` is applied by Pydantic AI as usual. Several gated calls in one
response become sequential approvals, one per turn. See
[approvals](../../SKILL.md#defining-gated-tools).

**Output types.** Structured output isn't supported yet: an `output_type`
other than `str` (a model, `ToolOutput`, `NativeOutput`, ...) fails
`pydantic_ai(agent)` with a `ValueError`, and the agent's answer is journaled
as text. Pausing needs `DeferredToolRequests` among the run's output types.
Sectr adds it, unless your agent already lists it. If your agent has
`@agent.output_validator`s, Pydantic AI refuses a per-run output type, so
declare `output_type=[str, DeferredToolRequests]` (or `str | DeferredToolRequests`) on the agent yourself;
otherwise the adapter raises a `ValueError` (at construction, and again at the
start of each turn in case a validator was added later).

**Limits to know about.**

- **External tool results aren't supported yet.** A tool that raises
  `CallDeferred` (or has `kind='external'`) ends the turn with an
  `ERROR` (`pydantic_ai_external_call`), because Sectr has no external
  tool-result path yet.
- **Auto-approvers bypass your own gates.** A `HandleDeferredToolCalls`
  capability on your agent approves natively gated tools without asking. Tools
  named in `approval_tools` (including MCP and other toolset tools)
  still pause.
- **The policy covers your agent's own tools only.** `approval_tools`
  doesn't reach the tools of an agent you delegate to: see
  [subagents](#subagents).
- **Provider-native tools** (web search, code execution) aren't journaled; the
  text answer is, so later turns don't see the tool call itself.
- **Text only.** Non-text user input is stringified, and binary tool returns
  or follow-up user content from `ToolReturn(content=...)` aren't journaled
  faithfully.
- **Compaction.** `history_processors` still run on the rebuilt history, so
  stateless trimming works. A summarizing processor re-summarizes every turn
  and is not journaled as a compaction.
- Pydantic AI's own UI adapters (`pydantic_ai.ui`) aren't used: Sectr serves
  AG-UI from the journal.

## Subagents

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

## Claude Agent SDK

> **🚧 Under Construction 🚧**
