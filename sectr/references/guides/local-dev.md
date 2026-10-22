# Local dev

## The dev loop

`sectr dev` is the local engine in one terminal (its options are in the
[CLI reference](../cli.md#sectr-dev)):

- **Journal + subprocess runner + gateway** — the same turn-based
  model as cloud Sectr, but on your machine. Sessions, approvals, and the SSE
  tail behave exactly as they do hosted.
- **The gateway** — bound to `127.0.0.1:8420` by default, serving the
  same sessions API the server does, and reachable as `https://local.sectr.dev:8420`
- **The dashboard** — opened automatically pointed at this local Sectr gateway.
- **Hot-reloading** — Agent code is re-loaded on the next turn, so you can keep `sectr dev` running
  while you edit your code and keep chatting with your evolving agent.
- **The same tool approval policy** — Tool approvals are gated the same as in prod, so you can
  accurately test how approvals will work when deployed. See [approvals](../../SKILL.md#defining-gated-tools) for more info.

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

## From the terminal

The dashboard isn't the only client. You can also send messages and approval decisions via the CLI, from a second terminal:

- **[`sectr send`](../cli.md#sectr-send)** sends a message (creating a
  session when none is given) and follows the turn's output. Pass
  `--approve`/`--deny` when the agent suspends on a pending tool approval. See the [CLI reference](../cli.md#sectr-send) for all options
- **[`sectr tail`](../cli.md#sectr-tail)** follows a session's journal,
  the same stream the dashboard inspector shows.

Both also talk to cloud Sectr with `--cloud --app <slug>` (auth and server
base from `sectr auth login` or flags).
