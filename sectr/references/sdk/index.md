# SDK reference

The `sectr` SDK (PyPI: `sectr`) is deliberately small - it only needs to translate the events of your agent into Sectr's internal event format, and handle resumption of your agent.

Install it in your agent project with:

```sh
uv add sectr
```

See [your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together) for a guided walkthrough of building an agent, the primary class of [AgentApp](app.md), or the available [framework adapters](adapters.md).
