# Approve or deny actions

When a session's turn suspends on a gated tool call, your UI can present the
decision.

## List pending approvals — `GET /approvals`

```sh
curl -s https://api.sectr.dev/approvals \
  -H "Authorization: Bearer $SECTR_PLATFORM_KEY"
```

Platform-key only. Returns pending approvals **across all apps** — each item pairs the session
with the `TOOL_APPROVAL_REQUIRED` event envelope, so your UI renders the
tool name, arguments, and reason directly from the payload. Per-session
status also flips to `awaiting_approval` (visible in `GET /sessions/{id}`
and `GET /sessions`).

Full schema: `GET /approvals` (look it up with `jq '.paths["/approvals"].get' references/api/openapi.json`).

## Decide — `POST /sessions/{id}/approval`

Allowed for a platform key, the app's invocation key, or a
[user token](auth.md#user-token-sks_) on a session that user
owns — so an in-app approval button needs no server-side proxy.

```sh
curl -X POST https://api.sectr.dev/sessions/$SESSION_ID/approval \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"approved": true, "approver": "ops-dashboard"}'
```

`202` — the decision is journaled (`TOOL_APPROVAL_GRANTED`/`DENIED`) and the
resume is scheduled; the resumed turn streams on the same
[journal tail](stream.md) as everything else.

- **`approved`** — required boolean. `false` denies: the agent receives the
  rejection as the tool's result and continues (it sees a denied call, not a
  crash).
- **`approver`** — optional attribution. Left out, a user token's decision
  is attributed to its user (`external_user_id`); with a key, no approver is
  recorded. The CLI sends `"sectr send"`; from a backend, send your user's id.

**409** = no pending approval on that session (already decided, or the turn
ended). **404** = unknown session. Decide exactly once — the decision is a
journal event, not a setting.

Full schema: `POST /sessions/{sessionId}/approval` (look it up with `jq '.paths["/sessions/{sessionId}/approval"].post' references/api/openapi.json`).
