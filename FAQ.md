# FAQ

**Is Heystack free?**
Yes, to install and use for yourself or for your organization's internal use, including business use. See [LICENSE](LICENSE).

**Is it open source?**
No. The source code is not published. You may not redistribute it, resell it, offer it as a service or distribute
modified versions of it; you may modify it locally for your own internal use. The licence text is in
[LICENSE](LICENSE), and by downloading, installing or using Heystack you accept it.

**What do I need to run it?**
A Linux machine and either Docker (PostgreSQL is included), or Node.js 20 or newer plus a PostgreSQL 15 or newer
server (the installer uses one you already have; it never installs PostgreSQL). It runs on arm64 (Raspberry Pi)
and amd64. macOS can run the package by hand (see README.txt inside the package); the installer is for Linux with
systemd.

**Where does my data go?**
Nowhere. Everything is stored in your own database on your own machine. There is no telemetry, no account with us
and no update check. Heystack only connects to services you set up (hosting providers, GitHub, your own status
check addresses, and Anthropic if you turn on AI queries). Details: [PRIVACY.md](PRIVACY.md).

**Are the AI queries required?**
No. They are off until you add an Anthropic API key. Without a key the feature is simply unavailable. When you ask a
question, information about your projects is sent to Anthropic to answer it.

**How do I back up and restore?**
The installer and the Docker setup make a daily backup. `heystack-backup` makes one now, `heystack-restore` restores
one (it saves a copy of your current data first). **Keep a copy of your encryption key**: it is not inside backups,
and without it stored provider tokens cannot be read after a restore. See [INSTALL.md](INSTALL.md).

**I lost my encryption key.**
Everything else is intact, but the provider tokens and webhook secrets Heystack stored cannot be decrypted. Enter
those integrations and webhooks again in Heystack. If you changed the key with `heystack-rotate-key`, the older keys
are kept in `config/previous-keys` and restores use them.

**Can I put it on the internet?**
By default it listens on this machine only. To reach it from other computers set `API_HOST=0.0.0.0` (or
`HEYSTACK_BIND=0.0.0.0` with Docker), keep sign-in on, set `HEYSTACK_ALLOWED_ORIGINS` to the address you use, and put
a TLS-terminating proxy in front. Heystack can run commands on the machine it is installed on (deployment runbooks):
treat access to it like shell access.

**How do I update?**
Run the installer from the newer package, or `docker compose pull && docker compose up -d`. It takes a backup first
and rolls back by itself if the new version does not start. `./install.sh --rollback` goes back by hand.

**How do I uninstall?**
`~/heystack/bin/heystack-uninstall` removes the program and the service and keeps your settings, backups and database.
For Docker: `docker compose down` (add `-v` to also delete the data).

**Something is wrong.**
Run `~/heystack/bin/heystack-doctor` (or `docker compose exec heystack heystack-doctor`). It checks the machine, the
settings, the service, the database, the stored secrets and the backups and says what to do about each problem.

**How do I get help or report a problem?**
Open an [issue](https://github.com/widgemo/heystack-releases/issues/new/choose) and paste the output of `heystack-doctor --report` (it leaves out secrets, addresses and your home folder), or email support@widgemo.com. Security problems: see [SECURITY.md](SECURITY.md).
