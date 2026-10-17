# Authentication

Three bearer credentials authenticate against the API. They differ in scope, and the
server derives behavior from which one a request carries.

## Platform API key

- HTTP header: `Authorization: Bearer skp_…`

Org-scoped; minted in the dashboard (Settings → API keys). For your own
tools and scripts (it's what the CLI and the dashboard use): the control plane
(apps, deployments, secrets, API keys), and every session in the org. Sessions
it creates have no owner and skip the session-creation
[rate limits](limits.md). It can't mint user tokens or call the
[AG-UI endpoint](ag-ui.md) (`403`): those take an invocation key
or a user token.

## Invocation API key

- HTTP header: `Authorization: Bearer ski_…`

Per-app; the **trusted-backend / minting credential**. Scoped to one app (the
`app_id` is minted with the key), with the full data plane for that app: it
can create sessions and read, stream, and talk to **every** session of the app
(no per-user scoping).

> **Never ship this key to a browser in a multi-user app**: Anyone holding it can read every end user's transcripts and create unlimited
> sessions (compute abuse). For multi-user apps, keep it on your backend and
> mint [user tokens](#user-token-sks_) for browsers.

```sh
curl -X POST https://api.sectr.dev/sessions \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"app": "support-agent", "route": "/chat", "input": {"message": "hi"}}'
```

A leaked invocation key is contained to its app: it can't read other apps'
sessions, touch secrets, or deploy. Rotate freely: new sessions use the new
key, existing sessions live in the journal regardless.

**Single-user / personal / internal apps** may use `ski_` directly as the
client credential — that just works. If such an app later gains end users,
note that (a) transcripts are shared between everyone holding the key, and
(b) sessions created with `ski_` have **no owner**, so user tokens can never
see them — they stay backend-only forever. A public page holding `ski_` also
lets anyone create sessions; per-app concurrency quotas bound, but don't
remove, that exposure.

## User token (`sks_…`)

- HTTP header: `Authorization: Bearer sks_…`
- SSE only: `?token=sks_…` (`EventSource` cannot set headers). Redact the
  `token` query parameter from any access logs you keep.

The browser credential for multi-user apps. Opaque, short-lived (24h by
default), bound to one `(app, external_user_id)`. It can create sessions
owned by that user, **list only that user's sessions** (`GET /sessions` is
filtered automatically; an `external_user_id` query parameter is rejected),
and get/stream/send input/decide approvals on sessions that user owns. It can
never see another user's — or an ownerless — session.

Sectr never authenticates end users: your app's own auth stays the source of
truth. Your backend asserts identity when minting:

```sh
curl -X POST https://api.sectr.dev/tokens \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"app": "support-agent", "external_user_id": "user_2938"}'
# 201 {"token": "sks_…"}
```

- `external_user_id` is an opaque, app-defined string (max 512 chars). It is
  immutable on a session — if you rotate your user ids, old sessions are
  orphaned.
- Mint once per page load and reuse the token for creating sessions, listing,
  and every stream. On a `401`, re-mint or reload.
- An open SSE stream ends when its token expires or is revoked (revocation is
  re-checked about every 30 seconds). Reconnect with a fresh token.
- Minting is rate-limited per (key, `external_user_id`); `429` when exhausted.
- Tokens are tied to the invocation key that minted them: revoking that key
  immediately revokes every token it minted (and it can mint no more). Rotate
  the invocation key if it leaks.
- Anonymous/guest apps: mint with a generated anonymous id bound to a
  cookie/localStorage.

Full schema: `POST /tokens` (look it up with `jq '.paths["/tokens"].post' references/api/openapi.json`).
