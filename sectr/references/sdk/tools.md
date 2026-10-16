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

(Snippet: `examples/raw/tools.py` — the conformance fixture.)

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

A tool is gated when **either** applies:

1. **Code marking** — `@tool(needs_approval=True)`.
2. **Platform policy** — `approval_tools` under `[deploy]` in `sectr.toml`
   can gate a tool the author forgot to mark. This is what makes "refund
   requires approval" enforceable without changing code. Sectr passes the
   list to the runner as `SECTR_APPROVAL_POLICY` (a JSON list of tool
   names), where the SDK reads it; setting that variable yourself is an
   error. See
   [gate by deployment config](../../SKILL.md#gate-by-deployment-config).

A gated call raises `sectr.tools.ApprovalRequired`, which the adapter
converts into the final `ToolApprovalRequired` event — the turn suspends.
See [approvals](../../SKILL.md#approvals-human-in-the-loop) for the full flow, and
[`SessionContext.execute`](context.md) for how the decision
returns.

`is_gated(name)` and `get_tool(name)` are the lookup half of the registry —
mostly adapter machinery, exposed for completeness.

> **Frameworks bypass this module**: Frameworks with native HITL — openai-agents `needs_approval`, claude-code
> permission modes — keep their own tool definitions; the adapter maps their
> interruption onto the same platform contract. `sectr.tools` is for
> hand-rolled agents (the raw example) and as the gating vocabulary the
> platform policy targets.
