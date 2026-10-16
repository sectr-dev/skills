---
name: sectr
description: >-
  Build, run locally, and deploy AI agents on Sectr, a PaaS for hosting AI
  agents. Covers the sectr CLI (init, dev, send, tail, deploy, secrets, auth);
  the sectr Python SDK (AgentApp, SessionContext, events, tools, and the
  adapters for the OpenAI Agents SDK, LangGraph, Pydantic AI and the Claude
  Agent SDK); human-in-the-loop tool approvals; connecting a frontend over AG-UI
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
Pydantic AI, the Claude Agent SDK, or plain Python: the `sectr` SDK's
adapters run each framework's agent on the platform. Every deployed route
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
     framework instead, e.g. `uv add "sectr[langgraph]"` (see
     `references/getting-started/` and `references/sdk/adapters.md`).
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

The sections below explain each step in more detail.

## Getting started

## Setting Up Sectr for Development

Developing agents requires the **`sectr`** CLI to be installed on your dev machine, to be able to run agents locally and deploy them to Sectr Cloud. Your agent itself only needs to depend on and import the **`sectr` Python SDK**.

### Installing the `sectr` CLI

One command, no runtime dependencies:

```sh
curl -fsSL https://sectr.dev/install.sh | sh
```

Verify it with `sectr --version`.

### Authenticating the `sectr` CLI

All cloud commands (`sectr deploy`, `sectr secrets`, etc.) authenticate with a **platform API key** (`skp_…`).
To get one, sign in at [app.sectr.dev](https://app.sectr.dev) with GitHub or Google, then click **Create CLI key** in the Getting-started checklist (or in Settings → API keys). The key is shown only once, so copy it right away.
Authenticate the CLI using `sectr auth login` so all future commands need no flags, entering your platform key when prompted:

```sh
sectr auth login
```

### Giving your coding agent the Sectr skill

If you build with a coding agent (Claude Code, Codex, Cursor, and others), install the `sectr` agent skill: these docs, packaged so your agent can look up the CLI, SDK and API while it works on your Sectr project.

```sh
npx skills add sectr-dev/skills
```

That installs it into the current project. Add `-g` to install it for your user instead, and `--agent <name>` to pick an agent (repeat it for several, e.g. `--agent claude-code --agent codex`). `npx skills update` picks up later docs changes.

## Your first agent

Now that `sectr` CLI is installed and authenticated, let's get your first agent up and running: a customer support agent that can look up orders and issue refunds with admin approval.

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

### Next: put it in your app

Your agent is live — now give it a frontend. [Connecting your frontend](references/getting-started/connecting-frontend.md) puts a CopilotKit chat on it in a few minutes, approvals included. (If you scaffolded with `--frontend copilotkit`, you already have one: see its `README.md`.)

## Approvals (human in the loop)

Some actions shouldn't run unattended: issuing refunds, sending emails,
calling paid APIs. Sectr makes those tools **gated**: the turn suspends at
the call and a human decides, then the turn resumes with the decision.

### Mark a tool as gated

Either mechanism (they union):

```python
@tool(needs_approval=True)          # framework-free (sectr.tools)
async def refund(order_id: str) -> str: ...
```

```python
@function_tool(needs_approval=True)  # openai-agents native HITL
async def refund(order_id: str) -> str: ...
```

```python
Tool(refund, requires_approval=True)  # pydantic-ai native deferred tools
```

If one model response calls several gated tools, they are approved one at a
time: you decide the first, and the turn pauses again on the next.

On LangGraph, mark the tool with `sectr.tools.tool` and convert it with
`as_langgraph_tool`, or gate any tool, MCP tools included, with LangChain's
`HumanInTheLoopMiddleware` on `create_agent` or with
`as_langgraph_tool(tool, needs_approval=True)`: Sectr pauses those calls
itself — see [below](#frameworks-without-native-approval-langgraph).

A tool inside a subagent (another agent your agent runs from a tool) can't
be gated: gate the subagent tool itself, and approving it runs the whole
subagent. See [subagents](references/sdk/adapters.md#subagents).

#### Gate by deployment config

You can also gate tools without touching the code: list them under
`approval_tools` in `sectr.toml`.

```toml
[deploy]
approval_tools = ["refund", "delete_account"]
```

These tools pause for approval even if the code doesn't gate them, MCP
tools included. This is the safety net for a tool its author forgot to
mark. The list applies in `sectr dev` and when deployed. A change takes
effect the next time you start `sectr dev` or run `sectr deploy`.

Sectr hands the list to your runner as the `SECTR_APPROVAL_POLICY`
environment variable, which the SDK reads. Don't set that variable yourself:
`approval_tools` is the only place to set the policy.

- `sectr dev` refuses to run with `SECTR_APPROVAL_POLICY` in its environment
  or in `.env`.
- `sectr secrets set SECTR_APPROVAL_POLICY` is refused.
- `sectr deploy` is refused while the app still has a secret by that name.
  Remove it with `sectr secrets rm SECTR_APPROVAL_POLICY`.

### What a suspension looks like

The gated call raises `ApprovalRequired`; the adapter emits
`ToolApprovalRequired` as the **final** event. No turn-end sentinel is
journaled — the turn is suspended, not over. The session reports
`awaiting_approval`, the dashboard shows a banner, and the approval lands in
the cross-app approvals inbox.

### Deciding

**From the CLI** (`sectr dev` / cloud with `--cloud`):

```sh
sectr send --approve   # resume the turn with "approved"
sectr send --deny      # resume with the rejection
```

**From the dashboard** — the session banner or the approvals inbox, one
click each.

**From your own UI** — the public API endpoint the dashboard itself uses
(a user token works too, for sessions its user owns):

```sh
curl -X POST https://api.sectr.dev/sessions/$SESSION_ID/approval \
  -H "Authorization: Bearer $SECTR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"approved": true}'
```

Full flow details, the 409 cases, and TypeScript snippets:
[approve or deny from your UI](references/api/approvals.md).

### What happens on resume

The decision is journaled (`TOOL_APPROVAL_GRANTED`/`DENIED`), and the
platform spawns a fresh runner whose invocation reason is the decision.
Your handler sees it as `ctx.reason.kind == "approval_decision"` with
`ctx.decision` and `ctx.pending_tool_call` in hand — execute the pending
tool (or record the denial) and continue:

```python
if ctx.reason["kind"] == "approval_decision":
    pending = ctx.pending_tool_call
    result = (
        await ctx.execute(pending["name"], call_id=pending["tool_call_id"], **pending["arguments"])
        if ctx.decision["approved"]
        else {"error": "denied"}
    )
    yield ToolResult(tool_call_id=pending["tool_call_id"], result=result)
```

(Real code: `examples/raw/main.py` — the framework-free resume path. The
openai-agents adapter does the equivalent with the framework's own items.)

The resumed turn runs exactly like any other: streamed, journaled, ending at
the sentinel. One decision per resume: two gated calls in one turn become
sequential approvals, decided one per turn.

### Frameworks without native approval (LangGraph)

When the framework has no per-tool human-in-the-loop that works without its
own persistence, the pause is **exception-based**: the gated tool wrapper
raises `ApprovalRequired`, a `BaseException` so frameworks' error handling
can't swallow it, and the adapter turns it into the final
`ToolApprovalRequired` frame. On resume the adapter rebuilds the paused
state from the journal and lets the framework run the approved call itself
(or answers it with the rejection text), journals the result, and continues —
the approved action runs exactly once, and nothing of the framework's state
survives between the pause and the resume except the journal. If one model message holds several
gated calls, they are approved one per turn. Details:
[LangGraph adapter](references/sdk/adapters.md#langgraph).

> **Why a fresh invocation**: The runner is stateless — a pure function of `(transcript, reason)`. The
> journal is the state; resuming after approval is just another invocation
> whose reason happens to be a decision. That's also why agent code can be
> edited between the pause and the resume without breaking anything.

## Reference

Deeper material lives in `references/`, one file per docs page:

| Page | File |
| --- | --- |
| **Connecting your frontend**: Put a CopilotKit chat on your deployed agent — every Sectr route is an AG-UI endpoint, so CopilotKit's React components work as-is. | references/getting-started/connecting-frontend.md |
| **Using LangGraph**: Serve a LangGraph agent on Sectr — streaming, approvals and durable sessions for your graph, with no checkpointer to run. | references/getting-started/langgraph.md |
| **Using Pydantic AI**: Serve a Pydantic AI agent on Sectr — streaming, native tool approvals and durable sessions, resumed from the journal alone. | references/getting-started/pydantic-ai.md |
| **Compaction**: Keep long-running conversations inside the context window — TRANSCRIPT_COMPACTED and the positional fold. | references/guides/compaction.md |
| **Framework adapters**: OpenAI Agents SDK, LangGraph, Pydantic AI, and Claude Agent SDK on Sectr. | references/guides/framework-adapters.md |
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
