# API overview

Any client - a web frontend, a mobile app, a script - drives a deployed
agent over the Sectr HTTP API. See the [full reference](openapi.json) for complete details.

## The data plane APIs

| Area      | Endpoints                                                           | Auth                                                              |
| --------- | ------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Sessions  | `POST /sessions`, `GET /sessions/{id}`, `POST /sessions/{id}/input` | user token (end users), invocation key (backend), or platform key |
| Streaming | `GET /sessions/{id}/events` (SSE)                                   | same as the session's                                             |
| Approvals | `POST /sessions/{id}/approval`, `GET /approvals`                    | decide: user token (own sessions) / keys; inbox: platform key     |
| AG-UI     | `POST /ag-ui/{app}/{route}` (run), `GET …?threadId=` (reconnect)    | user token or invocation key                                      |

Browser clients authenticate with a user token minted by your backend via
`POST /tokens` (invocation key) — see [authentication](auth.md).

## The control plane APIs

| Area        | Endpoints                                                     | Auth         |
| ----------- | ------------------------------------------------------------- | ------------ |
| Apps        | `GET /apps`, `GET /apps/{id}`, `GET /apps/{id}/deployments`   | platform key |
| Deployments | `POST /deployments`, `POST /deployments/{id}/activate`        | platform key |
| Secrets     | `PUT /secrets/{key}`, `GET /secrets`, `DELETE /secrets/{key}` | platform key |
| API keys    | `GET/POST /api-keys`, `DELETE /api-keys/{id}`                 | platform key |

## API Base URLs

- Prod: `https://api.sectr.dev`
- Locally, `sectr dev` serves the same API from its gateway, printed at server startup (`https://local.sectr.dev:8420` by default)

## Start here

1. [Authentication](auth.md) — which key class you need
2. [Send a message](send-message.md) — create a session, add turns
3. [Stream a session](stream.md) — the SSE tail + resumption
4. [Approve or deny from your UI](approvals.md)
5. [AG-UI endpoint](ag-ui.md) — CopilotKit and other AG-UI clients
6. [Deploy lifecycle](https://docs.sectr.dev/reference/api/deploy-lifecycle/)
7. [Limits](limits.md) — concurrency, rate limits, and the `429` codes
