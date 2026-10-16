# CLI reference

Below is the full CLI command reference. For the
workflows behind the commands, get setup with
[install](https://docs.sectr.dev/getting-started/install/), and the understand [the Sectr dev loop](../SKILL.md#conceptual-overview-how-a-sectr-app-fits-together).

## Authenticating the CLI

### `sectr auth login`

```text
Validate and store the platform key (the server base is stored too)

Usage: sectr auth login [OPTIONS]

Options:
      --api <API>  The server base URL. Default: the public endpoint
      --key <KEY>  Platform API key (`skp_…`). Omitted for `login`: prompted with hidden input
  -h, --help       Print help

```

### `sectr auth whoami`

```text
Print the stored identity (org + server) after re-validating

Usage: sectr auth whoami [OPTIONS]

Options:
      --api <API>  The server base URL. Default: the public endpoint
      --key <KEY>  Platform API key (`skp_…`). Omitted for `login`: prompted with hidden input
  -h, --help       Print help

```

### `sectr auth logout`

```text
Forget the stored key

Usage: sectr auth logout [OPTIONS]

Options:
      --api <API>  The server base URL. Default: the public endpoint
      --key <KEY>  Platform API key (`skp_…`). Omitted for `login`: prompted with hidden input
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
