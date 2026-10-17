# Using the OpenAI Agents SDK

Sectr runs [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
agents as ordinary SDK code, exposed on a Sectr route. Sectr journals every
turn, streams it to your frontend, and pauses for approval when a gated tool
is called.

This page walks through the support agent that `sectr init` scaffolds on the
OpenAI Agents SDK. You can run it right away:

```sh
sectr init my-agent --template openai-agents
```

Add `--frontend copilotkit` for a CopilotKit chat app wired to it. For the
parts of a project that don't depend on the framework (the manifest, secrets,
running and deploying), see [your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together).

## Install

To add the OpenAI Agents SDK to an existing project instead:

```sh
uv add "sectr[openai-agents]"
```

`sectr[openai-agents]` brings in `openai-agents`, at a version the adapter is
tested against (the [adapter reference](../sdk/adapters/openai-agents.md)
lists the range).

## Wire up the agent

```python
# app.py
from agents import Agent
from sectr import AgentApp
from sectr.adapters.openai_agents import openai_agents

from fake import setup_model
from tools import lookup_order, refund

model = setup_model()  # gpt-4o-mini with OPENAI_API_KEY set, else a key-free mock

agent = Agent(
    name="support",
    instructions=(
        "You are a customer-support agent for a small shop. "
        "Look up order status with the lookup_order tool, "
        "and issue refunds with the refund tool."
    ),
    tools=[lookup_order, refund],
    model=model,
)

app = AgentApp(name="my-agent", env=["OPENAI_API_KEY"])
app.add_route("/chat", openai_agents(agent, model=model))
```

`openai_agents(agent)` is the whole integration: it maps the SDK's streamed
events and its approval interruptions onto Sectr's session events. The agent
is the agent you'd write anyway: no Sectr imports in it, and nothing to
persist between turns.

## Tools and approvals

The tools are vanilla SDK tools:

```python
# tools.py
from agents import function_tool


@function_tool
async def lookup_order(order_id: str) -> dict[str, str]:
    """Look up the status of an order."""
    return {"order_id": order_id, "status": "shipped"}


@function_tool(needs_approval=True)
async def refund(order_id: str) -> str:
    """Refund an order (requires manager approval)."""
    return f"Refunded {order_id}"
```

`needs_approval=True` gates `refund`, with the SDK's own human-in-the-loop.
When the model calls it, the turn pauses until someone approves or denies
it, which can be seconds or weeks later (see [approvals](../../SKILL.md#defining-gated-tools)).
On approval, the SDK runs the tool as it runs any tool: with its guardrails,
your hooks, tracing, the run context and your `run_config`.

To pass a run context, give the adapter a factory:
`openai_agents(agent, context=lambda ctx: ...)`. The
[adapter reference](../sdk/adapters/openai-agents.md) covers it,
along with handoffs, subagents and the current limits.

## Run, deploy, connect

Run and deploy it like any Sectr project: `sectr dev` locally, then
`sectr deploy`, and `sectr secrets set OPENAI_API_KEY` for the deployed app's
key. [Your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) walks through both.

Every route is an [AG-UI](https://docs.ag-ui.com) endpoint, so the agent
connects to a frontend like any other.
[Connecting your frontend](connecting-frontend.md) covers it.
