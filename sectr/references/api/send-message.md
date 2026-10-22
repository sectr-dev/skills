# Send a message

A conversation is a **session**: create it once, then send follow-up turns.
Both verbs schedule work and return immediately — output arrives on the
[SSE journal tail](stream.md), not in the response.

## Create a session — `POST /sessions`

```sh
curl -X POST https://api.sectr.dev/sessions \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"app": "support-agent", "route": "/chat", "input": "Where is order 123?"}'
```

`201` with the session id:

```json
{ "session_id": "0198c7a2-…" }
```

- `app` — the project slug (as `sectr deploy` created it) or its id;
- `route` — which agent handles the conversation, by its path (e.g. `/chat`;
  the project's agents are in `GET /apps/{id}`);
- `input` — the user's message text, a string. It is journaled verbatim and
  arrives in the agent as `ctx.input`. Anything else is a **400**: an object,
  a list of content parts, a number, or a string of only whitespace (on
  follow-up turns too). Some model providers reject an empty user message.

Creating the session starts executing your agent immediately.

Full schema: `POST /sessions` (look it up with `jq '.paths["/sessions"].post' references/api/openapi.json`).

## Follow-up turns — `POST /sessions/{id}/input`

```sh
curl -X POST https://api.sectr.dev/sessions/$SESSION_ID/input \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": "and a refund please"}'
```

`202` — the turn is scheduled. **409** means the session isn't accepting
input right now: mid-turn (the previous turn's lease is live), awaiting an
approval decision (decide it first — a new message would strand the
unanswered tool call), or ended. A 409 after an *errored* turn is
deliberately impossible: sending a fresh message is the recovery path.

Full schema: `POST /sessions/{sessionId}/input` (look it up with `jq '.paths["/sessions/{sessionId}/input"].post' references/api/openapi.json`).
