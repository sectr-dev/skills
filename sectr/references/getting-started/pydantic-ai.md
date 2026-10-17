# Using Pydantic AI

Sectr runs [Pydantic AI](https://ai.pydantic.dev/) agents the same way it runs
OpenAI Agents SDK and LangGraph ones: your agent is ordinary `pydantic_ai.Agent`
code, exposed on a Sectr route. Sectr journals every turn, streams it to your
frontend, and pauses for approval when a gated tool is called.

This page walks through the support agent that `sectr init` scaffolds on
Pydantic AI. For the parts of a project that don't depend on the framework
(the manifest, secrets, running and deploying), see
[your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together). You can run it right away:

```sh
sectr init my-agent --template pydantic-ai
```

Add `--frontend copilotkit` for a CopilotKit chat app wired to it.

## Install

To add Pydantic AI to an existing project instead:

```sh
uv add "sectr[pydantic-ai]" "pydantic-ai-slim[openai]"
```

`sectr[pydantic-ai]` brings in `pydantic-ai-slim` (Pydantic AI without any
model provider). Add the provider extra you use (`pydantic-ai-slim[openai]`,
`[openrouter]`, `[anthropic]`, and so on), or install the full `pydantic-ai`.

## Wire up the agent

```python
# app.py
import os

from pydantic_ai import Agent, Tool
from sectr import AgentApp
from sectr.adapters.pydantic_ai import pydantic_ai

from fake import scenario_model
from tools import lookup_order, refund

if api_key := os.environ.get("OPENAI_API_KEY"):
    from pydantic_ai.models.openai import OpenAIChatModel
    from pydantic_ai.providers.openai import OpenAIProvider

    model = OpenAIChatModel("gpt-4o-mini", provider=OpenAIProvider(api_key=api_key))
else:
    model = scenario_model()  # a key-free mock (the template's fake.py)

agent = Agent(
    model,
    instructions=(
        "You are a customer-support agent for a small shop. "
        "Look up order status with the lookup_order tool, "
        "and issue refunds with the refund tool."
    ),
    tools=[lookup_order, Tool(refund, requires_approval=True)],
)

app = AgentApp(name="my-agent", env=["OPENAI_API_KEY"])
app.add_route("/chat", pydantic_ai(agent))
```

`pydantic_ai(agent)` is the whole integration: it maps the agent's streamed
text, reasoning, tool calls and tool results onto Sectr's session events.

Like any Sectr app, the module must import with no key set
([why](../../SKILL.md#serving-the-agent-on-a-route)). A model
*string* such as `Agent("openai:gpt-4o-mini")` raises at construction when its
provider key is missing, so either build the model only when the key is
present (as above), or pass `defer_model_check=True`:

```python
agent = Agent("openai:gpt-4o-mini", defer_model_check=True, ...)
```

## Tools and approvals

Tools are plain functions, with no framework or Sectr imports:

```python
# tools.py
async def lookup_order(order_id: str) -> dict[str, str]:
    """Look up the status of an order."""
    return {"order_id": order_id, "status": "shipped"}


async def refund(order_id: str) -> str:
    """Refund an order (requires manager approval)."""
    return f"Refunded {order_id}"
```

`requires_approval=True` gates `refund`. When the model calls it, the turn
pauses until someone approves or denies it (see
[approvals](../../SKILL.md#defining-gated-tools)).

Resuming is Pydantic AI's own deferred-tool resume, run from the session's
transcript alone. On approval, Pydantic AI runs `refund` itself, exactly once,
through its normal pipeline (argument validation, `RunContext`, `deps`, hooks),
and the model writes only the continuation. On denial, the model is told the
call wasn't approved and the tool never runs.

A tool can also decide for itself, by raising `ApprovalRequired`:

```python
from pydantic_ai import RunContext
from pydantic_ai.exceptions import ApprovalRequired


@agent.tool
def transfer(ctx: RunContext[None], amount: int) -> str:
    """Move money."""
    if amount > 1000 and not ctx.tool_call_approved:
        raise ApprovalRequired(metadata={"reason": "Large transfer"})
    return f"Moved {amount}"
```

A `reason` string in the metadata is shown with the pending approval in the
console. Any other metadata is not journaled.

## How state works

There's no Pydantic AI persistence layer to run: **Sectr's journal is the
history.** Every turn is a fresh `agent.iter(...)` whose `message_history` is
rebuilt from the session's transcript, so a conversation survives deploys,
restarts and scale-to-zero. Nothing needs serializing between turns.

To pass `deps`, give the adapter a factory:
`pydantic_ai(agent, deps=lambda ctx: ...)` calls it once per invocation with
the [`SessionContext`](../sdk/context.md) (it may be `async`), and a
resumed approved tool receives the same `deps`.

Its limits (text output only for now, output validators, auto-approvers) are
in the [adapter reference](../sdk/adapters/pydantic-ai.md), with how the
adapter works.

## Run, deploy, connect

Run and deploy it like any Sectr project: `sectr dev` locally, then
`sectr deploy`, and `sectr secrets set OPENAI_API_KEY` for the deployed app's
key. [Your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) walks through both.

Every route is an [AG-UI](https://docs.ag-ui.com) endpoint, whatever the
framework behind it, so a Pydantic AI route connects to a frontend exactly like
any other. [Connecting your frontend](connecting-frontend.md)
covers it.
