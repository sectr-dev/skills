# Secrets

Agents need provider keys, database URLs, internal API tokens. Sectr keeps
them in a **write-only** store: values can be set and removed but never read
back — not by the CLI, not by the dashboard, not by the API. The runner
receives them as environment variables at spawn time.

## sectr secrets

```text
Manage the app's server-side secrets (write-only: set/list/rm)

Usage: sectr secrets [OPTIONS] <COMMAND>

Commands:
  set   Store a secret (prompt hides input when `=VALUE` is omitted — avoiding both shell history and terminal echo)
  list  List secret NAMES for the app (values are write-only)
  rm    Remove a secret (idempotent)
  help  Print this message or the help of the given subcommand(s)

Options:
      --path <PATH>      Project directory (must contain sectr.toml — the app is identified by its slug, exactly as `sectr deploy` creates it) [default: .]
      --config <CONFIG>  sectr.toml location relative to the project [default: sectr.toml]
      --api <API>        The server base URL (`http(s)://host[:port]`). Defaults through `sectr auth login` (or SECTR_API_URL)
      --key <KEY>        Platform API key (`skp_…`). Defaults through `sectr auth login` (or SECTR_PLATFORM_KEY)
  -h, --help             Print help

```

Secrets are scoped **per app** (the `[app] name` slug from `sectr.toml` in
the project directory). Set a value:

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

Omit `=VALUE` and the CLI prompts with hidden input — the value never lands
in shell history or on screen. List names only:

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

`SECTR_APPROVAL_POLICY` can't be a secret: Sectr sets it from
[`approval_tools`](../../SKILL.md#gate-by-deployment-config) in `sectr.toml`.

Auth and server base come from [`sectr auth login`](https://docs.sectr.dev/getting-started/install/#authenticating-the-sectr-cli)
(or `--api` / `--key`, or `SECTR_API_URL` / `SECTR_PLATFORM_KEY`).

## Reaching your agent code

Setting a secret isn't enough — declare the variable name on your `AgentApp`:

```python
app = AgentApp(name="support-agent", env=["OPENROUTER_API_KEY"])
```

The `env` list is the manifest contract: at spawn the sidecar injects
exactly these names from the store. Names without a stored value simply
arrive unset; values never pass through the manifest itself.

For **local** dev, secrets come from the project `.env` file instead (the
`EnvFileSecretSource`) — same names, no cloud store needed. That's why the
scaffold ships `.env.example`.

> **Write-only by design**: The dashboard and API list secret *names* (with creation timestamps) but
> expose no value, no prefix, no hash to compare against. Rotate freely —
> `set` atomically replaces the value; the next spawned runner picks it up
> (in-flight sessions keep the env they started with).
