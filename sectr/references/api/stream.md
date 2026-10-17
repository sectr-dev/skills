# Stream a session

The session stream (`GET /sessions/{id}/events`, `text/event-stream`) is a
tail of the journal. Each SSE frame's
`data:` is one event envelope, and the SSE
`id:` is the event's UUIDv7 `id` - the journal key you resume with.

```text
id: 0198c7a2-9f7e-7cc3-bde1-9d24f2c0e001
data: {"id":"…","seq":0,"type":"SESSION_CREATED","payload":{…}}

id: 0198c7a2-9f7e-7cc3-bde1-9d24f2c0e002
data: {"id":"…","seq":3,"type":"MESSAGE_DELTA","payload":{"message_id":"a1b2c3d4e5f60718","delta":{"type":"text","text":"Let me"}}}
```

- Every frame carries the full envelope: `id`, `session_id`,
  `deployment_id`, `seq` (per-session monotonic), `type`, `payload` and
  `created_at`. A chat UI renders the `MESSAGE_*` types (the agent's text),
  `TOOL_CALL` / `TOOL_RESULT` (its tool use) and `TOOL_APPROVAL_REQUIRED`
  (a decision to present); the rest is lifecycle (inspect, audit, replay).
- The stream **replays the full journal from seq 0**, then holds the tail
  open. A fresh client renders history itself, no separate read API.

Full schema: `GET /sessions/{sessionId}/events` (look it up with `jq '.paths["/sessions/{sessionId}/events"].get' references/api/openapi.json`).

## Resumption — `Last-Event-ID`

Disconnect, then reconnect with the last frame's `id`:

```sh
curl -N "https://api.sectr.dev/sessions/$SESSION_ID/events" \
  -H "Authorization: Bearer $SECTR_INVOCATION_KEY" \
  -H "Last-Event-ID: 0198c7a2-9f7e-7cc3-bde1-9d24f2c0e002"
```

Replay restarts strictly after that event: exactly-once
delivery, no gaps, no duplicates. First connect can pass the same value as
the `last_event_id` query parameter instead of the header.
