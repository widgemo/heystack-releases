# Using Heystack with an AI agent or a script

Heystack has an API and a command line tool that an AI coding agent (or any script) can use to see what is going on
and to deploy a branch to your own machine, without ever getting your password or the power to change your setup.

## What you can give an agent

An access token has one of three levels. Create it in **Settings, Account, Access tokens** and choose the smallest
level that does the job:

| Level | Can | Cannot |
|---|---|---|
| **Read only** | Look at everything: projects, git state, releases, status, deployments and their logs, ask a question | Change anything, start anything |
| **Read and deploy** | Also start a deployment (or a pull) and refresh checks, for projects that allow it | Change settings, steps or secrets; force-resync, roll back, create tokens |
| **Full access** | Everything you can do | (this is for your own command line, not for an agent) |

A token with deploy access can deploy a project only after you have allowed it, because a deployment runs the
project's runbook, which runs commands on your machine:

- for one project: open the project, **Settings** tab, **Access tokens may deploy this project**, choose *Allow*; or
- for every project: **Settings, General, Agent access**, choose *Every project, except those that forbid it*.

A refused request answers `403` with a sentence that says what to change.

## Set it up (five minutes)

1. In Heystack, **Settings, Account**, create a token named for the agent, level **Read and deploy**. Copy it now; it
   is shown once.
2. Allow the project as described above.
3. Give the agent two things: the address of your Heystack API (for example `https://heystack-api.example.com`, or
   `http://localhost:3001` on the same machine) and the token. Keep the token out of the code: put it in an
   environment variable.

```
export HEYSTACK=http://localhost:3001      # your Heystack API address
export TOKEN=hsk_...                       # the token you just created
```

## The loop, with curl

Every answer is JSON shaped `{ "data": ..., "error": null, "meta": ... }`; on a problem `data` is `null` and `error`
says what is wrong.

**1. Which projects are there, and what is deployed where?**

```
curl -s -H "Authorization: Bearer $TOKEN" $HEYSTACK/projects
curl -s -H "Authorization: Bearer $TOKEN" $HEYSTACK/git-status
```

Each project has `gitSyncConfigs`: one entry per repository that deploys to this machine, with its `id`,
`watchBranch` and whether it is `enabled`. The `id` is what you deploy.

**2. Deploy it** (the call waits until the deployment has finished, which can take minutes):

```
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{}' \
  $HEYSTACK/runbook/CONFIG_ID/deploy
```

The answer is `{ "success": true|false, "output": "...", "error": "...", "deploymentRunId": "..." }`.

**3. Read what happened**, step by step, with each step's output:

```
curl -s -H "Authorization: Bearer $TOKEN" $HEYSTACK/runbook/runs/RUN_ID
curl -s -H "Authorization: Bearer $TOKEN" "$HEYSTACK/runbook/runs/recent?limit=5"
```

A run has a `status` (`QUEUED`, `RUNNING`, `SUCCESS`, `FAILED`, `SKIPPED`), the commit, a `note` when it did not run
as asked, and `stepRuns` with each step's status, duration, `stdout` and `stderr`. If the project has an HTTP check
step (the page must answer with the right status and text), a failing check is the step that says so.

**4. Ask whether it is up**: `GET /status/overview` (uptime and incidents) and `GET /health/summary` (what needs
attention).

The full list of routes an agent needs, with what each one needs, is at `GET $HEYSTACK/openapi.json` (OpenAPI 3.1).

## The same with the command line

Install is part of the Heystack package (`heystack`). Point it at your Heystack once, and use `--json` so the output is
easy to read by a program:

```
heystack config set-url http://localhost:3001
export HEYSTACK_TOKEN=hsk_...

heystack list --json
heystack git my-api --json
heystack deploy my-api --json      # waits; the exit code is 0 only when the deployment succeeded
heystack runs my-api --limit 5 --json
```

If a project has more than one Git Sync (for example one per repository), say which with `--repo docs` or
`--branch dev`; `heystack deploy` never guesses.

## Are my backups fine?

Heystack can watch backups and scheduled jobs ([JOBS.md](JOBS.md)). A read-only token is enough to ask:

```
heystack jobs --json               # every job: status, last success, size, which projects it covers
heystack jobs --check              # exit code 1 when a job failed or is overdue
curl -H "Authorization: Bearer $HEYSTACK_TOKEN" http://localhost:3001/jobs
```

`status` is `OK`, `FAILED`, `OVERDUE` or `UNKNOWN` (waiting for a first run), with `statusDetail` in words. Creating or
changing a job, and *check now* (which can run a command), need full access. A script reports a job's result with the
job's own token, not an access token.

## Is the machine fine?

```
heystack host --json       # disks, watched drives, temperature, restart needed, each with a sentence
heystack host --check      # exit code 1 when something needs attention
curl -H "Authorization: Bearer $HEYSTACK_TOKEN" http://localhost:3001/host
```

A read-only token is enough. `severity` is `ok`, `warn`, `crit` or `info` (a restart that is needed, or updates waiting, is
information); `failing` says whether the condition is wrong. Changing what is watched needs full access.

## Let Heystack tell the agent (events)

Instead of asking again and again, add a **webhook** channel in **Settings, General, Notifications**. Heystack then
sends a JSON message when a deployment fails or succeeds and when a project goes down or comes back. Every request
carries the header `X-Heystack-Signature: sha256=<hex>`, the HMAC-SHA256 of the request body made with the channel's
secret, so the receiver can check it came from your Heystack:

```
import { createHmac, timingSafeEqual } from "node:crypto";

function isFromHeystack(rawBody, signatureHeader, secret) {
  const expected = "sha256=" + createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(signatureHeader ?? "");
  return a.length === b.length && timingSafeEqual(a, b);
}
```

The body has `event` (`DEPLOY_SUCCEEDED`, `DEPLOY_FAILED`, `PROJECT_DOWN`, `PROJECT_RECOVERED`), `project`, `title`,
`message`, `link` and `detail`.

## A paragraph to give your agent

> You may use Heystack to deploy and to look at the result. Heystack's address is in `$HEYSTACK` and your token in
> `$TOKEN` (send it as `Authorization: Bearer $TOKEN`). After you have merged a change into the dev branch, find the
> project with `GET /projects`, deploy it with `POST /runbook/{gitSyncConfigId}/deploy` (body `{}`), and read the run
> with `GET /runbook/runs/{id}`. A deployment is finished when `status` is `SUCCESS` or `FAILED`. If it failed, read the
> failing step's `stderr` and fix the cause; do not retry the same thing more than twice. You cannot change Heystack's
> settings, and you must not try to.

## Good to know

- Starting a deployment can run build and install commands on your machine. Allow it only for projects where you are
  comfortable with that, and prefer **Read only** for anything that only needs to look.
- Tokens do not expire by themselves. Revoke a token in **Settings, Account** when the agent is retired; it stops
  working at once.
- Heystack never sends anything outside your network unless you set up a channel or an integration yourself.
