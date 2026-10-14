# Security

## Reporting a vulnerability

Email **support@widgemo.com** with "Heystack security" in the subject. Please include what you found, the Heystack
version (`heystack-doctor` prints it), and how to reproduce it. Do not open a public issue and do not post details
publicly until we have had a chance to fix it.

We will acknowledge your report as soon as we can, keep you informed, and tell you when a fix is released.

## Supported versions

Only the latest release is supported. Update with the installer (`install.sh`) or
`docker compose pull && docker compose up -d`.

## What Heystack protects, and what it does not

- Sign-in is on by default for installations made with the installer or Docker. Heystack listens on `127.0.0.1`
  unless you choose otherwise; put a TLS-terminating proxy in front before exposing it to a network.
- Provider tokens and webhook secrets are stored encrypted in the database with `HEYSTACK_ENCRYPTION_KEY`. The key
  is **not** in backups: keep a copy somewhere safe. `heystack-rotate-key` replaces it.
- Heystack can run commands on the machine it is installed on (deployment runbooks). Treat access to Heystack like
  shell access to that machine, and keep it private.
- `heystack-doctor` checks the installation for the common mistakes.
