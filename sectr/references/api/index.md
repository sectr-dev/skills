# API overview

Any client - a web frontend, a mobile app, a script - drives a deployed
agent over the Sectr HTTP API. See the [full reference](openapi.json) for complete details.

The full reference is `references/api/openapi.json`: every operation's exact request and response schemas. Look up what you need rather than reading it whole: `jq '.paths | keys' references/api/openapi.json` lists the endpoints, `jq '.paths["/sessions/{sessionId}/approval"].post' references/api/openapi.json` shows one, and `jq '.components.schemas.ApprovalDecision' references/api/openapi.json` resolves a `$ref` it names.

## The data plane APIs

| Area      | Endpoints                                                                               | Auth                                                              |
| --------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Sessions  | `POST /sessions`, `GET /sessions/{sessionId}`, `POST /sessions/{sessionId}/input`       | user token (end users), invocation key (backend), or platform key |
| Streaming | `GET /sessions/{sessionId}/events` (SSE)                                                | same as the session's                                             |
| Approvals | `POST /sessions/{sessionId}/approval`, `GET /approvals`                                 | decide: user token (own sessions) / keys; inbox: platform key     |
| AG-UI     | `POST /ag-ui/{app}/{route}` (run), `GET /ag-ui/{app}/{route}` (reconnect, `?threadId=`) | user token or invocation key                                      |

Browser clients authenticate with a user token, which your backend mints
with its invocation key via `POST /tokens` (look it up with `jq '.paths["/tokens"].post' references/api/openapi.json`);
see [authentication](auth.md).

## The control plane APIs

| Area        | Endpoints                                                         | Auth         |
| ----------- | ----------------------------------------------------------------- | ------------ |
| Projects    | `GET /apps`, `GET /apps/{appId}`, `GET /apps/{appId}/deployments` | platform key |
| Deployments | `POST /deployments`, `POST /deployments/{deploymentId}/activate`  | platform key |
| Secrets     | `PUT /secrets/{key}`, `GET /secrets`, `DELETE /secrets/{key}`     | platform key |
| API keys    | `GET /api-keys`, `POST /api-keys`, `DELETE /api-keys/{keyId}`     | platform key |

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
