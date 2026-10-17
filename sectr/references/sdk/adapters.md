# Framework adapters

Each adapter's signature, behavior and limits. For which frameworks are
supported, how to install each, and which to pick, see
[choosing a framework](https://docs.sectr.dev/guides/framework-adapters/).

## OpenAI Agents SDK

For a walkthrough built on a runnable sample, see
[using the OpenAI Agents SDK](../getting-started/openai-agents.md).

```python
from sectr import AgentApp
from sectr.adapters.openai_agents import openai_agents

app = AgentApp(name="support-agent")
app.add_route("/chat", openai_agents(agent))
```

- Requires `openai-agents` 0.22.0 through 0.23.1 (newer releases are added as
  they're tested). To install, see the [support matrix](../../SKILL.md#supported-agent-frameworks).
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
  runs. See [subagents](../guides/subagents.md).
- MCP servers (`mcp_servers`) are connected for each invocation, on the
  agent and on its subagents.
- Reasoning is sent back to the model on later turns and approval resumes,
  as the SDK sends it from its own history: each response's reasoning items
  (encrypted content included) are journaled with its message.

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

- Requires `langgraph>=1.2.11`, `langgraph-prebuilt>=1.1.0` and
  `langchain-core>=1.6.6`. To install, see the [support matrix](../../SKILL.md#supported-agent-frameworks).
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
  see [subagents](../guides/subagents.md));
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

**How the adapter works.** LangGraph's own human-in-the-loop (`interrupt()`
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
  supported. See [subagents](../guides/subagents.md).

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

- Requires `pydantic-ai-slim>=2.54`. `sectr[pydantic-ai]` pulls in no model
  provider, so add the one you use. To install, see the [support matrix](../../SKILL.md#supported-agent-frameworks).
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
response become sequential approvals, one per turn: the adapter holds the
undecided ones back until their turn. The adapter itself executes nothing,
and sibling calls that already finished aren't re-run. See
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
  [subagents](../guides/subagents.md).
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
