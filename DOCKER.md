# Heystack in Docker

Heystack with its own PostgreSQL, started with Docker Compose. The image is built from the release package, for
arm64 (Raspberry Pi) and amd64.

## Start

```
mkdir heystack && cd heystack
curl -fsSLO https://github.com/widgemo/heystack-releases/releases/latest/download/compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')" > .env
docker compose up -d
docker compose logs heystack
```

The log shows a one-time setup code. Open http://localhost:3001 and create the first account with it. Sign-in is
on from the start; the port is published on this machine only (`HEYSTACK_BIND=0.0.0.0` in `.env` opens it to other
computers; then set `HEYSTACK_URL` to the address people type, and put a TLS-terminating proxy in front).

## What lives where

- Volume `heystack-db`: the PostgreSQL data.
- Volume `heystack-data`: `/data/config/heystack.env` (settings, **including the encryption key**, mode 600) and
  `/data/backups` (a backup right after start and then daily, the newest 14 kept).

**Keep a copy of the encryption key**: `docker compose exec heystack grep HEYSTACK_ENCRYPTION_KEY /data/config/heystack.env`.
It protects the stored provider tokens and is not in the backups.

## Everyday commands

```
docker compose exec heystack heystack-doctor        # what is wrong, and how to fix it
docker compose exec heystack heystack-backup        # a backup now
docker compose exec heystack ls -l /data/backups
```

Restore a backup and change the encryption key need Heystack stopped:

```
docker compose stop heystack
docker compose run --rm heystack heystack-restore /data/backups/<file>.dump
docker compose run --rm heystack heystack-rotate-key
docker compose start heystack
```

(Restore saves a copy of the current data first. `heystack-rotate-key` keeps the old key in `config/previous-keys`
so older backups stay restorable. Both are described in [INSTALL.md](INSTALL.md).)

## Update

```
docker compose pull
docker compose up -d
```

The database is backed up by the daily job, migrations run on start. To go back, set `HEYSTACK_IMAGE` in `.env` to
the earlier version (for example `ghcr.io/widgemo/heystack:0.12.0`) and run `docker compose up -d`.

## What works in a container, and what needs the installer

Heystack in a container sees only the container: not the project folders on your machine and not the programs that
run there. So, from the same Heystack:

| Works in Docker | Needs the installer on the machine itself |
|---|---|
| The project registry, notes, hosting links, releases timeline | **Git Sync deployments**: pulling a branch into a project folder and running its runbook (install, build, restart) |
| Address (URL) checks, uptime and incidents, notifications | **Port checks** of your apps: a port check looks at the container's own `localhost`, not your machine |
| Provider integrations (Vercel, Netlify, …) and GitHub reporting | **Pipeline and Branches**: reading and changing the git state of your project folders |
| AI questions, access tokens, the API and the command line | **The project scanner** and **restarting** processes (PM2, systemd) |

If you want Heystack to deploy your projects to the machine it runs on (its main use for a developer's own box), install
it with the installer (see [INSTALL.md](INSTALL.md)): it runs as a service on that machine and can reach everything. Docker is
the quick way to look at Heystack, or to keep the registry and monitoring on a server that does not deploy anything.
(Giving the container your project folders and the programs that restart them is possible, but not something Heystack
supports or tests.)

Git Sync's "restart after deploy" uses systemd and does not apply here.

Git Sync's "restart after deploy" uses systemd and does not apply here.
