---
name: sectr
description: >-
  Build, run locally, and deploy AI agents on Sectr, a PaaS for hosting AI
  agents. Covers the sectr CLI (init, dev, send, tail, deploy, secrets, auth);
  the sectr Python SDK (AgentApp and the adapters for the OpenAI Agents SDK,
  LangGraph and Pydantic AI); human-in-the-loop tool approvals; connecting a
  frontend over AG-UI (CopilotKit); and the public HTTP API (sessions, SSE
  journal streaming, approvals, AG-UI endpoints, limits). Use when writing or
  debugging a Sectr agent, integrating the sectr SDK, deploying to Sectr Cloud,
  putting a chat UI on a deployed agent, driving deployed agents over the HTTP
  API, or working with Sectr concepts like journals, turns, and approvals.
---



# Sectr

Sectr is a PaaS for hosting AI agents: develop with any agent framework,
debug locally with `sectr dev`, and deploy to Sectr Cloud with
`sectr deploy`. Agents are written with the OpenAI Agents SDK, LangGraph,
Pydantic AI, or no framework at all (see "Supported agent frameworks"
below). Every deployed route
is also an AG-UI endpoint, so a CopilotKit chat (or any AG-UI client)
connects to it directly. Deployed apps run under concurrency and rate
limits (`references/api/limits.md`).

The runtime is built around a few invariants that shape everything
else — read these first, the details below all follow from them:

- **The journal is the single source of truth.** A session is an
  append-only event log; every live view (the chat stream, the dashboard,
  `sectr tail`) is a tail of that journal. Client SSE is always served by
  tailing the journal, never proxied from a runner.
- **Runners are stateless.** Your agent code is a pure function of
  `(transcript, reason)` — every turn re-invokes the handler with the full
  transcript rebuilt from the journal. State that should survive belongs
  in the conversation itself (journaled tool results), not in memory.
- **Sessions are turn-based.** A turn starts with a `user_message`
  invocation and ends at the `NO_MORE_ACTIONS` sentinel — which the
  platform appends itself; emitting it from user code is an error.
- **Approvals pause the turn.** A gated tool call suspends the turn
  (no turn-end sentinel is journaled); the human's decision resumes it as
  a fresh invocation whose reason is `approval_decision`.

## Before anything else: check that Sectr is set up

Do this before running any `sectr` command, and fix whatever is missing.
Never assume it's already installed.

1. **The `sectr` CLI.** Run `sectr --version`.
   - If it prints a version, go on.
   - If the command isn't found, run `~/.local/bin/sectr --version`. The
     installer puts `sectr` there, but a shell started before the install
     doesn't have it on its `PATH`. If that works, run
     `export PATH="$HOME/.local/bin:$PATH"` and go on.
   - Otherwise install it:
     `curl -fsSL https://sectr.dev/install.sh | sh`, then
     `export PATH="$HOME/.local/bin:$PATH"`, then check `sectr --version`
     again.
2. **`uv`.** `sectr init` and `sectr dev` run the agent's Python with uv.
   Run `uv --version`. If it isn't found, install it with
   `curl -LsSf https://astral.sh/uv/install.sh | sh`, put
   `$HOME/.local/bin` on the `PATH` as above, and check again.
3. **A Sectr project.** A project is a directory with a `sectr.toml`.
   - If there's none and the user wants a new agent, create one with
     `sectr init <name> --template <framework>`, where the framework is
     `openai-agents`, `langgraph` or `pydantic-ai`. Always pass
     `--template`: without it `sectr init` asks questions in the terminal,
     which you can't answer. Add `--frontend copilotkit` only if the user
     wants a chat UI.
   - If the user has an existing Python agent, add the SDK for its
     framework instead: the `uv add` command in "Supported agent
     frameworks" below.
4. **Logged in, only before a cloud command** (`sectr deploy`,
   `sectr secrets`, `sectr send --cloud`; local `sectr dev` doesn't need
   it). Run `sectr auth whoami`. If it fails or prints
   `no stored credentials`, ask the user to:
   1. sign in at https://app.sectr.dev with GitHub or Google, and click
      **Create CLI key** (in the Getting-started checklist, or
      Settings → API keys);
   2. run `sectr auth login` in their own terminal, where it asks for the
      key with hidden input.

   The key is a credential: don't ask the user to paste it into the
   conversation, and don't pass it with `--key` yourself. When they say
   they're done, run `sectr auth whoami` again.

## Supported agent frameworks

Sectr runs agents written with these. Each framework's agent is served
through an adapter from the `sectr` SDK:

| Framework                                                        | Status                                     | Serve it with                                                                                           | Add to a project                                                                          |
| ---------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| [OpenAI Agents SDK](references/getting-started/openai-agents.md) | Supported                                  | `openai_agents(agent)`, from `sectr.adapters.openai_agents`                                             | `uv add "sectr[openai-agents]"`                                                           |
| [LangGraph](references/getting-started/langgraph.md)             | Supported, for ReAct-shaped graphs         | `langgraph(graph)`, from `sectr.adapters.langgraph`                                                     | `uv add "sectr[langgraph]"`                                                               |
| [Pydantic AI](references/getting-started/pydantic-ai.md)         | Supported, for a plain `pydantic_ai.Agent` | `pydantic_ai(agent)`, from `sectr.adapters.pydantic_ai`                                                 | `uv add "sectr[pydantic-ai]"`, plus a model provider such as `"pydantic-ai-slim[openai]"` |
| [No framework](references/sdk/no-framework/index.md)             | Supported                                  | your own handler, `@app.agent_route("/chat")`, with tools from [`sectr.tools`](references/sdk/tools.md) | `uv add sectr`                                                                            |

Each adapter supports most, not all, of its framework: the
[adapter reference](references/sdk/adapters/index.md) lists what each one does and
its limits (subagents, structured output, checkpointers and so on).

## Conceptual overview: how a Sectr app fits together

This section walks through the parts every Sectr project has, whatever its
framework: the `sectr.toml` manifest, serving the agent on a route, tools and
approvals, running locally, and deploying. It uses the support agent that
`sectr init` scaffolds (it looks up orders and issues refunds, with a human
approving each refund) as the example.

**Read it to understand Sectr, not as steps to follow.** Don't scaffold a
project, copy this example, or run these commands unless the user asked
for that. In an existing project, find the same pieces in the user's own
code (`sectr.toml`, the `AgentApp` and its routes, their tools) and work
from those.

### Project setup

Scaffold a runnable project:

```sh
sectr init my-agent
```

`sectr init` asks two things:

- **The framework:** the OpenAI Agents SDK, LangGraph or Pydantic AI. Each is
  the same support agent on the same `/chat` route, with the same file names;
  see [using the OpenAI Agents SDK](references/getting-started/openai-agents.md),
  [using LangGraph](references/getting-started/langgraph.md) and
  [using Pydantic AI](references/getting-started/pydantic-ai.md).
- **A chat frontend:** none, or a CopilotKit chat app (Next.js, needs
  Node 20+) already wired to your agent.

To skip the questions, pass them as flags: `sectr init my-agent --template
openai-agents` (or `langgraph`, `pydantic-ai`), plus `--frontend copilotkit`
for the chat app.

Without a frontend, this generates:

```text
my-agent/
├── sectr.toml     # <-- the Sectr manifest
├── app.py         # <-- your agent, and the route that serves it
├── tools.py       # its tools
├── fake.py        # a mock model, used when no API key is set
├── pyproject.toml
├── .env.example   # copy to .env to set OPENAI_API_KEY
└── .gitignore
```

With the CopilotKit frontend, the Next.js app is at the root and the agent
(the same files as above) is in `agent/`. Run `sectr dev` and `sectr deploy`
from `agent/`; the generated `README.md` covers the rest.

```text
my-agent/
├── agent/         # <-- the agent, as above
├── app/           # Next.js pages, the Copilot Runtime and the token route
├── proxy.ts
├── package.json
├── .env.development  # points the chat at `sectr dev`
└── README.md
```

### The manifest

`sectr.toml` tells Sectr what your project is:

```toml
[app]
name = "my-agent"

[agent]
entry = "app:app"
python = "3.12"
```

- **`[app] name`** names the app; its slug identifies it to `sectr deploy`
  and `sectr secrets`.
- **`[agent] entry`** is where your `AgentApp` lives, as `module:attribute`.

Two optional sections come up later: `[deploy] approval_tools`, to
[gate tools by config](#marking-as-gated-in-deployment-config),
and `[dev] port`, to [pin the local port](references/guides/local-dev.md#ports).

### Serving the agent on a route

`app.py` defines your agent in ordinary framework code, then serves it on a
route of an `AgentApp`, through the adapter for its framework:

```python
from sectr import AgentApp

app = AgentApp(name="my-agent", env=["OPENAI_API_KEY"])
app.add_route("/chat", openai_agents(agent))  # or langgraph(graph), pydantic_ai(agent)
```

- **A route serves one agent.** An app can have several, each on its own path.
- **`env`** lists the secrets the agent needs, by name. Their values come
  from `.env` locally and from `sectr secrets` when deployed (see
  [secrets](references/guides/secrets.md)).
- **The adapter** maps your framework's streamed output and approval requests
  onto Sectr's session events. It's the only Sectr-specific line: the agent
  itself is the code you'd write anyway.

Notice what's *absent*: no session handling, no streaming plumbing, no state
management, no database to persist conversations. Sectr keeps every session
in its journal, so a conversation survives restarts, redeploys and
scale-to-zero.

One rule applies whatever the framework: **your module must import cleanly
with no secrets set.** `sectr dev` and `sectr deploy` import your app to find
its routes before any secret is injected, so build anything that needs a key
(a model client, say) only when the key is present. The scaffolded
`fake.py` does that: with no `OPENAI_API_KEY`, a mock model answers, so the
first run needs no keys at all.

### Run it locally

To use the real model, `cp .env.example .env` and put your `OPENAI_API_KEY`
in `.env`. Without it, the mock model answers. Then start the local dev
server:

```sh
# cd my-agent   (if not already there)
sectr dev       # Opens up a live dashboard for local dev server
```

This opens a live playground, where you can chat with your agent, watch its
actions, and iterate on its behavior:

![Chat Playground in the dashboard](https://docs.sectr.dev/screenshots/chat.png)

### Deploy your agent

To move your agent from your laptop to the cloud, run `sectr deploy` from
your project directory:

```sh
$ sectr deploy
packaging .
uploading to https://api.sectr.dev/deployments
deployment 01a0d532-d771-7563-96bd-56f40cb28b2b: version 1 — building…
✓ deployed: version 1 (01a0d532-d771-7563-96bd-56f40cb28b2b) is active
🚀 view live at: https://app.sectr.dev/apps/01a0d365-aa51-78d3-97f9-a2a02bcc00ce
🔌 connect via AG-UI at: https://api.sectr.dev/ag-ui/my-agent/chat
```

This builds your agent in the cloud, finds its routes, and serves each as a
live HTTP endpoint. The dashboard URL shows the deployed app. The last line
is the route's [AG-UI](https://docs.ag-ui.com) endpoint, what a chat frontend
connects to.

Each `sectr deploy` makes a new version of the app. New sessions run on the
newest version that built successfully; a session already under way keeps
the version it started on, so a deploy never interrupts a conversation. To
roll back, use **Rollback to this deployment** on the app's page in the
dashboard.

Then give the deployed app its key, and chat with it:

```sh
sectr secrets set OPENAI_API_KEY
```

To give the deployed agent a chat UI, see [Connecting your frontend](references/getting-started/connecting-frontend.md): it puts a CopilotKit chat on it, approvals included. (A project scaffolded with `--frontend copilotkit` already has one: see its `README.md`.)

## Defining Gated Tools

Gate a tool when it shouldn't run unattended (issuing refunds, sending
emails, calling paid APIs): the turn suspends at the call, a human approves
or denies it, and the turn resumes with the decision. A tool is gated if
either its code or the deployment's config says so.

### Marking as gated in code

Gate a tool where it's defined. Which way depends on how the agent is
written:

- **OpenAI Agents SDK or Pydantic AI: use the framework's own approval
  flag.** Sectr's adapter turns the framework's approval request into a
  Sectr pause and resumes the framework with the decision, so the tool stays
  ordinary framework code. These adapters read the framework's flag, not
  `sectr.tools`.
- **LangGraph: use one of the options below, not LangGraph's
  `interrupt()`.** `interrupt()` can't be resumed without a checkpointer, so
  it fails on Sectr.
- **No framework: use `sectr.tools`.**

A tool inside a subagent (another agent your agent runs from a tool) can't
be gated: gate the subagent tool itself, and approving it runs the whole
subagent. Each [adapter's page](references/sdk/adapters/index.md) shows how, under
"Subagents".

#### OpenAI Agents SDK

```python
from agents import function_tool

@function_tool(needs_approval=True)
async def refund(order_id: str) -> str: ...
```

#### Pydantic AI

```python
from pydantic_ai import Agent, Tool

agent = Agent(model, tools=[lookup_order, Tool(refund, requires_approval=True)])
```

#### LangGraph

Any of these, and they can be mixed:

- A Sectr tool, converted for a `ToolNode` or `create_agent`:

  ```python
  from sectr.adapters.langgraph import as_langgraph_tool
  from sectr.tools import tool

  @tool(needs_approval=True)
  async def refund(order_id: str) -> str:
      """Refund an order."""
      ...

  tools = [as_langgraph_tool(refund)]
  ```

- Any LangChain tool (a `@tool`, an MCP tool, a community tool), gated as
  it's converted: `as_langgraph_tool(some_tool, needs_approval=True)`.

- On `create_agent`, LangChain's own `HumanInTheLoopMiddleware`. Each
  `interrupt_on` entry must allow both approve and reject (`True` does):

  ```python
  from langchain.agents import create_agent
  from langchain.agents.middleware import HumanInTheLoopMiddleware

  graph = create_agent(
      model,
      tools=tools,
      middleware=[HumanInTheLoopMiddleware(interrupt_on={"refund": True})],
  )
  ```

Sectr pauses all of these itself: see
[how the LangGraph adapter works](references/sdk/adapters/langgraph.md).

#### No framework

A handler written directly against the SDK (`@app.agent_route`) gates its
tools with [`sectr.tools`](references/sdk/tools.md):

```python
from sectr.tools import tool

@tool(needs_approval=True)
async def refund(order_id: str) -> str:
    """Refund an order."""
    ...
```

### Marking as gated in deployment config

You can also gate tools without touching the code: list them under
`approval_tools` in `sectr.toml`.

```toml
[deploy]
approval_tools = ["refund", "delete_account"]
```

These tools pause for approval even if the code doesn't gate them, MCP
tools included. This allows declarative, policy-directed gating: which
tools need approval is configuration, set per deployment rather than in
the agent's code. The list applies in `sectr dev` and when deployed. A change takes
effect the next time you start `sectr dev` or run `sectr deploy`.

Sectr hands the list to your runner as the `SECTR_APPROVAL_POLICY`
environment variable, which the SDK reads. Don't set that variable yourself:
`approval_tools` is the only place to set the policy.

## Reference

Deeper material lives in `references/`, one file per docs page:

| Page | File |
| --- | --- |
| **Connecting your frontend**: Put a CopilotKit chat on your deployed agent — every Sectr route is an AG-UI endpoint, so CopilotKit's React components work as-is. | references/getting-started/connecting-frontend.md |
| **Using LangGraph**: Serve a LangGraph agent on Sectr — streaming, approvals and durable sessions for your graph, with no checkpointer to run. | references/getting-started/langgraph.md |
| **Using the OpenAI Agents SDK**: Serve an OpenAI Agents SDK agent on Sectr — streaming, native tool approvals and durable sessions, with the agent unchanged. | references/getting-started/openai-agents.md |
| **Using Pydantic AI**: Serve a Pydantic AI agent on Sectr — streaming, native tool approvals and durable sessions, resumed from the journal alone. | references/getting-started/pydantic-ai.md |
| **Local dev**: The sectr dev loop — local engine, gateway, dashboard, send/tail. | references/guides/local-dev.md |
| **Secrets**: Write-only secrets for deployed agents — set, list, remove, and how they reach your code. | references/guides/secrets.md |
| **AG-UI endpoint**: Every deployed route is an AG-UI 1.0 endpoint — run a turn with POST, reconnect with GET. Requests, events, approvals, auth and errors. | references/api/ag-ui.md |
| **Approve or deny actions**: The approvals inbox endpoints — decide pending tool calls from your own frontend. | references/api/approvals.md |
| **Authentication**: Platform keys, invocation keys, user tokens, and the session cookie — which principal for which endpoint. | references/api/auth.md |
| **API overview**: The public HTTP API — one event journal, SSE tails, and a handful of verbs. | references/api/index.md |
| **Limits**: Concurrency, capacity and rate limits on the Sectr API, the 429 error codes, and how to retry. | references/api/limits.md |
| **Send a message**: Create a session, add follow-up turns — 202 means scheduled. | references/api/send-message.md |
| **Stream a session**: The SSE journal tail — every frame is a journal event, resumption is Last-Event-ID. | references/api/stream.md |
| **CLI reference**: Every sectr command, generated from the built binary. | references/cli.md |
| **Framework adapters**: sectr.adapters — one adapter per supported framework, each on its own page. | references/sdk/adapters/index.md |
| **LangGraph adapter**: sectr.adapters.langgraph — serve a LangGraph graph: its signatures, behavior, the graphs it supports, how it works, and its limits. | references/sdk/adapters/langgraph.md |
| **OpenAI Agents SDK adapter**: sectr.adapters.openai_agents — serve an OpenAI Agents SDK agent: its signature, behavior, handoffs, subagents and limits. | references/sdk/adapters/openai-agents.md |
| **Pydantic AI adapter**: sectr.adapters.pydantic_ai — serve a Pydantic AI agent: its signature, behavior, output types and limits. | references/sdk/adapters/pydantic-ai.md |
| **AgentApp**: The route registry and manifest source — the one sectr object your project exposes. | references/sdk/app.md |
| **SessionContext**: The argument of a handler without a framework: the conversation, why the turn started, and running tools. | references/sdk/no-framework/context.md |
| **No framework**: Write the agent loop yourself: a handler that reads the conversation, calls the model, and yields the events Sectr journals. | references/sdk/no-framework/index.md |
| **Tools**: sectr.tools: tools from plain functions, for agents without a framework and for LangGraph. | references/sdk/tools.md |
