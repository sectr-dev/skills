# Limits

Sectr is **free during the beta**, and a few guardrails keep it fair and
available for everyone. Limits are **server policy**: they apply on
`api.sectr.dev`; the local `sectr dev` gateway never returns them.

## Defaults

| Limit                           | Default                 | Applies to                                                             |
| ------------------------------- | ----------------------- | ---------------------------------------------------------------------- |
| Concurrent turns per project    | 3                       | every credential of the project                                        |
| Concurrent turns, platform-wide | set by Sectr (capacity) | all projects                                                           |
| Session creation per end user   | 60 / hour, burst 10     | [user tokens](auth.md) (`sks_`), per `(project, user)`                 |
| Session creation per project    | 600 / hour, burst 60    | [invocation keys](auth.md) (`ski_`), shared by everyone using that key |

The platform key (dashboard playground, CLI) is exempt from the session
creation rate limits. A turn counts against the concurrency limits while it
is running; a turn suspended on an [approval](approvals.md)
holds no slot. Follow-up inputs and approval decisions start a turn, so they
count too.

## The 429 response

Rejections are `429 Too Many Requests` with a `Retry-After` header (seconds)
and a JSON body:

```json
{ "error": "this app is running its maximum of 3 turns at once; retry shortly",
  "code": "concurrency_limit" }
```

| `code`              | Meaning                                                       | `Retry-After`             |
| ------------------- | ------------------------------------------------------------- | ------------------------- |
| `concurrency_limit` | Your project already has its maximum number of turns running. | 5 s                       |
| `platform_capacity` | Sectr as a whole is at capacity right now.                    | 10 s                      |
| `rate_limited`      | Too many sessions created recently (per user or per project). | time until the next token |

A rejected `POST /sessions/{id}/input` or `/approval` changes nothing: no
event is journaled, so it can simply be retried. A rejected `POST /sessions`
normally creates no session either. Rarely, when another request takes the
last slot at the same moment, an empty session is left behind; it appears in
`GET /sessions` and is harmless to ignore.

## Retrying

Wait at least `Retry-After` seconds, then retry the same request, with a
little jitter. Don't retry in a tight loop. In a browser UI, surface a
"busy, retrying…" state rather than an error.
