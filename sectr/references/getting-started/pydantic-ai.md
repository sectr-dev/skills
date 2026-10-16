# Using Pydantic AI

Sectr runs [Pydantic AI](https://ai.pydantic.dev/) agents the same way it runs
OpenAI Agents SDK and LangGraph ones: your agent is ordinary `pydantic_ai.Agent`
code, exposed on a Sectr route. Sectr journals every turn, streams it to your
frontend, and pauses for approval when a gated tool is called.

This page walks through the support agent from
[your first agent](../../SKILL.md#your-first-agent), rebuilt on Pydantic AI. You
can run it right away: `sectr init` scaffolds it.

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

Your module must import cleanly with no key set: `sectr dev` imports your app
to find its routes before it injects `.env` secrets. A model *string* such as
`Agent("openai:gpt-4o-mini")` raises at construction when its provider key is
missing, so either build the model only when the key is present (as above),
or pass `defer_model_check=True`:

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
pauses and the session waits for a decision: in the dashboard's approval
inbox, from the CLI (`sectr send --approve`), or as an interrupt in your chat
UI. The decision can come seconds or weeks later. See
[approvals](../../SKILL.md#approvals-human-in-the-loop).

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

- **One approval per turn.** If the model calls several gated tools in one
  response, you approve them one at a time: after you decide the first, the
  turn pauses again on the next. Nothing runs twice, and a call you haven't
  decided yet never runs early.
- **Pass `deps` with a factory.** `pydantic_ai(agent, deps=lambda ctx: ...)`
  calls your factory once per invocation with the
  [`SessionContext`](../sdk/context.md) (it may be `async`). A resumed
  approved tool receives the same `deps`.
- **Text output only, for now.** Structured output (an `output_type` other
  than `str`) isn't supported yet: `pydantic_ai(agent)` raises a `ValueError`.
- **Output validators.** To pause for approval, a run needs
  `DeferredToolRequests` among its output types. Sectr adds it for you, unless
  your agent has `@agent.output_validator`s: Pydantic AI refuses a per-run
  output type then, so declare it yourself:
  `Agent(..., output_type=[str, DeferredToolRequests])`. Sectr raises a
  `ValueError` that says so if you forget.
- **Auto-approvers bypass the gate you wrote.** A
  `HandleDeferredToolCalls` capability on your agent approves your natively
  gated tools without asking anyone; that's your explicit choice. Tools gated
  by deployment config
  ([`approval_tools`](../../SKILL.md#gate-by-deployment-config))
  still pause.

The [adapter reference](../sdk/adapters.md#pydantic-ai) has the details
and the current limits.

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
framework behind it, so a Pydantic AI route connects to a frontend exactly like
any other. [Connecting your frontend](connecting-frontend.md)
covers user tokens, approvals as interrupts, and reattaching on refresh.
