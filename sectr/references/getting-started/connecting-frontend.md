# Connecting your frontend

Every agent you deploy is also an [AG-UI](https://docs.ag-ui.com) endpoint.
That means [CopilotKit](https://docs.copilotkit.ai)'s React chat components
(and any other AG-UI client) talk to your agent with no SDK of ours and no
event parsing on your side: streaming text, tool calls and their results,
and approvals as native interrupts.

`sectr dev` and `sectr deploy` both print the endpoint for each agent:

```text
🔌 connect via AG-UI at: https://api.sectr.dev/ag-ui/my-agent/chat
```

> **Starting fresh?**: `sectr init my-agent --frontend copilotkit` scaffolds a Next.js app with a
> CopilotKit chat already wired to a new agent.
> [Your first agent](../../SKILL.md#conceptual-overview-how-a-sectr-project-fits-together) walks through running it
> locally and against the deployed agent.

This page covers adding Sectr to a CopilotKit app, and how the pieces fit
together.

## How it fits together

The code that touches Sectr is one package plus three small files.

**1. Install the package** — [`@sectr/copilotkit`](https://www.npmjs.com/package/@sectr/copilotkit)
provides `SectrAgent` and `SectrRunner`, and works with the CopilotKit packages
your app already has:

```sh
npm install @sectr/copilotkit
```

**2. The Copilot Runtime route** — CopilotKit's
[Copilot Runtime](https://docs.copilotkit.ai/backend/copilot-runtime), with a
`SectrAgent` pointing at your agent instead of CopilotKit's built-in one, and
`SectrRunner` in place of CopilotKit's default runner:

```ts
// app/api/copilotkit/[[...slug]]/route.ts
import {
    CopilotRuntime,
    createCopilotRuntimeHandler,
} from "@copilotkit/runtime/v2";
import { SectrAgent, SectrRunner } from "@sectr/copilotkit";

const sectrAgent = new SectrAgent({ url: process.env.SECTR_AGUI_URL! });
const agents = { default: sectrAgent };
const runtime = new CopilotRuntime({
    agents,
    runner: new SectrRunner({ agents }),
});
const handler = createCopilotRuntimeHandler({
    runtime,
    basePath: "/api/copilotkit",
});

export const GET = handler;
export const POST = handler;
```

The runner decides where conversations live. CopilotKit's default keeps them
in your server's memory; `SectrRunner` keeps them on Sectr, so your server is
stateless: deploy it anywhere, on serverless or several instances.

To serve several agents from one CopilotKit app, give each its own `agentId`
in the Runtime's agents map and set the `agentId` property on your `CopilotSidebar`. The
[`sectr-copilotkit-starter`](https://github.com/sectr-dev/sectr-copilotkit-starter)
does this with the OpenAI Agents SDK and LangGraph versions of its agent.

**3. A token-mint route** — It translates the notion of *your* users into Sectr user tokens.
It mints a user token (`sks_…`) that
can only see that user's conversations:

```ts
// app/api/agent-token/route.ts
const res = await fetch("https://api.sectr.dev/tokens", {
    method: "POST",
    headers: {
        Authorization: `Bearer ${process.env.SECTR_INVOCATION_KEY}`,
        "Content-Type": "application/json",
    },
    body: JSON.stringify({ app: "my-agent", external_user_id: user.id }),
});
return Response.json({ token: (await res.json()).token });
```

**4. The provider** — fetch the token once per page load and hand it to
CopilotKit as a header. The Runtime forwards `Authorization` to Sectr by
default:

```tsx
<CopilotKitProvider
    runtimeUrl="/api/copilotkit"
    headers={{ Authorization: `Bearer ${token}` }}
>
    <CopilotSidebar threadId={threadId} />
</CopilotKitProvider>
```

Sectr never authenticates your end users: your app's login stays the source
of truth, and `external_user_id` is whatever id it uses. Tokens last 24 hours
and the scaffold mints one on every page load, so the only tab that outlives
its token is one left open for a day; its runs then fail with `401` until
it reloads (or your app re-mints). Under `sectr dev` there
is no auth: the scaffold skips minting when no invocation key is set. See
[authentication](../api/auth.md#user-token-sks_) for the token rules.

The app needs two server-side variables wherever it runs:

- `SECTR_AGUI_URL`: the AG-UI URL `sectr dev` or `sectr deploy` printed.
- `SECTR_INVOCATION_KEY`: your project's invocation key. Create one from
  **Connect your frontend** on the project's page in the dashboard, or under
  **Settings → API keys**. Leave it unset under `sectr dev`.

> **Never put the invocation key in the browser**: It can read every user's conversations and create sessions without limit.
> Mint user tokens instead. (A single-user or internal tool may use the
> invocation key as its client credential — see
> [authentication](../api/auth.md).)

### Conversations are sessions

AG-UI's `threadId` **is** the Sectr session id, and it must be a UUID
(CopilotKit's are). A run on a new `threadId` starts a session owned by the
token's user; later runs on it continue that conversation. The scaffold keeps
the thread id in the URL (`?thread=…`), so a reload lands back in the same
conversation.

The session's journal is the conversation's history. Each run on an existing
thread starts with a complete `MESSAGES_SNAPSHOT` — including tool calls and
their results — so the chat always matches what the agent actually saw.

### Approvals are interrupts

A tool that needs approval ends the run with an AG-UI interrupt
(`RUN_FINISHED` with `outcome.type: "interrupt"`, `reason: "tool_approval"`,
the tool name and arguments in `metadata`). CopilotKit's `useInterrupt`
renders it, and resolving it starts the resume run:

```tsx
useInterrupt({
    enabled: ({ value }) => value?.reason === "tool_approval",
    render: ({ interrupt, resolve }) => (
        <div>
            Run <code>{interrupt?.metadata?.toolName}</code>?
            <button onClick={() => resolve(true)}>Approve</button>
            <button onClick={() => resolve(false)}>Deny</button>
        </div>
    ),
});
```

`resolve(true)` approves, `resolve(false)` or `cancel()` denies. The
decision is recorded with the token's user as the approver, and the agent
continues the same turn — whether that is seconds or weeks later.

### Refreshing mid-answer

The turn runs on Sectr, not in the browser: closing or reloading the page
never stops it. A reloaded page reattaches from Sectr: the conversation so
far, the answer still streaming, or the approval still waiting for a
decision. Nothing about it lives in your server, so this holds on any host,
across restarts and across devices.

### When Sectr is busy

During the beta, a run can be refused before it starts with
`429 Too Many Requests`: your project already has its maximum number of turns
running, Sectr is at capacity, or a user has started too many conversations
recently. Nothing is recorded for a refused run, so wait the `Retry-After`
header's seconds and send the same run again. See
[limits](../api/limits.md) for the codes and defaults; `sectr dev`
never applies them.

## Two ways to connect

**Through the Copilot Runtime (recommended)** — everything above. Browser
traffic stays same-origin; the Runtime is CopilotKit's layer, running in your
Next.js server.

**Direct from the browser** — skip the Runtime and give CopilotKit the agent
itself. Handy for prototypes. A reloaded page still reattaches: the
`SectrAgent` reconnects straight to Sectr.

```tsx
import { SectrAgent } from "@sectr/copilotkit";

const agent = new SectrAgent({
    url: "https://api.sectr.dev/ag-ui/my-agent/chat",
    headers: { Authorization: `Bearer ${token}` },
});

<CopilotKitProvider selfManagedAgents={{ default: agent }}>
    <CopilotChat threadId={threadId} />
</CopilotKitProvider>;
```

Use `SectrAgent`, not `@ag-ui/client`'s plain `HttpAgent`: `HttpAgent`
can't reconnect, so after a reload the chat starts empty, and an approval
that was pending can no longer be answered (every new message gets `409`).

The AG-UI endpoint accepts cross-origin requests from any origin (auth is the
bearer token, never cookies), so no proxy is needed. The
[AG-UI endpoint reference](../api/ag-ui.md) has the request, event and
error details.

## What's not supported yet

- **Shared state** (`STATE_SNAPSHOT` / `STATE_DELTA`): not emitted. Keep state
  in the conversation (e.g. a todo-list tool).
- **Frontend tools**: tools your frontend registers are accepted but the
  agent can't call them yet.
- **Token usage** on `RUN_FINISHED`: not reported yet.
- **`forwardedProps` / `context`** are recorded with each run but not yet
  exposed to your agent code.
- **Importing history**: a new `threadId` must start empty — a run that
  creates a thread with earlier messages in it is rejected (`400`).

## The power path: Sectr's own API

AG-UI is a view over the session journal. For anything it doesn't cover —
custom clients, several devices following one conversation, listing a user's
conversation history — use the sessions API directly, with the same user
token:

1. **Create a session** — `POST /sessions` with `{app, route, input}`. See
   [send a message](../api/send-message.md).
2. **Stream it** — `GET /sessions/{id}/events` is a server-sent-events tail
   of the journal; every event has a resumable `id`, and reconnecting with
   `Last-Event-ID` fills any gap exactly once. `EventSource` can't send
   headers, so pass the token as `?token=sks_…`. See
   [stream a session](../api/stream.md).
3. **Add turns** — `POST /sessions/{id}/input`. A `409` means the previous
   turn is still running or awaiting approval.
4. **Decide approvals** — `POST /sessions/{id}/approval`. See
   [approve or deny actions](../api/approvals.md).
5. **List history** — `GET /sessions` with a user token returns only that
   user's sessions.

Steps 1, 3 and 4 start work, so during the beta they can also answer `429`
— see [When Sectr is busy](#when-sectr-is-busy) and
[limits](../api/limits.md).
