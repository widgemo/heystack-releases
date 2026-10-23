# What Heystack does not do (yet)

Heystack is in beta. This page is the honest list, so you can decide before you install. Items marked *planned* are on the
roadmap; the rest are choices.

## Deploying

- **It serves nothing itself.** Heystack pulls a branch into a folder on your machine, builds it and restarts the program you
  already run there (PM2 or systemd). Your app's port, proxy and domain are yours. Restarting Docker containers is not
  supported.
- **One checkout and one watched branch per repository.** There are no separate preview environments for every feature branch
  or pull request yet (*planned*). Today the box shows one version of a project: your `dev` branch.
- **Rollback puts the code back, not the database.** Migrations that already ran are not undone.
- **Runbooks run commands on your machine.** That is the point, and it is why access is limited: tokens for scripts and AI
  agents are *read only* or *read and deploy*, deploying with a token must be allowed per project, and force-resync, rollback
  and changing steps need full access.
- **Webhooks: GitHub and Bitbucket.** GitLab webhooks are not supported.

## Providers

- **Integrations are read only.** Heystack reads projects and deployments from Vercel, Netlify, Cloudflare Pages, Render,
  Railway and Fly.io; it does not create DNS records or tunnels for you (the guide shows how to set up a Cloudflare Tunnel by
  hand). Deployment history (and "what is live") needs Vercel today; the others list projects only.
- **The pull request comment and deploy status are GitHub only**, and need a token with *Pull requests: read and write*.

## Running it

- **A Docker install is the dashboard, not the deployer.** A container cannot see your project folders, so Git Sync
  deployments, Pipeline, port checks of your apps and the scanner need the installer. [DOCKER.md](DOCKER.md) has the table.
- **Watching backups needs the same machine, or a report.** Heystack only *watches* backups and jobs (it never runs or restores them). Looking at a folder, a status file or a command works for the installer; a Docker install cannot see the host, so a job there reports in itself with `curl` ([JOBS.md](JOBS.md)).
- **Host health is for the native install.** Inside a Docker container the disk, drives and temperature belong to the container, so the Host page says it is not available and collects nothing. Temperature and the power and throttling flags are read from the Raspberry Pi (and any machine that reports a CPU temperature); other machines simply do not show them.
- **PostgreSQL is required** (15 or newer). The installer can set it up on Debian, Ubuntu and Raspberry Pi OS; there is no
  SQLite option.
- **The installer is for Linux with systemd.** On macOS you start the package by hand; Windows is not supported.
- **One kind of user.** There are accounts and limited tokens, but no roles or teams. Cloudflare Access can control who may
  sign in.
- **Messages go to ntfy or a signed webhook.** There are no built-in e-mail, Slack or Discord channels yet (*planned*); a
  webhook plus a small bridge covers them.
- **AI questions use Anthropic (Claude)** with your own key. There is no OpenAI or local model option.

## About the software

- **Closed source.** Free to use, no redistribution (see [License](README.md#license)).
- **Beta.** It runs the author's own projects every day, but it has had few other users. Back up (the installer does it
  daily), read the release notes before updating, and tell us what breaks ([BETA.md](BETA.md)).
