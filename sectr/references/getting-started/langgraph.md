# Using LangGraph

Sectr runs [LangGraph](https://langchain-ai.github.io/langgraph/) agents the
same way it runs OpenAI Agents SDK ones: your graph is ordinary LangGraph
code, exposed on a Sectr route. Sectr journals every turn, streams it to your
frontend, and pauses for approval when a gated tool is called.

This page walks through the support agent from
[your first agent](../../SKILL.md#your-first-agent), rebuilt on LangGraph. You
can run it right away: `sectr init` scaffolds it.

```sh
sectr init my-agent --template langgraph
```

Add `--frontend copilotkit` for a CopilotKit chat app wired to it. The same
agent is also the `/chat_langgraph` route of the
[`sectr-copilotkit-starter`](https://github.com/sectr-dev/sectr-copilotkit-starter),
next to the OpenAI Agents SDK version on `/chat_oai`, with a chat page for
each.

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

Your module must import cleanly with no key set: `sectr dev` imports your app
to find its routes before it injects `.env` secrets, and the OpenAI chat
model raises if it's built without a key. The template falls back to a
scripted model, which also means the first run needs no keys at all; the
runs themselves get the real key and the real model.

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
pauses and the session waits for a decision: in the dashboard's approval
inbox, from the CLI (`sectr send --approve`), or as an interrupt in your chat
UI. On approval, Sectr runs the call exactly once and the graph continues; on
denial, the model is told the call wasn't approved. The decision can come
seconds or weeks later. See [approvals](../../SKILL.md#approvals-human-in-the-loop).

Tools that aren't sectr tools (a LangChain `@tool`, an MCP tool from
`langchain-mcp-adapters`, a community tool) can be gated too:

- On `create_agent`, with LangChain's own `HumanInTheLoopMiddleware`:
  `middleware=[HumanInTheLoopMiddleware(interrupt_on={"refund": True})]`.
  Sectr pauses those calls itself, so the middleware never calls
  `interrupt()`. Sectr's decisions are approve and reject, so each entry
  must allow both (`True` does).
- On any graph, with `as_langgraph_tool(tool, needs_approval=True)`.

Calling LangGraph's `interrupt()` yourself fails loudly instead of pausing:
it needs a checkpointer to resume, and Sectr runs graphs without one.

## How state works

There's no LangGraph checkpointer: **Sectr's journal is the checkpoint.**
Every turn is a fresh run of your graph, with its messages rebuilt from the
session's transcript. So there's no checkpointer database to run, and a
conversation survives deploys, restarts and scale-to-zero.

The model this needs has limits:

- **ReAct-shaped graphs only**, with exactly one top-level `ToolNode` holding
  your sectr tools. A graph compiled with a checkpointer, or with sectr tools
  in several `ToolNode`s or in a subgraph (multi-agent and supervisor
  graphs), fails with a clear error when you build the route.
- **Messages are the only state.** Other state channels don't survive from
  one turn to the next. Keep state in the conversation, for example a
  todo-list tool whose journaled call *is* the write.
- **One approval per turn.** Several gated calls in one model message become
  sequential approvals.

The [adapter reference](../sdk/adapters.md#langgraph) has the details.

## Run, deploy, connect

Run it locally:

```sh
sectr dev
# - 🔌 connect via AG-UI at: https://local.sectr.dev:8420/ag-ui/my-agent/chat
```

The dashboard opens with a playground for the route. To use the real model,
put `OPENAI_API_KEY` in `.env`.

Deploy it, and give the deployed app its key:

```sh
sectr deploy
sectr secrets set OPENAI_API_KEY
```

Every route is an [AG-UI](https://docs.ag-ui.com) endpoint, whatever the
framework behind it, so a LangGraph route connects to a frontend exactly
like any other: no CopilotKit LangGraph integration and no LangGraph
Platform. `sectr init --frontend copilotkit` scaffolds that frontend for you.
To serve several routes from one CopilotKit app (as the starter does with its
OpenAI Agents SDK and LangGraph routes), give each its own `agentId` in the
Runtime's agents map:

```ts
const agents = {
  default: new SectrAgent({ url: "https://api.sectr.dev/ag-ui/support-agent/chat_oai" }),
  langgraph: new SectrAgent({ url: "https://api.sectr.dev/ag-ui/support-agent/chat_langgraph" }),
};
const runtime = new CopilotRuntime({ agents, runner: new SectrRunner({ agents }) });
```

and pick one in the chat:

```tsx
<CopilotSidebar agentId="langgraph" threadId={threadId} />
```

[Connecting your frontend](connecting-frontend.md) covers the
rest: user tokens, approvals as interrupts, and reattaching on refresh.
