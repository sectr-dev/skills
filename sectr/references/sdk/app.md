# AgentApp

`sectr.AgentApp` is an ordered route registry. One per project; the platform
discovers it via `entry = "module:attribute"` in `sectr.toml` by
importing your module.

```python
from sectr import AgentApp

app = AgentApp(name="support-agent", env=["OPENROUTER_API_KEY"])
```

## Constructor

- **`name`** — the app slug used by `sectr deploy` / `sectr secrets` /
  session creation. It should match `[app] name` in `sectr.toml`.
- **`env`** — the names of the secrets the agent needs; their values come from
  `.env` locally and from `sectr secrets` when deployed (see
  [secrets](../guides/secrets.md)).

## Routes

Your app consists of multiple *routes*, each of which corresponds to an agent. If you are using an adapter, you simply use `add_route`:

```python
from sectr.adapters.openai_agents import openai_agents

app.add_route("/chat", openai_agents(agent))
```

Without a framework adapter, register a handler of your own with
`@app.agent_route("/chat")`: see [no framework](no-framework.md).

Paths must start with `/` and be unique - Sectr routes sessions by
`(app, path)`.
