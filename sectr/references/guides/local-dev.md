# Local dev

## The loop

```text
Run the app locally end-to-end: SQLite journal, subprocess runner, gateway for clients. See `dev --help`

Usage: sectr dev [OPTIONS]

Options:
      --path <PATH>            Project directory (must contain sectr.toml) [default: .]
      --config <CONFIG>        Config file path (relative to --path) [default: sectr.toml]
      --no-open                Don't open the dashboard (CI / containers)
      --dashboard <DASHBOARD>  Dashboard to open (the deployed console by default; point it at a local `next dev` when working on the dashboard itself) [env: SECTR_DASHBOARD_URL=] [default: https://app.sectr.dev]
      --port <PORT>            Gateway port. Default: the port this project used last, else 8420. A port given here is strict: if it's busy, `sectr dev` exits
  -h, --help                   Print help

```

`sectr dev` is the local engine in one terminal:

- **SQLite journal + subprocess runner + gateway** — the same turn-based
  model as production, on your machine. Sessions, approvals, and the SSE
  tail behave exactly as they do hosted.
- **The gateway** — bound to `127.0.0.1:8420` by default, serving the
  same sessions API the server does (plus a few `/dev` extras, no auth).
  When a `local.sectr.dev` certificate is installed it's also reachable as
  `https://local.sectr.dev:8420` from any browser.
- **The dashboard** — opened automatically pointed at this gateway
  (skip with `--no-open` in CI). The local journal appears in the same
  session inspector you use for deployed apps.
- **Manifest-aware** — the manifest is re-inspected on the next turn, so
  agent-code edits apply on the following invocation; a changed manifest
  creates the next local deployment version. `sectr dev` holds the engine,
  so leave it running: every other surface is a plain HTTP client of it.
  Ctrl-C drains in-flight turns, then exits.
- **The same approval policy** — `[deploy] approval_tools` in `sectr.toml`
  gates the same tools here as deployed. See
  [gate by deployment config](../../SKILL.md#marking-as-gated-in-deployment-config).

### Ports

The same project keeps its port across runs, so the local URL a frontend
points at stays put. If the port is busy (another project's `sectr dev`, or
anything else), `sectr dev` moves up to the next free one and says so. Two
projects running side by side settle on 8420 and 8421 and keep them.

To pin a port, pass `--port` or set it in `sectr.toml`:

```toml
[dev]
port = 8500
```

A pinned port is strict: if something else holds it, `sectr dev` exits with
an error instead of moving.

Client discovery rides `.agent/dev.json` (gitignored local state): the
running gateway's URL, which `sectr send` and `sectr tail` read.

## sectr send

```text
Send a message / decide an approval through the running `sectr dev` gateway (a thin client — it never starts the engine)

Usage: sectr send [OPTIONS] [MESSAGE]

Arguments:
  [MESSAGE]  The message (positional). Omitted: prompted on the terminal, or read from all of stdin when piped

Options:
      --path <PATH>        Project directory (must contain a `.agent/dev.json` from `sectr dev`) [default: .]
      --session <SESSION>  Existing session (a plain `sectr send "msg"` creates one)
      --approve            Approve the session's pending tool call (resumes the turn)
      --deny               Deny the session's pending tool call (resumes with the rejection)
      --no-tail            Don't follow the turn's output — send, print the session id, exit
      --json               Raw event envelopes (one JSON per line) instead of the human renderer
      --cloud              CLOUD mode (WS J): talk to a sectr-server instead of the local dev gateway. Requires --app; auth/server come from --api/--key, the env, or `sectr auth login` (shared resolution)
      --app <APP>          The app slug (cloud mode) — as `sectr deploy` created it
      --route <ROUTE>      Route path (cloud mode). Default: the app's first route
      --api <API>          Server base (cloud mode). Defaults through `sectr auth login`
      --key <KEY>          Platform key (cloud mode). Defaults through `sectr auth login`
  -h, --help               Print help

```

A thin client of the running gateway — it never starts anything. Send a
message (creating a session when none is given), follow the turn's output in
the terminal, and pass `--approve`/`--deny` when the agent suspends on a
pending tool approval. `--no-tail` prints the session id and exits; `--json`
emits raw event envelopes (one per line) for scripts.

## sectr tail

```text
Follow a session's journal (never stops until ctrl-c)

Usage: sectr tail [OPTIONS] --session <SESSION>

Options:
      --path <PATH>        Project directory (must contain a `.agent/dev.json` from `sectr dev`) [default: .]
      --session <SESSION>  Session to follow (all-sessions tail is future work)
      --cloud              CLOUD mode (WS J): follow a session on a sectr-server (auth/server from --api/--key, the env, or `sectr auth login`)
      --api <API>          Server base (cloud mode). Defaults through `sectr auth login`
      --key <KEY>          Platform key (cloud mode). Defaults through `sectr auth login`
      --json               Raw event envelopes (one JSON per line) instead of the human renderer
  -h, --help               Print help

```

Follow a session's journal — the same stream the dashboard inspector shows,
from your terminal. It never exits until Ctrl-C.

## Cloud mode

`send`/`tail` also talk to a hosted server with `--cloud --app <slug>`
(auth and server base from `sectr auth login` or flags) — the same command
against the same protocol, just pointed at `api.sectr.dev`.

## Next step

[Deploy](../../SKILL.md#deploy-your-agent) the same project unchanged — `sectr dev`
and the hosted server run the identical invoke contract.
