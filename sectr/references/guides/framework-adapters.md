# Framework adapters

The `sectr` SDK adapts supported frameworks to the platform's invoke
contract — your agent code stays 100% framework code. One line per project:

```python
app.add_route("/chat", openai_agents(agent, model=model))
```

## What an adapter does

Frameworks already have streaming and HITL machinery. The adapter maps them
onto the platform contract (and back):

| Framework concept                      | Platform contract                                                                          |
| -------------------------------------- | ------------------------------------------------------------------------------------------ |
| framework's stream events              | `RunnerEvent`s journaled in order                                                          |
| native approval/interruption machinery | `ToolApprovalRequired` final frame → turn suspended                                        |
| resume after interruption              | fresh invocation, `reason.kind = "approval_decision"`, transcript rebuilt from the journal |
| framework memory/summarization         | journaled `TranscriptCompacted` (see [compaction](compaction.md))                          |

The platform never sees framework internals — the journal stores the
canonical event vocabulary only, which is what makes the session inspector,
the SSE tail, and cross-framework tooling uniform. Full details per page:

- [`openai_agents`](../sdk/adapters.md) — complete: streaming, native
  interruptions, transcript-rebuild resume.
- [`langgraph`](../sdk/adapters.md#langgraph) — complete for ReAct-shaped
  graphs: streaming, exception-based pause, adapter-executed resume. No
  LangGraph checkpointer — the journal is the checkpoint. Walkthrough:
  [using LangGraph](../getting-started/langgraph.md).
- [`pydantic_ai`](../sdk/adapters.md#pydantic-ai) — complete: streaming,
  native deferred-tool pause, resume from the journal alone, sequential
  approvals. Walkthrough: [using Pydantic AI](../getting-started/pydantic-ai.md).
- `claude_code` — deferred; renders the canonical transcript as text.

## Support matrix

|                               | Status                                                                                |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| **OpenAI Agents SDK**         | first-class                                                                           |
| **LangGraph**                 | first-class (ReAct-shaped graphs; see [limits](../sdk/adapters.md#langgraph))         |
| **Anything that serves HTTP** | works today — see the raw example (`examples/raw/`)                                   |
| **Pydantic AI**               | first-class (plain `pydantic_ai.Agent`; see [limits](../sdk/adapters.md#pydantic-ai)) |
| **Claude Agent SDK**          | backlog — not planned for this release                                                |

## How LangGraph maps (no checkpointer)

LangGraph's own human-in-the-loop (`interrupt()` + `Command(resume=...)`)
needs a checkpointer, which would be a second source of truth next to the
journal. The adapter doesn't use one:

- **Every turn is a fresh `graph.astream`.** Its message state is rebuilt from
  the journaled transcript; the journal is the checkpoint.
- **Pause is an exception.** A gated tool converted with `as_langgraph_tool`
  raises `ApprovalRequired` inside the tool node. It is control flow, not an
  error, so it escapes LangGraph's tool error handling. The adapter journals
  the model's tool call and ends the turn with `ToolApprovalRequired`. Any
  other tool gated in code (by `HumanInTheLoopMiddleware`'s `interrupt_on`,
  or `as_langgraph_tool(tool, needs_approval=True)`) or by the deployment's
  [`approval_tools`](../../SKILL.md#marking-as-gated-in-deployment-config)
  pauses the same way, from the adapter's wrapper around the tool node; the
  middleware never calls `interrupt()`.
- **Resume re-enters the graph at its tool node.** On the approval turn the
  adapter rebuilds the paused state from the journal (a throwaway in-memory
  checkpointer for that one turn), and the graph's own tool node runs the
  approved call (or answers it with the rejection text) and routes on from
  there; the model generates only the continuation.

## How Pydantic AI maps (native deferred tools)

Pydantic AI can resume a paused run from message history alone, which is
exactly Sectr's "transcript in, events out" model. So this adapter is the
thinnest of the three:

- **Every turn is a fresh `agent.iter(...)`.** Its `message_history` is rebuilt
  from the journaled transcript; there is no Pydantic AI persistence and no
  serialized run state.
- **Pause is native.** A tool marked `requires_approval=True` (or one that
  raises `ApprovalRequired`) ends the run with `DeferredToolRequests`. The
  adapter journals it as `ToolApprovalRequired`.
- **Resume is Pydantic AI's.** On the approval turn the adapter passes the
  decision as `DeferredToolResults`; Pydantic AI runs the approved tool itself,
  through its normal pipeline, and continues. The adapter executes nothing, and
  nothing executes twice: sibling calls that already finished are not re-run.
- **Several gated calls** in one response are approved one per turn: the
  adapter holds the undecided ones back until their turn comes.

## Which to pick

- **openai-agents** — production-ready today; the
  [first-agent guide](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) is built on it.
- **LangGraph** — if your agent is already a LangGraph graph, or you want
  LangChain's model and tool ecosystem. ReAct-shaped graphs are supported;
  start with [using LangGraph](../getting-started/langgraph.md).
- **Pydantic AI** — if your agent is a `pydantic_ai.Agent`, or you want typed
  tools and the broadest model-provider support. Approvals
  resume natively; start with [using Pydantic AI](../getting-started/pydantic-ai.md).
- **No framework** — [`sectr.tools`](../sdk/tools.md) +
  [`SessionContext`](../sdk/context.md) hand you the whole agentic
  loop: the raw example (`examples/raw/`) is the reference, and the clearest
  way to learn what the journal captures in every case.
- Anything else that streams text and tool calls over an async generator —
  write a small adapter yourself (the contract is four rules, listed in the
  [adapters reference](../sdk/adapters.md)).
