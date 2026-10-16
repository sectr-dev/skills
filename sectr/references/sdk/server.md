# sectr.server

`sectr.server` mounts an `AgentApp` as the **runner invoke contract**
(`schema/invocation.schema.json`): `POST /invocations` with
`{session_id, invocation_id, route, transcript, reason}`, answered by an SSE
stream of runner events. You almost never touch this module — `sectr dev`
and the production sidecar do:

- `sectr dev` spawns `uv run python -m sectr.server <entry>` — the module
  form of `main()` below.
- The production sidecar imports your app and calls `run(app)` directly.

## `create_asgi(app)`

Builds the FastAPI app implementing the contract:

1. structurally validates the request body (missing/wrongly-typed fields →
   422\);
2. dispatches by `route` to the registered handler;
3. constructs the `SessionContext` (transcript envelopes + the polymorphic
   reason);
4. streams the handler through `sectr.drive` as SSE — `data:` = one runner
   event JSON per frame.

`sectr.drive` enforces **finality**: exactly one terminal event —
`NO_MORE_ACTIONS` appended on generator exhaustion (turn over) or
`ToolApprovalRequired` last (turn suspended). Handlers can't end a turn
wrongly; the adapter makes the contract impossible to violate by accident.

## `run(app)` / `python -m sectr.server <entry>`

`run(app)` serves on `127.0.0.1:$SECTR_PORT` (default 8080) — the sidecar
sets the port. The CLI form resolves `module:attribute`, imports it, and
serves — that's the entry `sectr dev` uses for the subprocess runner.

> **Why localhost-only**: The runner is a pure function: `(transcript, reason) → event stream`. It
> holds **no platform credentials and no journal access** — the only caller is
> the sidecar, on localhost, with the invocation's events forwarded over the
> sidecar's own connection. That's the trust boundary: user-code compromise
> can't touch other sessions or the platform.
