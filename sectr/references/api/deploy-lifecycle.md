# Deploy lifecycle

Deploys are asynchronous by design: the upload returns immediately, the
build runs off-request, and activation is a separate, observable state
change. Status lives on the deployment row:

```text
building ──▶ active ──▶ retired
    │
    └──▶ failed (with error)
```

## Upload — `POST /deployments`

```sh
curl -X POST https://api.sectr.dev/deployments \
  -H "Authorization: Bearer $SECTR_PLATFORM_KEY" \
  -F "tarball=@agent.tar.gz" \
  -F "manifest=@sectr.toml"
```

`202` with the deployment row in `building` state:

```json
{
  "id": "0198c7a2-…", "app_id": "…", "version": 3,
  "status": "building", "created_at": "…", "active": false
}
```

The tarball is your project (packaged exactly as `sectr deploy` does —
respecting `.gitignore`); the manifest is the `sectr.toml` contents. The
server builds the runner image (prebuilt base + your code), then runs it
once with `SECTR_MANIFEST=1` to extract the **authoritative** manifest by
real import — an unimportable entry fails the build, honestly.

**Validation failures** are immediate: `400` for unreadable multipart,
`422` for a bad manifest (not TOML, missing/empty `agent.entry`) or a
server-side rejection (e.g. a version collision from concurrent deploys of
the same app).

## Poll — `GET /apps/{appId}/deployments`

```sh
curl -s "https://api.sectr.dev/apps/$APP_ID/deployments" \
  -H "Authorization: Bearer $SECTR_PLATFORM_KEY"
```

Poll until the uploaded row's `status` reaches `active` or `failed` — a
`failed` row carries its truncated build output in `error`. This is what
`sectr deploy` does with `--timeout-secs` (default 600): it exits on a
**resolved** build, not on "accepted".

## Activation and rollback

The newest **successful** deployment is automatically current — every new
session pins to it. Existing sessions keep the deployment they started on
(an in-flight conversation is never interrupted by a deploy).

To promote an older build (rollback) or re-activate any resolved one:

```sh
curl -X POST https://api.sectr.dev/deployments/$DEPLOYMENT_ID/activate \
  -H "Authorization: Bearer $SECTR_PLATFORM_KEY"
```

`202` accepted; `409` if the target is `building`/`failed` (only resolved
rows can be activated). From then on, new sessions pin to that version.

## TypeScript

```ts
// createDeployment (multipart) → poll → report
const form = new FormData();
form.append("tarball", new Blob([tarball]), "agent.tar.gz");
form.append("manifest", sectrToml);
const dep = await fetch("https://api.sectr.dev/deployments", {
  method: "POST", headers: { Authorization: `Bearer ${platformKey}` }, body: form,
}).then((r) => r.json<Deployment>());   // 202, status: "building"

for (;;) {
  const rows = await fetch(`https://api.sectr.dev/apps/${dep.app_id}/deployments`, {
    headers: { Authorization: `Bearer ${platformKey}` },
  }).then((r) => r.json<Deployment[]>());
  const row = rows.find((d) => d.id === dep.id)!;
  if (row.status === "active") break;
  if (row.status === "failed") throw new Error(row.error ?? "build failed");
  await new Promise((r) => setTimeout(r, 2000));
}
```

The CLI's poller (`crates/sectr-cli/src/deploy.rs`) is the reference
implementation of this loop.
