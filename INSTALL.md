# Installing Heystack

Heystack is a self-hosted developer dashboard. This guide installs the ready-to-run **release package** on a Linux machine (Raspberry Pi OS, Ubuntu, Debian and similar) with systemd. You do not need to build anything.

## What you need

- Linux with **systemd**, and a normal user account (do not run the installer as root).
- **Node.js 20 or newer** (`node -v`).
- **PostgreSQL 15 or newer**, already installed and running, plus its client tools (`psql`, `pg_dump`; on Debian/Ubuntu: `sudo apt install postgresql postgresql-client`). The installer never installs PostgreSQL for you. The client tools must be **the same version as the server or newer** (`pg_dump` refuses to back up a newer server): the installer and `heystack-doctor` tell you if they are not.
- `curl`.

macOS and other systems: unpack the package and follow `README.txt` inside it to start Heystack by hand.

## Get the package

Download `heystack-<version>.tar.gz` and `SHA256SUMS` from https://github.com/widgemo/heystack-releases/releases (no account needed) and check them:

```bash
VERSION=0.14.1    # the newest version is at the top of the releases page
curl -fsSLO https://github.com/widgemo/heystack-releases/releases/download/v$VERSION/heystack-$VERSION.tar.gz
curl -fsSLO https://github.com/widgemo/heystack-releases/releases/download/v$VERSION/SHA256SUMS
sha256sum -c SHA256SUMS
tar -xzf heystack-$VERSION.tar.gz
cd heystack-$VERSION
```

## Create a database

Heystack needs an empty PostgreSQL database and a user for it. Either let the installer do it, or do it yourself:

```bash
sudo -u postgres psql -c "CREATE ROLE heystack LOGIN PASSWORD 'choose-a-password'" -c "CREATE DATABASE heystack OWNER heystack"
```

Your connection address then looks like `postgresql://heystack:choose-a-password@localhost:5432/heystack`. If you have a PostgreSQL account that may create roles, `./install.sh --admin-url postgresql://admin:password@localhost:5432/postgres` creates a role and database named `heystack` with a random password for you.

## Install

```bash
./install.sh
```

It checks your machine, asks you to accept the Heystack licence (read it in `LICENSE`; in a script use `--accept-license`, because `--yes` alone does not accept it), asks the port (3001) and whether other computers may connect (default: no), and asks for the database address (typed hidden). Then it writes the settings, creates the database tables, installs a systemd user service called `heystack`, starts it and checks that it answers. At the end it prints:

- the address to open (for example `http://127.0.0.1:3001`),
- a **one-time setup code** to create the first account,
- the **encryption key**, once. **Keep a copy somewhere safe.** It protects the provider tokens Heystack stores, and it is not in database backups: without it a restored database cannot read those tokens. It is also in `~/heystack/config/heystack.env` (only you can read that file).

Open the address, create your account with the setup code, and sign in. Sign-in is on from the start.

To start Heystack at boot even when nobody is logged in, run once: `sudo loginctl enable-linger $USER` (the installer prints this when it is needed).

Useful options (`./install.sh --help` lists all): `--prefix DIR` (default `~/heystack`), `--port N`, `--database-url URL`, `--admin-url URL`, `--allow-network --address http://10.0.0.5:3001`, `--no-service`, `--dry-run` (show what would be done, change nothing).

### Let other computers connect

By default Heystack listens on this machine only. To reach it from another computer, answer yes when asked, or install with `--allow-network --address http://<this-machine>:3001`. Sign-in is on, but anything on your network can then reach the login page, so use a trusted network or put Heystack behind a reverse proxy or tunnel with HTTPS.

## Update

Download the newer package, unpack it, and run **its** `install.sh`:

```bash
./install.sh
```

Heystack detects the existing installation and keeps your settings. It backs up the database first (`~/heystack/backups`), applies the new migrations, switches to the new version, restarts, and checks that the new version answers. **If it does not come up, the installer switches back to the previous version by itself** and tells you. The database is migrated forward in that case; migrations only add things, so the previous version keeps working.

## Go back, remove

- Go back to the previous version by hand: `~/heystack/current/install.sh --rollback`
- Remove the program: `~/heystack/bin/heystack-uninstall`. It stops and removes the service and the program files. It keeps `~/heystack/config` (the encryption key!), `~/heystack/backups` and the database. Delete those yourself when you are sure.

## Backups and restoring

The installer sets up a **daily backup** (systemd timer, 02:30, kept 14 days) in `~/heystack/backups`, and also backs up before every update. Skip the timer with `--no-backup-timer` and schedule it yourself if you prefer.

- Back up now: `~/heystack/bin/heystack-backup`
- See the timer: `systemctl --user list-timers`
- Restore: `~/heystack/bin/heystack-restore ~/heystack/backups/<file>.dump`

`heystack-restore` checks the file, asks you to type `RESTORE`, stops Heystack, **saves a copy of your current data first** (`...-before-restore.dump`, so a restore can be undone by restoring that file), replaces the data, applies any newer migrations, checks that the stored tokens can still be read, starts Heystack and waits until it answers. If the restore fails part way, it puts the saved copy back by itself. A backup made by a *newer* Heystack than the one installed is refused (update first).

**The encryption key is not in the backups.** If you restore onto a machine with a different key, the restore finishes but the provider tokens cannot be read: the command exits with 3 and says so. Put the key you saved into `~/heystack/config/heystack.env` (`HEYSTACK_ENCRYPTION_KEY=`), restart, and run `heystack-doctor` to confirm.

For an installation not made by `install.sh`, use `scripts/backup-db.sh` with `HEYSTACK_ENV_FILE` and `HEYSTACK_BACKUP_DIR`.

## Changing the encryption key

`~/heystack/bin/heystack-rotate-key` replaces `HEYSTACK_ENCRYPTION_KEY` with a new random key and re-encrypts the stored provider tokens and secrets. Use it if the key was ever exposed (pasted in a chat, committed by mistake) or as routine housekeeping.

It stops Heystack briefly, takes a backup (`...-before-rotate.dump`), changes everything in one database transaction (if anything fails, nothing changes), writes the new key to `~/heystack/config/heystack.env`, starts Heystack and checks that every stored secret can be read. The new key is shown **once**: keep a copy.

The old key is kept in `~/heystack/config/previous-keys` (readable by you only), because **backups made before the rotation are encrypted with the old key**. When you restore such a backup, `heystack-restore` finds the old key there and re-encrypts the tokens with the current key by itself. You can also run `heystack-rotate-key --recover` after any restore that left tokens unreadable. Keep `previous-keys` with your copy of the key, and delete an old line only when you have deleted every backup that needs it.

## With Docker

Prefer containers? `docker compose` runs Heystack with its own PostgreSQL, a daily backup and sign-in on, with nothing else to install. The image is built from the same package for Raspberry Pi (arm64) and ordinary PCs (amd64):

```
mkdir heystack && cd heystack
curl -fsSLO https://github.com/widgemo/heystack-releases/releases/latest/download/compose.yml
printf 'POSTGRES_PASSWORD=%s\n' "$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')" > .env
docker compose up -d
docker compose logs heystack      # shows the one-time setup code for the first account
```

Then open http://localhost:3001. Settings (including the encryption key) and backups live in the `heystack-data` volume; doctor, backup, restore and key rotation work as described above, run through `docker compose exec` / `docker compose run --rm`. Update with `docker compose pull && docker compose up -d`. The full guide, including restore and key rotation commands, is [DOCKER.md](DOCKER.md).

## Something wrong?

Start with the doctor. It changes nothing; it checks the machine, the settings, the service, the database, the stored secrets and the backups, and says what to do about anything wrong:

```bash
~/heystack/bin/heystack-doctor          # a readable report
~/heystack/bin/heystack-doctor --json   # for scripts
```

To ask for help, run `~/heystack/bin/heystack-doctor --report` and paste the output into an issue (https://github.com/widgemo/heystack-releases/issues/new/choose) or an email to support@widgemo.com. The report says what is installed and what the checks found, and leaves out secrets, passwords, addresses and your home folder (read it before you post it).

The exit code is 0 when everything is fine, 1 with warnings and 2 when something needs fixing. For a Heystack that was not made by `install.sh`, point it at its settings file: `heystack-doctor --env-file /path/to/packages/api/.env --no-service`. What it checks:

- **System:** Node.js 20+, `pg_dump`, free disk space.
- **Settings:** the file is private (mode 600), the database address is set, the encryption key is strong, and sign-in is not off while Heystack listens on the network.
- **Service:** the systemd service runs and is enabled, it will start at boot (`linger`), and the running version is the installed one.
- **Database:** it connects, PostgreSQL 15+, and all migrations are applied.
- **Secrets:** every stored token and webhook secret can be decrypted with the current encryption key (a wrong or lost key shows up here).
- **Backups:** the newest backup is recent.

By hand:

- Is it running? `systemctl --user status heystack`
- Logs: `journalctl --user -u heystack -f`
- The settings: `~/heystack/config/heystack.env` (change a value, then `systemctl --user restart heystack`)
- Check the database tables: `~/heystack/bin/heystack-migrate --status`
- The command line tool: `~/heystack/bin/heystack --help` (if you chose another port: `~/heystack/bin/heystack config set-url http://127.0.0.1:<port>`)
