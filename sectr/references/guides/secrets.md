# Secrets

Agents need provider keys, database URLs, internal API tokens. Sectr keeps
them in a write-only store: values can be set and removed but never read
back: not by the CLI, not by the dashboard, not by the API. Only the isolated agent runner
receives them as environment variables at spawn time.

## Managing secrets

Secrets are scoped **per project** (the `[app] name` slug in `sectr.toml`). From
the project directory:

```sh
sectr secrets set OPENAI_API_KEY   # prompts for the value, hidden
sectr secrets list                 # names only: values are write-only
sectr secrets rm OPENAI_API_KEY
```

`sectr secrets set NAME=VALUE` also takes the value inline, but leave out
`=VALUE` and it prompts with hidden input, so the value never lands in your
shell history or on screen. All the options are in the
[CLI reference](../cli.md#sectr-secrets-set).

`SECTR_APPROVAL_POLICY` can't be a secret: Sectr sets it from
[`approval_tools`](../../SKILL.md#marking-as-gated-in-deployment-config) in `sectr.toml`.

Auth and server base come from [`sectr auth login`](https://docs.sectr.dev/getting-started/install/#authenticating-the-sectr-cli)
(or `--api` / `--key`, or `SECTR_API_URL` / `SECTR_PLATFORM_KEY`).

## Reaching your agent code

Setting a secret isn't enough, you also must declare the variable name on your `AgentApp`:

```python
app = AgentApp(name="support-agent", env=["OPENROUTER_API_KEY"])
```

The `env` list is the contract: when the agent starts, Sectr sets exactly
these names from the store. Names without a stored value simply
arrive unset; values never pass through the manifest itself.

For **local** dev, secrets come from the project's `.env` file instead:
same names, no cloud store needed.
