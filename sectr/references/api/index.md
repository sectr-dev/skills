# API overview

Any client - a web frontend, a mobile app, a script - drives a deployed
agent over the Sectr HTTP API. See the [full reference](openapi.json) for complete details.

## The data plane APIs

| Area      | Endpoints                                                                                                                         | Auth                                                              |
| --------- | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Sessions  | [`POST /sessions`](openapi.json), [`GET /sessions/{sessionId}`](openapi.json), [`POST /sessions/{sessionId}/input`](openapi.json) | user token (end users), invocation key (backend), or platform key |
| Streaming | [`GET /sessions/{sessionId}/events`](openapi.json) (SSE)                                                                          | same as the session's                                             |
| Approvals | [`POST /sessions/{sessionId}/approval`](openapi.json), [`GET /approvals`](openapi.json)                                           | decide: user token (own sessions) / keys; inbox: platform key     |
| AG-UI     | [`POST /ag-ui/{app}/{route}`](openapi.json) (run), [`GET /ag-ui/{app}/{route}`](openapi.json) (reconnect, `?threadId=`)           | user token or invocation key                                      |

Browser clients authenticate with a user token minted by your backend via
[`POST /tokens`](openapi.json) (invocation key) — see [authentication](auth.md).

## The control plane APIs

| Area        | Endpoints                                                                                                         | Auth         |
| ----------- | ----------------------------------------------------------------------------------------------------------------- | ------------ |
| Apps        | [`GET /apps`](openapi.json), [`GET /apps/{appId}`](openapi.json), [`GET /apps/{appId}/deployments`](openapi.json) | platform key |
| Deployments | [`POST /deployments`](openapi.json), [`POST /deployments/{deploymentId}/activate`](openapi.json)                  | platform key |
| Secrets     | [`PUT /secrets/{key}`](openapi.json), [`GET /secrets`](openapi.json), [`DELETE /secrets/{key}`](openapi.json)     | platform key |
| API keys    | [`GET /api-keys`](openapi.json), [`POST /api-keys`](openapi.json), [`DELETE /api-keys/{keyId}`](openapi.json)     | platform key |

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
