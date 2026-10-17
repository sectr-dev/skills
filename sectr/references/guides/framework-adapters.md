# Choosing a framework

Your agent stays ordinary framework code. The `sectr` SDK's adapter for the
framework is the one line that serves it on a route:

```python
app.add_route("/chat", openai_agents(agent))
```

## What an adapter does

Frameworks already have streaming and human-in-the-loop machinery. The
adapter maps them onto Sectr's session model, and back:

| Framework concept                                 | On Sectr                                                    |
| ------------------------------------------------- | ----------------------------------------------------------- |
| the framework's stream of messages and tool calls | events journaled in order, streamed to clients              |
| the framework's approval request                  | the turn suspends until someone decides                     |
| resuming after the decision                       | a fresh run, with the conversation rebuilt from the journal |
| the framework's memory or summarization           | a journaled compaction (see [compaction](compaction.md))    |

The journal only stores Sectr's own event vocabulary, never framework
internals, which is what makes the dashboard, the session stream and every
other client the same whichever framework you use.

## Which to pick

- **The OpenAI Agents SDK** if your agent is already built on it. Start with
  [using the OpenAI Agents SDK](../getting-started/openai-agents.md).
- **LangGraph** if your agent is already a LangGraph graph, or you want
  LangChain's model and tool ecosystem. ReAct-shaped graphs are supported;
  start with [using LangGraph](../getting-started/langgraph.md).
- **Pydantic AI** if your agent is a `pydantic_ai.Agent`, or you want typed
  tools and the broadest model-provider support. Start with
  [using Pydantic AI](../getting-started/pydantic-ai.md).
- **No framework** if you'd rather write the agent loop yourself:
  [`sectr.tools`](../sdk/tools.md) and
  [`SessionContext`](../sdk/context.md) give you everything, and the raw
  example (`examples/raw/`) is the reference.

The Claude Agent SDK has no adapter yet; it's on the backlog.
