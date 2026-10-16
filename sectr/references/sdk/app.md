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
- **`env`** — the environment variable **names** that should be present for your app's execution. Values are not defined here: in local dev those come from your .env file, and in cloud they are set in the Sectr dashboard (Settings → Secrets).

## Routes

Your app consists of multiple *routes*, each of which corresponds to an agent. If you are using an adapater, you simply use `add_route`:

```python
from sectr.adapters.openai_agents import openai_agents

app.add_route("/chat", openai_agents(agent))
```

If you are using raw code rather than an agent framework adapter, you can use the `@app.agent_route` decorator:

```python
@app.agent_route("/chat")
async def chat(ctx: SessionContext) -> AsyncIterator[RunnerEvent]:
    ...
```

Paths must start with `/` and be unique - Sectr routes sessions by
`(app, path)`.
