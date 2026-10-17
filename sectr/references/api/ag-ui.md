# AG-UI endpoint

Every route you deploy is also an [AG-UI](https://docs.ag-ui.com) 1.0
endpoint, so any AG-UI client (CopilotKit, `@ag-ui/client`'s `HttpAgent`, your
own) can drive your agent. This page is the wire reference; for wiring up a
chat UI, start with [connecting your frontend](../getting-started/connecting-frontend.md).

```text
https://api.sectr.dev/ag-ui/{app}/{route}
```

`{app}` is the app's slug or id; `{route}` is the route path from
`sectr.toml` without its leading slash (`/chat` → `/ag-ui/support-agent/chat`).
`sectr deploy` and `sectr dev` print the full URL for each route. The
endpoint is another view of the [session journal](index.md): the same
conversations the sessions API serves.

| Method           | What                                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------------------ |
| `POST`           | **Run**: start one turn and stream it.                                                                       |
| `GET ?threadId=` | **Reconnect**: re-stream the thread's latest run, read-only. A Sectr extension (AG-UI 1.0 has no reconnect). |

Full schemas: [`POST /ag-ui/{app}/{route}`](openapi.json) and
[`GET /ag-ui/{app}/{route}`](openapi.json).

## Run — `POST`

The body is an AG-UI `RunAgentInput`:

```sh
curl -N https://api.sectr.dev/ag-ui/support-agent/chat \
  -H "Authorization: Bearer $SECTR_USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "threadId": "0192f3a4-b5c6-7e2f-8a9b-1c2d3e4f5a6b",
    "runId": "run-1",
    "messages": [{ "id": "msg-user-1", "role": "user", "content": "hello" }]
  }'
```

- **One run is one turn.** `threadId` **is** the Sectr session id and must be
  a UUID. A new `threadId` creates the session, owned by the user token's end
  user. Later runs on it continue the conversation.
- **The trailing message is the input.** It must be `role: "user"`; its
  `content` (a string or content parts) is what the agent receives. Empty
  content (only whitespace, or no parts) is a `400`.
- **The journal is the history.** On an existing thread, earlier `messages`
  are ignored, and the stream opens with a `MESSAGES_SNAPSHOT` of the real
  conversation, tool calls and results included. A new thread must start
  empty: earlier messages (other than `system` / `developer`) are a `400`.
- `runId` is yours: it's echoed in `RUN_STARTED` and recorded with the turn.
  `protocolVersion`, if sent, must be `1.x`.
- `forwardedProps` and `context` are recorded with the turn but not yet
  exposed to agent code. `tools` (frontend tools) and `state` are accepted
  and ignored.

### The stream

`text/event-stream`, one JSON event per `data:` frame, keep-alive comments
every 15 seconds:

```text
data: {"type":"RUN_STARTED","threadId":"0192f3a4-…","runId":"run-1","protocolVersion":"1.0","input":{…},"timestamp":1768478401123,"metadata":{"seq":1}}

data: {"type":"TEXT_MESSAGE_START","messageId":"a1b2c3d4e5f60718","role":"assistant","timestamp":1768478402123,"metadata":{"seq":2}}

data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"a1b2c3d4e5f60718","delta":"Hi ","timestamp":1768478403123,"metadata":{"seq":3}}

data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"a1b2c3d4e5f60718","delta":"there!","timestamp":1768478404123,"metadata":{"seq":4}}

data: {"type":"TEXT_MESSAGE_END","messageId":"a1b2c3d4e5f60718","timestamp":1768478405123,"metadata":{"seq":5}}

data: {"type":"RUN_FINISHED","threadId":"0192f3a4-…","runId":"run-1","outcome":{"type":"success"},"timestamp":1768478406123,"metadata":{"seq":6}}
```

Every stream ends in exactly one `RUN_FINISHED` or `RUN_ERROR`, with every
message and tool call it opened closed first. Events you'll see:

| Events                                                    | When                                                                                        |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `RUN_STARTED`, then `MESSAGES_SNAPSHOT`                   | every run; the snapshot only on an existing thread                                          |
| `TEXT_MESSAGE_START` / `_CONTENT` / `_END`                | assistant text                                                                              |
| `REASONING_START`, `REASONING_MESSAGE_*`, `REASONING_END` | model reasoning, under the id `<messageId>:reasoning`                                       |
| `TOOL_CALL_START` / `_ARGS` / `_END`, `TOOL_CALL_RESULT`  | tool calls (arguments arrive in one chunk) and their results                                |
| `RUN_FINISHED`                                            | the turn finished (`outcome.type: "success"`) or is waiting for an approval (`"interrupt"`) |
| `RUN_ERROR`                                               | the turn failed; `message` says why                                                         |

Each event carries `timestamp` (ms) and `metadata.seq`, the journal sequence
number it came from. Not emitted yet: `STATE_SNAPSHOT` / `STATE_DELTA`,
`*_CHUNK`, and token usage on `RUN_FINISHED`.

## Approvals — interrupts and `resume`

A tool that [needs approval](approvals.md) ends the run with an
interrupt. The tool's name and arguments are in `metadata`:

```json
{
  "type": "RUN_FINISHED",
  "outcome": {
    "type": "interrupt",
    "interrupts": [{
      "id": "0192f3a4-0103-7000-8000-000000000005",
      "reason": "tool_approval",
      "message": "Refunds over $25 need approval",
      "toolCallId": "call_refund",
      "responseSchema": { "type": "boolean" },
      "metadata": { "toolName": "issue_refund", "arguments": { "order_id": "A-1234", "amount": 42 } }
    }]
  }
}
```

Answer it with a new run on the same thread carrying `resume`:

```json
{
  "threadId": "0192f3a4-b5c6-7e2f-8a9b-1c2d3e4f5a6b",
  "runId": "run-2",
  "messages": [],
  "resume": [{ "interruptId": "0192f3a4-0103-7000-8000-000000000005", "status": "resolved", "payload": true }]
}
```

`payload: true` approves; `payload: false` or `status: "cancelled"` denies.
The decision is recorded with the token's user as the approver, and the
resume run streams the rest of the turn. Approvals don't expire. A `resume`
that doesn't match the pending interrupt gets a stream of `RUN_STARTED` +
`RUN_ERROR` (`code: "invalid_run"`), not an HTTP error.

## Reconnect — `GET ?threadId=`

```sh
curl -N "https://api.sectr.dev/ag-ui/support-agent/chat?threadId=0192f3a4-b5c6-7e2f-8a9b-1c2d3e4f5a6b" \
  -H "Authorization: Bearer $SECTR_USER_TOKEN"
```

Streams the thread's latest run the way a run would: `RUN_STARTED` (with the
original `runId`), a `MESSAGES_SNAPSHOT` of the conversation before it, then
the run's events — replayed, then live. So a reloaded page gets back:

| Thread                           | Stream                                                             |
| -------------------------------- | ------------------------------------------------------------------ |
| doesn't exist yet, or has no run | empty `200` (nothing is created)                                   |
| last turn finished               | replay ending in `RUN_FINISHED`                                    |
| mid-turn                         | replay, then the live answer to its end                            |
| waiting for an approval          | replay ending in the same interrupt; answer it with a `resume` run |
| last turn failed                 | replay ending in its `RUN_ERROR`                                   |

It starts nothing and writes nothing. It's AG-UI's `connect()`:
[`@sectr/copilotkit`](https://www.npmjs.com/package/@sectr/copilotkit)'s
`SectrAgent` implements `connect()` with it, which reattaches CopilotKit chats
both through the Copilot Runtime (via `SectrRunner`) and direct from the
browser. `@ag-ui/client`'s plain `HttpAgent` doesn't call it.

## Auth and CORS

`Authorization: Bearer` with a [user token](auth.md#user-token-sks_)
(`sks_…`, for browsers: it only sees its own user's threads) or an invocation
key (`ski_…`, backend only). Platform keys are refused (`403`). There is no
`?token=` fallback.

Any origin may call the endpoint (auth is the bearer token, never cookies):
preflights are answered for `GET` and `POST` with the headers
`authorization`, `content-type` and `accept`.

## Errors

Requests refused before the stream starts get a JSON body
`{"error": "…"}`:

| Status | Run (`POST`)                                                                                                                      | Reconnect (`GET`)                   |
| ------ | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `400`  | malformed `RunAgentInput`, `protocolVersion` not 1.x, `threadId` not a UUID, trailing message not `user`, history on a new thread | `threadId` missing or not a UUID    |
| `401`  | missing, unknown, revoked or expired token                                                                                        | same                                |
| `403`  | a platform key, or a token for another app                                                                                        | same                                |
| `404`  | unknown app, route not in the active deployment (the message lists the routes), or someone else's thread                          | same                                |
| `409`  | the thread is mid-turn (reconnect instead), is waiting for an approval (answer it with `resume`), or belongs to another route     | the thread belongs to another route |
| `429`  | a [limit](limits.md) — retry after `Retry-After`; nothing was recorded                                                            | never                               |

Failures after the stream starts are `RUN_ERROR` events.

## Local development

`sectr dev` serves the same endpoint from its gateway, with no auth, no
limits and any origin allowed:

```text
🔌 connect via AG-UI at: https://local.sectr.dev:52817/ag-ui/support-agent/chat
```

`{app}` is the local app's name. The port changes each time `sectr dev`
starts.
