# AgentApp

`sectr.AgentApp` is the one object your project exposes: `[agent] entry` in
`sectr.toml` names it as `module:attribute`, and Sectr imports your module to
find it. [Serving the agent](../../SKILL.md#serving-the-agent)
shows it in use.

## `AgentApp(*, name, env)`

- **`name`** — the project slug. It should match `[app] name` in `sectr.toml`.
- **`env`** — the names of the secrets the agent needs (see
  [secrets](../guides/secrets.md#reaching-your-agent-code)).

## `app.add_route(path, handler)`

Serves `handler` on `path`. With a framework, the handler is the adapter's:
`openai_agents(agent)`, `langgraph(graph)` or `pydantic_ai(agent)` (see the
[adapter reference](adapters/index.md)).

## `@app.agent_route(path)`

The decorator form, for a handler of your own: see
[no framework](no-framework/index.md).

Paths must start with `/` and be unique in the project: Sectr directs sessions by
`(project, path)`. Registering a bad or duplicate path raises `ValueError`.
