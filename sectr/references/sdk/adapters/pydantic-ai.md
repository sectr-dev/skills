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
  - `deps` is a **factory** called once per invocation with the turn's
    `SessionContext` (its `ctx.session_id` identifies the session), sync or
    `async`. A static value is `deps=lambda ctx: value`. A resumed approved
    tool gets the same `deps`.
  - Raises `TypeError` at construction for anything but a plain
    `pydantic_ai.Agent`: durable-execution wrappers (`TemporalAgent`,
    `DBOSAgent`, `PrefectAgent`) own a second durable store, which conflicts
    with the journal being the checkpoint.

## Behavior

- **Each turn runs `agent.iter(...)`** on a `message_history` rebuilt from the
  journal: there's no persistence to set up.
- **Thinking carries over.** A response's thinking (signatures included) is
  sent back on later turns and approval resumes, to the provider that issued
  it, as Pydantic AI does natively.
- **System prompts** (`system_prompt=`, static or dynamic) are restored on
  every turn; `instructions=` is applied by Pydantic AI as usual.
- **Gated tools pause the turn**: `requires_approval=True`, a tool that raises
  `ApprovalRequired`, or the deployment's
  [`approval_tools`](../../../SKILL.md#marking-as-gated-in-deployment-config).
- **An approved call runs once**: Pydantic AI resumes from the history with
  `DeferredToolResults` and runs the tool itself. A denied call gets the
  standard rejection text and never runs. Sibling calls that already finished
  aren't re-run.
- **Several gated calls in one response** become sequential approvals, one
  per turn. See [approvals](../../../SKILL.md#defining-gated-tools).

**You don't need to add `DeferredToolRequests` to your output types.** Pausing
needs it among the run's output types, and Sectr adds it to each run for you.
The one exception is an agent with `@agent.output_validator`s: Pydantic AI
refuses a per-run output type then, so declare
`output_type=[str, DeferredToolRequests]` (or `str | DeferredToolRequests`)
on the agent yourself. Otherwise the adapter raises a `ValueError`, at
construction and again at the start of each turn (in case a validator was
added later).

## Limits to know about

- **Structured output isn't supported yet.** An `output_type` other than
  `str` (a model, `ToolOutput`, `NativeOutput`, ...) fails `pydantic_ai(agent)`
  with a `ValueError`. The agent's answer is journaled as text.

- **External tool results aren't supported yet.** A tool that raises
  `CallDeferred` (or has `kind='external'`) ends the turn with an
  `ERROR` (`pydantic_ai_external_call`), because Sectr has no external
  tool-result path yet.

- **Auto-approvers bypass your own gates.** A `HandleDeferredToolCalls`
  capability on your agent approves natively gated tools without asking. Tools
  named in `approval_tools` (including MCP and other toolset tools)
  still pause.

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

### Subagents

A subagent is another agent your agent runs from a tool. On Sectr it's an
ordinary tool, and an opaque one: the journal, and every client streaming the
session, see the tool call and its result, but not the subagent's own messages
and tool calls, so later turns don't see those either.

To require approval for what a subagent does, gate its tool: approving it runs
the whole subagent, and denying it runs none of it. A gated tool *inside* a
subagent isn't supported yet, because Sectr can't pause a turn partway through
one: such a tool never runs. Instead the agent is rejected when the runner
starts, or the turn ends with an `ERROR` whose code is `subagent_gated_tool`.

An `escalate` tool that runs a `specialist` agent, gated as a whole:

```python
specialist = Agent(model, instructions="...", tools=[track_parcel])

async def escalate(ctx: RunContext, question: str) -> str:
    """Escalate a question to the shipping specialist."""
    result = await specialist.run(question, usage=ctx.usage)
    return result.output

agent = Agent(model, tools=[Tool(escalate, requires_approval=True)])
```

Where your own code starts the subagent or handles what it returns, the
adapter can't see its run:

- **The policy doesn't reach a delegate's tools.** A tool
  `approval_tools` names on an agent your tool runs (`specialist.run(...)`)
  runs **without approval**. Gate the delegating tool instead.
- **A delegate that allows `DeferredToolRequests` doesn't fail
  the turn.** A delegate with `output_type=[str, DeferredToolRequests]` hands
  a gated tool's requests back to your tool instead of raising. The gated tool
  doesn't run, but the turn carries on with whatever your tool returns. Gate
  the delegating tool instead.
