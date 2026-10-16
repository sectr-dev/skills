---
name: sectr
description: >-
  Build, run locally, and deploy AI agents on Sectr, a PaaS for hosting AI
  agents. Covers the sectr CLI (init, dev, send, tail, deploy, secrets, auth);
  the sectr Python SDK (AgentApp, SessionContext, events, tools, and the
  adapters for the OpenAI Agents SDK, LangGraph and Pydantic AI);
  human-in-the-loop tool approvals; connecting a frontend over AG-UI
  (CopilotKit); and the public HTTP API (sessions, SSE journal streaming,
  approvals, AG-UI endpoints, limits). Use when writing or debugging a Sectr
  agent, integrating the sectr SDK, deploying to Sectr Cloud, putting a chat UI
  on a deployed agent, driving deployed agents over the HTTP API, or working
  with Sectr concepts like journals, turns, and approvals.
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

| Framework | Serve it with | Add to a project |
| --- | --- | --- |
| OpenAI Agents SDK | `app.add_route("/chat", openai_agents(agent))`, from `sectr.adapters.openai_agents` | `uv add "sectr[openai-agents]"` |
| LangGraph | `app.add_route("/chat", langgraph(graph))`, from `sectr.adapters.langgraph` | `uv add "sectr[langgraph]"` |
| Pydantic AI | `app.add_route("/chat", pydantic_ai(agent))`, from `sectr.adapters.pydantic_ai` | `uv add "sectr[pydantic-ai]"` plus a model extra, e.g. `"pydantic-ai-slim[openai]"` |
| No framework | a handler of your own, `@app.agent_route("/chat")`, with tools from `sectr.tools` | `uv add sectr` |

Each adapter supports most, not all, of its framework: check the
framework's section in `references/sdk/adapters.md` before relying on a
feature (subagents, structured output, checkpointers and so on).

## Conceptual overview: how a Sectr app fits together

This section walks through the example project that `sectr init` creates
(a support agent that looks up orders and issues refunds behind an
approval) to show how the pieces of a Sectr app relate: the `sectr.toml`
manifest, the agent code and the route it's served on, tools and approval
gating, the local dev loop, and deploying.

**Read it to understand Sectr, not as steps to follow.** Don't scaffold a
project, copy this example, or run these commands unless the user asked
for that. In an existing project, find the same pieces in the user's own
code (`sectr.toml`, the `AgentApp` and its routes, their tools) and work
from those.

### Project Setup

Scaffold a runnable project:

```sh
sectr init my-agent
```

`sectr init` asks two things:

- **The framework:** OpenAI Agents SDK or LangGraph. This page walks through
  the OpenAI Agents SDK version; [Using LangGraph](references/getting-started/langgraph.md)
  covers the other. Both are the same agent on the same `/chat` route, with
  the same file names.
- **A chat frontend:** none, or a CopilotKit chat app (Next.js, needs Node 20+)
  already wired to your agent.

To skip the questions, pass them as flags: `sectr init my-agent --template
openai-agents` (or `langgraph`), plus `--frontend copilotkit` for the chat app.

Without a frontend, this generates the following project layout:

```text
my-agent/
├── sectr.toml     # <-- The Sectr manifest
├── app.py         # <-- OpenAI Agents SDK code + a Sectr endpoint
├── tools.py       # (tool definitions)
├── fake.py        # (mock model)
├── pyproject.toml
├── .env.example   # copy to .env to setup an OPENAI_API_KEY
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

### The agent

Using Sectr is as easy as writing the agent code you are already familiar with, and then exposing it on a Sectr route. No special frameworks or code patterns are required.

#### Agent defintions and routes

Let's take quick look at `app.py`, where we define the agent and its Sectr route:

```py
from agents import Agent
from sectr import AgentApp
from sectr.adapters.openai_agents import openai_agents

from fake import setup_model
from tools import lookup_order, refund

model = setup_model()

agent = Agent(
    name="support",
    instructions=(
        "You are a customer-support agent for a small shop. "
        "Look up order status with the lookup_order tool, "
        "and issue refunds with the refund tool."
    ),
    tools=[lookup_order, refund],
    model=model,
)

app = AgentApp(name="my-agent", env=["OPENAI_API_KEY"])
app.add_route("/chat", openai_agents(agent, model=model))
```

First, you define your `Agent` using standard OpenAI Agents SDK code. You don't have to do anything different here: this is the agent code you already write, no special SDK or domain specific language needed. Sectr is designed to be agnostic to the agent framework, with more coming soon.

Second, you simply expose your agent as a route on your Sectr app. Thats all.

Notice what's *absent*: no session handling, no streaming
plumbing, no state management, no Postgres queries to persist conversations.
Sectr provides that, so you can focus on the AI.

#### Tool Defintions

The tools are vanilla OpenAI Agents SDK tools, defined in `tools.py`:

```python
from agents import function_tool

@function_tool
async def lookup_order(order_id: str) -> dict[str, str]:
    ...

@function_tool(needs_approval=True)
async def refund(order_id: str) -> str:
    ...
```

Note that the `refund` tool is marked with `needs_approval=True`. Sectr automatically handles human-in-the-loop (HITL), suspends your agent's execution, and elevates the request to allow for approval within the Sectr dashboard. It doesn't matter how long an approval takes: even weeks later, Sectr will resume execution of your agent after an approval.

### Run it locally

If you want to use a live OpenAI model, just `cp .env.example .env`, and then put in your `OPENAI_API_KEY` in the `.env` file. If you don't, then the example uses a mocked LLM.

Once you have your OpenAI API key setup, start the local dev server:

```sh
# cd my-agent   (if not already there)
sectr dev       # Opens up a live dashboard for local dev server
```

This opens up a live playground, where you can immediatley chat with your agent, observe its actions, test out, and iterate on its behavior:

![Chat Playground in the dashboard](https://docs.sectr.dev/screenshots/chat.png)

### Deploy your agent

To move your agent from running on your laptop to being cloud-deployed, all you need to do is run `sectr deploy` from your project directory:

```sh
$ sectr deploy
packaging .
uploading to https://api.sectr.dev/deployments
deployment 01a0d532-d771-7563-96bd-56f40cb28b2b: version 1 — building…
✓ deployed: version 1 (01a0d532-d771-7563-96bd-56f40cb28b2b) is active
🚀 view live at: https://app.sectr.dev/apps/01a0d365-aa51-78d3-97f9-a2a02bcc00ce
🔌 connect via AG-UI at: https://api.sectr.dev/ag-ui/my-agent/chat
```

This performs a cloud build of your agent, analyze its routes, and deploys it as a live HTTP endpoint on Sectr. You can jump straight to view it in the Sectr production dashboard at that URL. The last line is your route's [AG-UI](https://docs.ag-ui.com) endpoint — what a chat frontend connects to.

You can then setup set your `OPENAI_API_KEY` as an encrypted secret for your app in the Sectr Dashboard at Settings → Secrets, and chat with your cloud-deployed support agent.

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
subagent. See [subagents](references/sdk/adapters.md#subagents).

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

Sectr pauses all of these itself — see
[frameworks without native approval](https://docs.sectr.dev/guides/approvals/#frameworks-without-native-approval-langgraph).

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

- `sectr dev` refuses to run with `SECTR_APPROVAL_POLICY` in its environment
  or in `.env`.
- `sectr secrets set SECTR_APPROVAL_POLICY` is refused.
- `sectr deploy` is refused while the app still has a secret by that name.
  Remove it with `sectr secrets rm SECTR_APPROVAL_POLICY`.

## Reference

Deeper material lives in `references/`, one file per docs page:

| Page | File |
| --- | --- |
| **Connecting your frontend**: Put a CopilotKit chat on your deployed agent — every Sectr route is an AG-UI endpoint, so CopilotKit's React components work as-is. | references/getting-started/connecting-frontend.md |
| **Using LangGraph**: Serve a LangGraph agent on Sectr — streaming, approvals and durable sessions for your graph, with no checkpointer to run. | references/getting-started/langgraph.md |
| **Using Pydantic AI**: Serve a Pydantic AI agent on Sectr — streaming, native tool approvals and durable sessions, resumed from the journal alone. | references/getting-started/pydantic-ai.md |
| **Compaction**: Keep long-running conversations inside the context window — TRANSCRIPT_COMPACTED and the positional fold. | references/guides/compaction.md |
| **Framework adapters**: OpenAI Agents SDK, LangGraph and Pydantic AI on Sectr — what an adapter does, and what each supports. | references/guides/framework-adapters.md |
| **Local dev**: The sectr dev loop — local engine, gateway, dashboard, send/tail. | references/guides/local-dev.md |
| **Secrets**: Write-only secrets for deployed agents — set, list, remove, and how they reach your code. | references/guides/secrets.md |
| **AG-UI endpoint**: Every deployed route is an AG-UI 1.0 endpoint — run a turn with POST, reconnect with GET. Requests, events, approvals, auth and errors. | references/api/ag-ui.md |
| **Approve or deny actions**: The approvals inbox endpoints — decide pending tool calls from your own frontend. | references/api/approvals.md |
| **Authentication**: Platform keys, invocation keys, user tokens, and the session cookie — which principal for which endpoint. | references/api/auth.md |
| **Deploy lifecycle**: Upload, build, activate, roll back — the 202 + polling contract. | references/api/deploy-lifecycle.md |
| **API overview**: The public HTTP API — one event journal, SSE tails, and a handful of verbs. | references/api/index.md |
| **Limits**: Concurrency, capacity and rate limits on the Sectr API, the 429 error codes, and how to retry. | references/api/limits.md |
| **Send a message**: Create a session, add follow-up turns — 202 means scheduled. | references/api/send-message.md |
| **Stream a session**: The SSE journal tail — every frame is a journal event, resumption is Last-Event-ID. | references/api/stream.md |
| **CLI reference**: Every sectr command, generated from the built binary. | references/cli.md |
| **Framework adapters**: sectr.adapters — run framework agents on the platform contract. | references/sdk/adapters.md |
| **AgentApp**: The route registry and manifest source — the one sectr object your project exposes. | references/sdk/app.md |
| **SessionContext**: The one argument every handler receives — transcript, reason, tools, messages. | references/sdk/context.md |
| **Events**: The runner-emittable event types — what handlers yield. | references/sdk/events.md |
| **SDK reference**: The sectr Python SDK — modules, mental model, install. | references/sdk/index.md |
| **sectr.server**: The invoke-contract adapter — how your AgentApp becomes POST /invocations. | references/sdk/server.md |
| **Tools**: Framework-free tool registration and approval gating — sectr.tools. | references/sdk/tools.md |
