# Self-hosting

You can run all of Sectr yourself: the server, the dashboard, the journal and your secrets. Builds and agent runners stay on your machine too. It's one Docker Compose stack, with Postgres included.

## Where to run it

**A single VM with Docker is the main target**, such as EC2, Hetzner, DigitalOcean or GCE. A laptop or home server works for trying it out.

Fargate, Cloud Run, Railway, Render and Fly generally won't work, because they don't give you a Docker socket, and the server builds and runs agents through one. Kubernetes isn't supported yet.

The server runs as one instance and holds long-lived connections, so it can't be scaled out behind a load balancer. It needs a direct Postgres connection, not a pooler.

**Starting sizes for a small team:** 4 vCPU, 8–16 GB of RAM and 50+ GB of disk. Builds and agent images are what use the disk.

## Install

You need Docker with the Compose plugin.

```sh
git clone https://github.com/sectr-dev/sectr && cd sectr/selfhost
./setup.sh && docker compose up -d
```

`setup.sh` writes `.env` with generated secrets and prints a platform key. It won't overwrite an existing `.env`.

Open [http://localhost:3000](http://localhost:3000) and sign in with the key. Then deploy an agent:

```sh
curl -fsSL https://sectr.dev/install.sh | sh
sectr auth login --api http://localhost:8080
sectr init my-agent && cd my-agent && sectr deploy
```

Logging in with `--api` once points every later CLI command at your server. `sectr deploy` prints your agent's dashboard link and endpoints.

## The Docker socket

> **Caution**: The server has the Docker socket, which is effectively root on this machine. Anyone who can deploy to your server, or who gets into the server container, can take over the machine. Treat a platform key like a root password, and run Sectr on a machine dedicated to it.

## How it's isolated

|                             | Reaches                                                           |
| --------------------------- | ----------------------------------------------------------------- |
| Postgres                    | The `sectr-internal` network only. It publishes no port           |
| Server                      | `sectr-internal`, and `sectr-runners` so that runners can dial in |
| Dashboard                   | `sectr-internal`                                                  |
| Runners (your agents' code) | `sectr-runners` and the internet                                  |

Each turn runs in a new container, as uid 1000, with CPU, memory and lifetime limits. Runners have no route to the database and hold no platform credentials. They do have:

- **Each other:** runners on the network can reach one another. That's fine for one team on its own machine.
- **Your network and the internet:** agents usually need to call out. On a cloud VM, a runner can reach anything the machine can, which can include the cloud's instance metadata service. Restrict that the way you would for any container, for example with IMDSv2 and a hop limit of 1 on EC2.

There's one organization per install. To give more people access, make more keys in Settings.

## On a remote machine: URLs and TLS

Sectr doesn't terminate TLS. As with most self-hosted software, put a reverse proxy in front of it. There are two Sectr-specific steps.

**1. Set your public URLs in `.env`:**

```sh
SECTR_PUBLIC_API_URL=https://api.sectr.example.com
SECTR_DASHBOARD_URL=https://sectr.example.com
```

To keep the plain-HTTP ports off the internet, publish them on loopback only:

```sh
SECTR_SERVER_PORT=127.0.0.1:8080
SECTR_DASHBOARD_PORT=127.0.0.1:3000
```

**2. Point a reverse proxy at ports `8080` and `3000`.** With [Caddy](https://caddyserver.com), which gets its own certificates:

```
api.sectr.example.com {
    reverse_proxy localhost:8080
}

sectr.example.com {
    reverse_proxy localhost:3000
}
```

Whatever proxy you use, turn off response buffering and allow long-lived connections on the API, because live session streams use server-sent events. Then apply the changes with `docker compose up -d`, and log in with `sectr auth login --api https://api.sectr.example.com`.

## Settings

These go in `.env`. Apply changes with `docker compose up -d`.

| Setting                              | Default                 | What it does                                                            |
| ------------------------------------ | ----------------------- | ----------------------------------------------------------------------- |
| `SECTR_VERSION`                      | Set by the release      | The version to run                                                      |
| `SECTR_PUBLIC_API_URL`               | `http://localhost:8080` | How browsers and the CLI reach the server                               |
| `SECTR_DASHBOARD_URL`                | `http://localhost:3000` | How browsers reach the dashboard. Used in links                         |
| `SECTR_PG_URL`                       | Bundled Postgres        | The database. Must be a direct connection                               |
| `COMPOSE_PROFILES`                   | `postgres`              | Remove `postgres` when you use your own database                        |
| `SECTR_SECRETS_MASTER_KEY`           | Generated               | Encrypts secrets at rest. If you lose it, stored secrets can't be read  |
| `SECTR_BOOTSTRAP_PLATFORM_KEY`       | Generated               | The first platform key                                                  |
| `SECTR_MAX_CONCURRENT_TURNS`         | `10`                    | Turns running at once. Set it to fit the machine                        |
| `SECTR_MAX_CONCURRENT_TURNS_PER_APP` | `0`                     | The same, per app. `0` turns it off                                     |
| `SECTR_RUNNER_CPU_COUNT`             | `1`                     | CPUs per runner. May be fractional                                      |
| `SECTR_RUNNER_MEMORY_MB`             | `1024`                  | Memory per runner                                                       |
| `SECTR_RUNNER_TIMEOUT_SECS`          | `3600`                  | The longest a runner can live. A turn that hits this ends with an error |
| `SECTR_SERVER_PORT`                  | `8080`                  | The server's published port, optionally with an address                 |
| `SECTR_DASHBOARD_PORT`               | `3000`                  | The dashboard's published port, optionally with an address              |
| `SECTR_DOCKER_SOCKET_PATH`           | `/var/run/docker.sock`  | The Docker socket, for rootless Docker                                  |

The defaults for `SECTR_MAX_CONCURRENT_TURNS` and the runner limits allow up to 10 CPUs and 10 GB at once, so lower them on a smaller machine. A turn that would go over the limit is refused, and a client can retry it.

### Your own database

Use any Postgres 16 that the server can reach directly. In `.env`, set `SECTR_PG_URL` to its URL and remove `postgres` from `COMPOSE_PROFILES`. Don't use a connection pooler such as PgBouncer or RDS Proxy: live streams rely on `LISTEN/NOTIFY`.

## Upgrades

Change `SECTR_VERSION` in `.env`, then:

```sh
docker compose pull && docker compose up -d
```

Database migrations run when the server starts. Apps you've deployed keep their built images, so they keep running. Downgrades aren't supported, so back up first.

## Backups

Back up these two things together:

- **The database.** With the bundled Postgres: `docker compose exec postgres pg_dump -U sectr sectr > sectr.sql`
- **Your `.env` file.** Without `SECTR_SECRETS_MASTER_KEY`, the stored secrets in a backup can't be read.

The `artifacts` volume holds your uploaded app code, which you can also redeploy from source. Agent images are rebuilt by deploying again.

## Pruning images

Each deploy builds an image named `sectr-app-<deployment-id>:v<version>`, and these aren't removed automatically. To see them:

```sh
docker images 'sectr-app-*'
```

Remove old ones with `docker rmi` or `docker image prune`. If you remove the image of a deployment that's still in use, its next turn fails with an error telling you to redeploy the app.

## What it sends to Sectr

Nothing. A self-hosted install has no telemetry: the server and dashboard contact no Sectr service. The CLI's `sectr dev` does fetch a TLS certificate for `local.sectr.dev`, and without a login it opens the hosted dashboard, where the page comes from Sectr but your browser talks only to your machine.
