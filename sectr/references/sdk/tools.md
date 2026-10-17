# Tools

`sectr.tools` is the framework-free tool story: plain functions become
tools with a JSON schema derived from their signature, and approval gating
is evaluated locally — no runtime round-trip to the platform.

```python
from sectr.tools import tool

@tool
async def lookup_order(order_id: str) -> dict[str, str]:
    """Look up the status of an order."""
    return {"order_id": order_id, "status": "shipped"}

@tool(needs_approval=True)
async def refund(order_id: str) -> str:
    """Refund an order (requires manager approval)."""
    return f"Refunded {order_id}"
```

## What `@tool` builds

- **`name`** — from the function name (`@tool(name="…")` to override).
- **`description`** — from the docstring (`description=…` to override).
- **`openai_schema`** — a chat-completions `tools=[…]` entry whose
  `parameters` JSON schema is derived from the signature's type annotations
  (un-annotated parameters accept any JSON; parameters with defaults are
  optional). Feed it to your provider's tool list:
  `[t.openai_schema for t in TOOLS]`.
- Calling a `Tool` awaits the function if it's async.

## Approval gating

`@tool(needs_approval=True)` gates a tool: calling it raises
`sectr.tools.ApprovalRequired`, and the turn suspends until someone approves
or denies the call. The deployment's `approval_tools` can gate any tool too.
[Approvals](../../SKILL.md#defining-gated-tools) covers both.

`is_gated(name)` and `get_tool(name)` are the lookup half of the registry,
mostly used by adapters.

## Who uses it

`sectr.tools` is for agents written [without a framework](no-framework/index.md),
which run its tools with `ctx.execute`, and for LangGraph, which converts its
tools with `as_langgraph_tool` (see the
[LangGraph adapter](adapters/langgraph.md)). The OpenAI Agents SDK
and Pydantic AI define tools their own way, with their own approval flags.
