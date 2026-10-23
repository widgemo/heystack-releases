# Watching your backups and scheduled jobs

Heystack shows whether your projects are up and deployed. **Jobs & backups** adds the question "is my data safe?":
for each backup, database dump or scheduled task you tell Heystack how often it should succeed, and it tells you,
on the dashboard and on your phone, when it did not.

Heystack **only watches**. It never runs, schedules or restores a backup, so it works with whatever you already use:
restic, borg, rsync, `pg_dump`, rclone, a cloud service, your own script.

## How it works

A **job** has:

- a name and a kind (**Backup**, **Database dump**, **Other job**),
- how often it should succeed (*every 1 day*) and a **grace** time before it counts as late (*6 hours*, so a backup that
  takes long, or starts a little late, is not an alarm),
- the projects it covers: one, several, or **every project** (a backup of the whole machine),
- how Heystack learns about it (below).

Its status is one of:

| Status | Meaning |
|---|---|
| **OK** | The last success is inside the interval plus grace. |
| **Failed** | The last run failed (a failing exit code, a failed report, an empty dump). |
| **Overdue** | No success within the interval plus grace. A job that *stops* running ends up here. |
| **Waiting** | A new job that has not run yet. It gets one full interval. |

You see it on the **Jobs** page, as a **Backups & jobs** card on each project, as a chip on the project's dashboard
card, as a **Backup** column on the Status page, and with `heystack jobs`.

## Add a job

**Jobs, + Add job.** Choose how Heystack learns about it. You can mix the ways across jobs.

### 1. Heystack looks: the newest file in a folder

Best for database dumps and any tool that leaves a file behind. Give the folder, a name pattern such as `*.dump`, and a
**smallest sensible size** so an empty file counts as a failure, not a success.

```
Folder:  /home/you/backups/myapp
Pattern: *.dump
Smallest size: 1000
```

### 2. Heystack looks: a status file

A small file your script writes at the end, in either shape:

```json
{"status":"ok","finishedAt":"2026-10-22T04:00:11Z","sizeBytes":921000000,"durationSeconds":16,"message":"snapshot dd55cfb0"}
```

or just a time on the first line and a note on the second:

```
2026-10-22 04:00:11
snapshot dd55cfb0
```

`status` is `ok` or `failed`; everything but the status is optional (the file's own time is used when it names none).

### 3. Heystack looks: a command

Heystack runs the command (with `sh`, as the user Heystack runs as, with a time limit) and reads the result:
*exit code 0 is success*, or the command prints a status as JSON (as above), or use the **restic preset**, which reads
the newest snapshot of a repository without opening any of your data:

```
Repository:    /mnt/backup/restic-repo
Password file: /home/you/.config/restic/password
```

A check that **cannot tell** (a backup disk that is not mounted, a repository that cannot be opened) is not a failed
backup. Heystack records no run, shows the reason, and the job becomes overdue if it stays that way.
**Never put a password in a command**: use a password file or a key file.

### 4. The job reports in

Add the job with *Reports in itself*. Heystack shows you a **token once**; the script calls Heystack when it finishes:

```bash
curl -fsS -X POST "http://127.0.0.1:3001/jobs/<job id>/report" \
  -H "Authorization: Bearer <job token>" -H "Content-Type: application/json" \
  -d '{"status":"ok","sizeBytes":921000000,"durationSeconds":16,"message":"snapshot dd55cfb0"}'
```

`status` is `ok` or `failed`; a failure is announced at once. A job that stops reporting becomes overdue. The token
belongs to this one job (it can do nothing else), is stored only as a hash, and can be replaced on the job's panel.
Reports work from **any machine** that can reach Heystack, and in **Docker** installs, where Heystack cannot look at
your folders. A script on the same machine as Heystack uses `http://127.0.0.1:<port>`; if Heystack sits behind
Cloudflare Access, a script elsewhere needs the path `/jobs/*/report` to bypass Access (it is protected by the token).

## Recipes

The pattern for **any** tool: run it, then report success or failure.

```bash
#!/usr/bin/env bash
# the backup, whatever it is
if restic -r /mnt/backup/restic-repo backup /home/you/projects --quiet; then status=ok; else status=failed; fi

curl -fsS -X POST "http://127.0.0.1:3001/jobs/$JOB_ID/report" \
  -H "Authorization: Bearer $JOB_TOKEN" -H "Content-Type: application/json" \
  -d "{\"status\":\"$status\"}"
```

- **restic**: either the preset above (no change to your script), or the recipe with `restic backup`.
- **borg**: `if borg create --stats ::'{hostname}-{now}' /home/you/projects; then status=ok; else status=failed; fi`, then report.
- **rsync**: `if rsync -a --delete /home/you/projects/ /mnt/backup/projects/; then touch /mnt/backup/projects/.last-ok; fi`
  and add a job that looks at the file `/mnt/backup/projects/.last-ok`.
- **PostgreSQL dumps**: a job that looks at the dump folder (`*.dump`, smallest size 1000), for example the daily dump
  Heystack itself makes in `~/heystack/backups`.
- **A cron task**: `0 3 * * * /home/you/bin/cleanup.sh && curl -fsS ... -d '{"status":"ok"}' || curl -fsS ... -d '{"status":"failed"}'`
  (the two `curl` lines are the same call with a different status).
- **rclone and cloud tools**: the recipe above around `rclone sync`.

## Messages

Open **Settings, Notifications** and tick **A backup or job failed**, **is overdue** and **is back to normal** on a
channel (new channels have them ticked; existing channels are not changed). A failed report is announced at once; an
overdue job after two checks in a row, so a mount that hiccups is not news; "back to normal" is sent once, after a
message about the problem. Every job has its own **messages on/off** switch.

**Settings, Monitoring and messages** has the switch for the whole feature (off means nothing is checked, shown or
sent, and reports are refused), a switch for command checks, and **Pause every message** for 1, 4 or 24 hours (a planned
reboot, maintenance).

## Test a restore

A backup you never restored is a hope. Give a job **Remind me to test a restore every N days** and press **I tested a
restore today** on its panel; Heystack marks it *restore test due* when the time is up.

## For scripts and agents

`heystack jobs` lists the jobs, failed and overdue first; `heystack jobs --check` exits with 1 when any failed or is
overdue, and `--json` prints them for a program. The API has `GET /jobs`, `GET /jobs/<id>` and
`GET /jobs/<id>/runs` for read-only tokens (see [AGENTS.md](AGENTS.md)); `heystack-doctor` warns when a watched job
needs attention.

## Limits

- Heystack watches; it does not run or restore backups, and it cannot tell whether a backup is *complete*, only that it
  ran, how big it was and, with a status file or report, what the job said about itself.
- Folder, status-file and command checks need Heystack to run on the same machine as the backup (the installer, not a
  Docker container). Use reports otherwise.
