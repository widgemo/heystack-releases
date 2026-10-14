# Heystack

**A self-hosted dashboard and registry for your development projects.** Keep every project, port, repository, hosting
service and deployment in one place on a machine you control, and see at a glance what is up, what changed and
what needs attention.

Heystack runs on a Raspberry Pi or any Linux machine, needs a PostgreSQL database, and sends nothing to us: there is
no telemetry and no account. This repository holds the downloads and the user documentation. Heystack is
proprietary software, free to install and use (see [License](#license)); its source code is not published.

## What it does

- **Project registry**: name, status, notes, stack, ports, repositories and hosting links for each project.
- **Status checks and history**: ping ports and URLs on a schedule and see a timeline of up and down.
- **Pipeline and releases**: branch state, uncommitted changes and unreleased commits per project; deployment state
  from Vercel, Netlify, Render, Railway, Fly.io and Cloudflare.
- **Git Sync**: deploy a project when you push to a branch (GitHub webhooks), with a runbook of steps and a deploy
  history.
- **Project scanner**: find the projects on your disk and import them.
- **AI queries** (optional, off by default): ask questions about your projects in plain English, answered by Claude
  with your own API key.
- **Web dashboard and command line tool**, sign-in with accounts, health alerts.
- **Looks after itself**: `heystack-doctor` finds problems, daily backups, restore, and encryption key rotation.

## Install

You need a Linux machine (a Raspberry Pi 4 or 5 is fine) and either Docker or Node.js 20+ with PostgreSQL 15+.

**With Docker** (PostgreSQL included, arm64 and amd64):

```
mkdir heystack && cd heystack
curl -fsSLO https://github.com/widgemo/heystack-releases/releases/latest/download/compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')" > .env
docker compose up -d
docker compose logs heystack     # shows the one-time setup code for your first account
```

Open http://localhost:3001 and create the first account with that code. See [DOCKER.md](DOCKER.md).

**With the installer** (Linux with systemd, uses a PostgreSQL server you already have): download the package from the
[releases](https://github.com/widgemo/heystack-releases/releases), check it and run `./install.sh`. See
[INSTALL.md](INSTALL.md).

Updating is the same command again with the newer package (the installer backs up first and switches back by
itself if the new version does not start), or `docker compose pull && docker compose up -d`.

## Documentation

- [INSTALL.md](INSTALL.md): requirements, install, update, backups and restore, key rotation, troubleshooting
- [DOCKER.md](DOCKER.md): running with Docker Compose
- [FAQ.md](FAQ.md)
- [BETA.md](BETA.md): what to try and how to tell us what is wrong
- [PRIVACY.md](PRIVACY.md): what leaves your machine
- [SECURITY.md](SECURITY.md): reporting a vulnerability

## License

Free to install and use for yourself or for your organization's internal use, including business use. Not open
source: you may not redistribute it, resell it, host it as a service for others or distribute modified versions;
you may modify it locally for your own internal use. Full terms in [LICENSE](LICENSE); the licences of the
third-party software it contains are in `THIRD-PARTY-NOTICES.txt` inside every download. By downloading, installing
or using Heystack you accept the licence.

## Support and feedback

Found a problem or have an idea? Open an [issue](https://github.com/widgemo/heystack-releases/issues/new/choose)
(run `heystack-doctor --report` first and paste the output), or email support@widgemo.com.
