# CLI reference

Below is the full CLI command reference. For the
workflows behind the commands, get setup with
[install](https://docs.sectr.dev/getting-started/install/), and the understand [the Sectr dev loop](../SKILL.md#conceptual-overview-how-a-sectr-project-fits-together).

## Authenticating the CLI

### `sectr auth login`

```text
Sign in: opens the dashboard in your browser to approve the request (or pass --key / --with-key to store an API key instead)

Usage: sectr auth login [OPTIONS]

Options:
      --with-key    Read the platform key from a hidden prompt or stdin instead of using the browser
      --no-browser  Print the approval link without trying to open a browser (remote machines, coding agents)
      --api <API>   The server base URL. Default: the public endpoint
      --key <KEY>   Platform API key (`skp_…`). With `login`, stores that key instead of opening the browser
  -h, --help        Print help

```

### `sectr auth whoami`

```text
Print the stored identity (org + server) after re-validating

Usage: sectr auth whoami [OPTIONS]

Options:
      --api <API>  The server base URL. Default: the public endpoint
      --key <KEY>  Platform API key (`skp_…`). With `login`, stores that key instead of opening the browser
  -h, --help       Print help

```

### `sectr auth logout`

```text
Forget the stored key (and revoke it, if the browser login made it)

Usage: sectr auth logout [OPTIONS]

Options:
      --api <API>  The server base URL. Default: the public endpoint
      --key <KEY>  Platform API key (`skp_…`). With `login`, stores that key instead of opening the browser
  -h, --help       Print help

```

## Project Setup

### `sectr init`

```text
Scaffold a new agent project: framework and frontend pickers (or --template / --frontend), file emission, `uv sync`, and a smoke inspect. See `init --help`

Usage: sectr init [OPTIONS] [NAME]

Arguments:
  [NAME]  Project directory name (created if missing; refused if non-empty unless --force)

Options:
      --template <TEMPLATE>  Agent framework: `openai-agents`, `langgraph` or `pydantic-ai`. Given → fully non-interactive (docs/CI). Omitted → interactive picker (needs a terminal)
      --frontend <FRONTEND>  Chat frontend: `none` (agent only) or `copilotkit` (a Next.js + CopilotKit app at the root, the agent in `agent/`; needs Node 20+). Omitted → `none` with --template, else an interactive picker
      --force                Scaffold into a non-empty directory anyway
  -h, --help                 Print help

```

## Local Dev Server

### `sectr dev`

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

### `sectr send`

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

### `sectr tail`

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

## Sectr Cloud Secrets Management

### `sectr secrets set`

```text
Store a secret (prompt hides input when `=VALUE` is omitted — avoiding both shell history and terminal echo)

Usage: sectr secrets set [OPTIONS] <KEY_VALUE>

Arguments:
  <KEY_VALUE>  

Options:
      --path <PATH>      Project directory (must contain sectr.toml — the app is identified by its slug, exactly as `sectr deploy` creates it) [default: .]
      --config <CONFIG>  sectr.toml location relative to the project [default: sectr.toml]
      --api <API>        The server base URL (`http(s)://host[:port]`). Defaults through `sectr auth login` (or SECTR_API_URL)
      --key <KEY>        Platform API key (`skp_…`). Defaults through `sectr auth login` (or SECTR_PLATFORM_KEY)
  -h, --help             Print help

```

### `sectr secrets list`

```text
List secret NAMES for the app (values are write-only)

Usage: sectr secrets list [OPTIONS]

Options:
      --path <PATH>      Project directory (must contain sectr.toml — the app is identified by its slug, exactly as `sectr deploy` creates it) [default: .]
      --config <CONFIG>  sectr.toml location relative to the project [default: sectr.toml]
      --api <API>        The server base URL (`http(s)://host[:port]`). Defaults through `sectr auth login` (or SECTR_API_URL)
      --key <KEY>        Platform API key (`skp_…`). Defaults through `sectr auth login` (or SECTR_PLATFORM_KEY)
  -h, --help             Print help

```

### `sectr secrets rm`

```text
Remove a secret (idempotent)

Usage: sectr secrets rm [OPTIONS] <KEY>

Arguments:
  <KEY>  

Options:
      --path <PATH>      Project directory (must contain sectr.toml — the app is identified by its slug, exactly as `sectr deploy` creates it) [default: .]
      --config <CONFIG>  sectr.toml location relative to the project [default: sectr.toml]
      --api <API>        The server base URL (`http(s)://host[:port]`). Defaults through `sectr auth login` (or SECTR_API_URL)
  -h, --help             Print help

```

## Deploy to Sectr Cloud

### `sectr deploy`

```text
Ship the project to a server: package → upload → the server builds and activates. See `deploy --help`

Usage: sectr deploy [OPTIONS]

Options:
      --path <PATH>                  Project directory (must contain sectr.toml) [default: .]
      --config <CONFIG>              sectr.toml location relative to the project [default: sectr.toml]
      --api <API>                    The server base URL (`http(s)://host[:port]`). Defaults through `sectr auth login` (or SECTR_API_URL)
      --key <KEY>                    Platform API key (`skp_…`). Defaults through `sectr auth login` (or SECTR_PLATFORM_KEY)
      --timeout-secs <TIMEOUT_SECS>  How long to wait for the server-side build to resolve, in seconds [default: 600]
  -h, --help                         Print help

```
