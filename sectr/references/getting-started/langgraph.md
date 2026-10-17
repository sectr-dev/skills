# Using LangGraph

Sectr runs [LangGraph](https://langchain-ai.github.io/langgraph/) agents the
same way it runs OpenAI Agents SDK ones: your graph is ordinary LangGraph
code, exposed on a Sectr route. Sectr journals every turn, streams it to your
frontend, and pauses for approval when a gated tool is called.

This page walks through the support agent that `sectr init` scaffolds on
LangGraph. For the parts of a project that don't depend on the framework (the
manifest, secrets, running and deploying), see
[your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together). You can run it right away:

```sh
sectr init my-agent --template langgraph
```

Add `--frontend copilotkit` for a CopilotKit chat app wired to it.

## Install

To add LangGraph to an existing project instead:

```sh
uv add "sectr[langgraph]" langchain langchain-openai
```

`sectr[langgraph]` brings in `langgraph` and `langchain-core`. `langchain`
provides `create_agent` and `init_chat_model`, and `langchain-openai` the
OpenAI chat model; use another provider's package if you prefer.

## Wire up the graph

```python
# app.py
import os

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from sectr import AgentApp
from sectr.adapters.langgraph import as_langgraph_tool, langgraph

from fake import ScriptedChatModel
from tools import TOOLS

if api_key := os.environ.get("OPENAI_API_KEY"):
    model = init_chat_model("openai:gpt-4o-mini", api_key=api_key)
else:
    model = ScriptedChatModel()  # a key-free mock (the template's fake.py)

graph = create_agent(
    model,
    tools=[as_langgraph_tool(t) for t in TOOLS],
    system_prompt=(
        "You are a customer-support agent for a small shop. "
        "Look up order status with the lookup_order tool, "
        "and issue refunds with the refund tool."
    ),
)

app = AgentApp(name="my-agent", env=["OPENAI_API_KEY"])
app.add_route("/chat", langgraph(graph))
```

`langgraph(graph)` is the whole integration: it maps the graph's streamed
messages and tool calls onto Sectr's session events. Note there's no
checkpointer in `create_agent`; see [how state works](#how-state-works).

Like any Sectr app, the module must import with no key set
([why](../../SKILL.md#serving-the-agent-on-a-route)), and
LangChain's OpenAI chat model raises if it's built without one: hence the
fallback to a scripted model above.

A hand-built graph works too, as long as it's ReAct-shaped: an agent node
plus one `ToolNode` holding your sectr tools.

```python
from langgraph.graph import END, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode

lc_tools = [as_langgraph_tool(t) for t in TOOLS]
bound = model.bind_tools(lc_tools)  # `model` as above


async def call_model(state: MessagesState):
    return {"messages": [await bound.ainvoke(state["messages"])]}


def route_tools(state: MessagesState):
    return "tools" if state["messages"][-1].tool_calls else END


builder = StateGraph(MessagesState)
builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode(lc_tools))
builder.add_conditional_edges("agent", route_tools, ["tools", END])
builder.add_edge("tools", "agent")
builder.set_entry_point("agent")
graph = builder.compile()  # no checkpointer
```

## Tools and approvals

Tools are plain [sectr tools](../sdk/tools.md), converted for
LangGraph with `as_langgraph_tool`:

```python
# tools.py
from sectr.tools import tool


@tool
async def lookup_order(order_id: str) -> dict[str, str]:
    """Look up the status of an order."""
    return {"order_id": order_id, "status": "shipped"}


@tool(needs_approval=True)
async def refund(order_id: str) -> str:
    """Refund an order (requires manager approval)."""
    return f"Refunded {order_id}"


TOOLS = [lookup_order, refund]
```

`needs_approval=True` gates `refund`. When the model calls it, the turn
pauses until someone approves or denies it. On approval, the graph's own tool
node runs the call, exactly once, and the graph continues; on denial, the
model is told the call wasn't approved.

Tools that aren't sectr tools (a LangChain `@tool`, an MCP tool from
`langchain-mcp-adapters`, a community tool) can be gated too, with
`as_langgraph_tool(tool, needs_approval=True)` or LangChain's
`HumanInTheLoopMiddleware`. LangGraph's own `interrupt()` doesn't work on
Sectr. [Approvals](../../SKILL.md#langgraph) shows each option.

## How state works

There's no LangGraph checkpointer: **Sectr's journal is the checkpoint.**
Every turn is a fresh run of your graph, with its messages rebuilt from the
session's transcript. So there's no checkpointer database to run, and a
conversation survives deploys, restarts and scale-to-zero.

That model has limits: ReAct-shaped graphs only, and messages are the only
state that carries from one turn to the next. The
[adapter reference](../sdk/adapters/langgraph.md) lists them, with how
the adapter works.

## Run, deploy, connect

Run and deploy it like any Sectr project: `sectr dev` locally, then
`sectr deploy`, and `sectr secrets set OPENAI_API_KEY` for the deployed app's
key. [Your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) walks through both.

Every route is an [AG-UI](https://docs.ag-ui.com) endpoint, whatever the
framework behind it, so a LangGraph route connects to a frontend exactly
like any other: no CopilotKit LangGraph integration and no LangGraph
Platform. [Connecting your frontend](connecting-frontend.md)
covers it.
