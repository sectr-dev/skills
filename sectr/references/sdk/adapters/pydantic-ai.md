# Pydantic AI adapter

For a walkthrough built on a runnable sample, see
[using Pydantic AI](../../getting-started/pydantic-ai.md).

```python
from pydantic_ai import Agent, Tool

from sectr import AgentApp
from sectr.adapters.pydantic_ai import pydantic_ai

agent = Agent(model, tools=[lookup_order, Tool(refund, requires_approval=True)])

app = AgentApp(name="support-agent")
app.add_route("/chat", pydantic_ai(agent))
```

- Requires `pydantic-ai-slim>=2.54`. `sectr[pydantic-ai]` pulls in no model
  provider, so add the one you use. To install, see the [support matrix](../../../SKILL.md#supported-agent-frameworks).
- Signature: `pydantic_ai(agent, *, model=None, deps=None, model_settings=None, usage_limits=None) -> Handler`
  - `model`, `model_settings` and `usage_limits` pass through to `agent.iter`;
    `model` is for deterministic fakes.
  - `deps` is a **factory** called once per invocation with the
    [`SessionContext`](../context.md), sync or `async`. A static value
    is `deps=lambda ctx: value`. A resumed approved tool gets the same `deps`.
  - Raises `TypeError` at construction for anything but a plain
    `pydantic_ai.Agent`: durable-execution wrappers (`TemporalAgent`,
    `DBOSAgent`, `PrefectAgent`) own a second durable store, which conflicts
    with the journal being the checkpoint.

## Behavior

Each turn runs `agent.iter(...)` on a `message_history` rebuilt
from the journal, and streams the run: text and reasoning become `MESSAGE_*`
events (one start/end bracket per model response), tool calls `TOOL_CALL`
(journaled once, after the message they belong to), and tool results
`TOOL_RESULT`. A response's thinking (signatures included) is journaled with
its message and sent back on later turns and approval resumes, to the
provider that issued it, as Pydantic AI does natively. A gated tool (`requires_approval=True`, a tool that raises
`ApprovalRequired`, or the deployment's
[`approval_tools`](../../../SKILL.md#marking-as-gated-in-deployment-config))
suspends the turn with `TOOL_APPROVAL_REQUIRED`. On approval, Pydantic AI resumes from the history
with `DeferredToolResults` and runs the tool itself, once; when denied it
returns the standard rejection text and the tool never runs. System prompts
(`system_prompt=`, static or dynamic) are restored on every turn;
`instructions=` is applied by Pydantic AI as usual. Several gated calls in one
response become sequential approvals, one per turn: the adapter holds the
undecided ones back until their turn. The adapter itself executes nothing,
and sibling calls that already finished aren't re-run. See
[approvals](../../../SKILL.md#defining-gated-tools).

## Output types

Structured output isn't supported yet: an `output_type`
other than `str` (a model, `ToolOutput`, `NativeOutput`, ...) fails
`pydantic_ai(agent)` with a `ValueError`, and the agent's answer is journaled
as text. Pausing needs `DeferredToolRequests` among the run's output types.
Sectr adds it, unless your agent already lists it. If your agent has
`@agent.output_validator`s, Pydantic AI refuses a per-run output type, so
declare `output_type=[str, DeferredToolRequests]` (or `str | DeferredToolRequests`) on the agent yourself;
otherwise the adapter raises a `ValueError` (at construction, and again at the
start of each turn in case a validator was added later).

## Limits to know about

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
  [subagents](../../guides/subagents.md).
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
