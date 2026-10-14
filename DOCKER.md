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

## Not available in a container

Git Sync's "restart after deploy" uses systemd and does not apply here.
