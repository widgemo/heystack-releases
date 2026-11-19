# Using Heystack with an AI agent or a script

Heystack has an API and a command line tool that an AI coding agent (or any script) can use to see what is going on
and to deploy a branch to your own machine, without ever getting your password or the power to change your setup.

## What you can give an agent

An access token has one of three levels. Create it in **Settings, Account, Access tokens** and choose the smallest
level that does the job:

| Level | Can | Cannot |
|---|---|---|
| **Read only** | Look at everything: projects, git state, releases, status, whether each deployment step passed, ask a question | Change anything, start anything. See what a deployment printed or the commands of script steps (see *Extra permissions*) |
| **Read and deploy** | Also start a deployment (or a pull) and refresh checks, for projects that allow it | Change settings, steps or secrets; force-resync, roll back, create tokens |
| **Full access** | Everything you can do | (this is for your own command line, not for an agent) |

### Extra permissions

Under the level you can switch on extra permissions, one by one. They are for an agent that needs to do a little more
than deploy, without handing it full access. **Full access already includes all of them**, and a token made without
extras can do exactly what it could before.

| Permission | Lets the token | Careful |
|---|---|---|
| **Edit projects** (`projects:write`) | Create and change projects, and their stacks, ports and hosting links | It cannot delete a project, and it cannot change whether tokens may deploy a project (that stays your decision) |
| **Edit runbook steps (shell access)** (`runbook:write`) | Add, change, delete and reorder the steps a deployment runs, and change a repository's address | **A script step is a shell command that runs on your machine.** A token with this can run any command there. Give it only to a tool you would trust with a login on that machine |
| **See deployment output and script commands** (`logs:read`) | Read what each deployment step printed (`stdout`, `stderr`) and the commands of script steps | Build output can contain secrets. Without it, a token still sees whether each step passed, how long it took and its exit code |

A token without `logs:read` gets runs and steps with the output and commands left out. Before this permission existed,
**Read only** and **Read and deploy** tokens could see them: if an agent needs the output, create a new token with **See
deployment output and script commands** ticked and replace the old one.

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

**3. Read what happened**, step by step (each step's output too, if the token has *See deployment output and script commands*):

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

## The public API (`/api/v1`)

A calmer, versioned alternative to the routes above for runs. It needs sign-in to be switched on (`HEYSTACK_AUTH=on`, see
the installation guide); otherwise `/api/v1` answers `404` and the log says why. Every answer is
`{ "data": ..., "error": null, "meta": ... }`; when something is wrong `error` is one sentence and a rejected input
lists its problems in `meta.issues`. Ids are opaque strings: copy them, do not parse them.

```
# runs of one project that failed, newest first (read access is enough)
curl -s -H "Authorization: Bearer $TOKEN" "$HEYSTACK/api/v1/runs?projectId=PROJECT_ID&status=FAILED&pageSize=5"

# one run with how each step went (no output)
curl -s -H "Authorization: Bearer $TOKEN" $HEYSTACK/api/v1/runs/RUN_ID

# what the steps printed: needs the "See deployment output and script commands" permission
curl -s -H "Authorization: Bearer $TOKEN" "$HEYSTACK/api/v1/runs/RUN_ID/logs?tail=40"
curl -s -H "Authorization: Bearer $TOKEN" "$HEYSTACK/api/v1/runs/RUN_ID/logs?stepId=STEP_ID"

# start a run of a runbook (deploy access, and the project must allow it); send {}
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{}' \
  $HEYSTACK/api/v1/runbooks/RUNBOOK_ID/runs
```

A **runbook** is what deploys one repository of a project: its id is `gitSyncConfigs[].id` in `GET /projects`. Starting a
run answers `202` at once, for example `{ "runId": "...", "outcome": "started", "statusUrl": "/api/v1/runs/..." }`;
poll `statusUrl` until `status` is `SUCCESS`, `FAILED` or `SKIPPED`. `outcome` says how it was queued: `started`,
`waiting` (another run of the same folder is going; yours follows) or `replaced-waiting` (a run that was still waiting
was replaced by yours and ended `SKIPPED`). Only the newest waiting request runs, exactly as for pushes. A runbook with
no steps still gives you a run (it is a plain `git pull`).

### Projects and runbooks

```
# find a project and the runbook to start (needs read access)
curl -s -H "Authorization: Bearer $TOKEN" "$HEYSTACK/api/v1/projects?q=myapp"
# -> data[0].runbooks = [{ "id": "RUNBOOK_ID", "repository": "...", "watchBranch": "dev", "enabled": true, "stepCount": 4 }]

# the steps of that runbook, in order
curl -s -H "Authorization: Bearer $TOKEN" $HEYSTACK/api/v1/runbooks/RUNBOOK_ID

# change a project (needs the "Edit projects" permission); send only what changes, null clears a field
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"description":"New text","pinned":true}' $HEYSTACK/api/v1/projects/PROJECT_ID

# add a health check last, move steps, remove one (needs "Edit runbook steps (shell access)")
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"HEALTH_CHECK","label":"Site answers","healthCheckUrl":"http://localhost:3000/health"}' \
  $HEYSTACK/api/v1/runbooks/RUNBOOK_ID/steps
curl -s -X PUT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"steps":["STEP_ID_2","STEP_ID_1","STEP_ID_3"]}' $HEYSTACK/api/v1/runbooks/RUNBOOK_ID/steps/order
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" $HEYSTACK/api/v1/runbooks/RUNBOOK_ID/steps/STEP_ID
```

Mind these:

- **Editing steps is shell access.** A `SCRIPT` step is a command that runs on your machine when the runbook runs.
  Give **Edit runbook steps** only to an agent you trust like a person with a login there.
- A token without **See deployment output and script commands** sees the steps but not their commands, also in the
  answer to a call that changed them.
- Send only the fields you know: an unknown field (for example `sourcePath`, which cannot be set through the API) is
  a `400`, so a typo does not silently do nothing. `PUT .../steps/order` needs every step id exactly once.
- A runbook's `enabled` only says whether a push or polling starts a run by itself. You can start a run of a switched-off
  runbook, just as the **Deploy Now** button can.
- Whether tokens may deploy a project (`allowAgentDeploy`) cannot be changed by an agent token, even one that can edit
  projects: that stays your decision.

If the run belongs to Heystack itself, it restarts Heystack while it runs. The `202` has already been sent and the run
id still works afterwards, so keep polling and ignore a few failed requests. The run list shows who started a run:
`trigger` is `API` and `triggeredBy` the token's name.

Paging: `page` from 1 and `pageSize` from 1 to 100 (default 20); `meta` gives `total`, `page`, `pageSize` and `pages`.
A value outside the limits is a `400`.

## Claude Code and other MCP clients (`/mcp`)

Heystack speaks the Model Context Protocol, so an agent can use it as a set of tools instead of writing `curl` calls.
It needs sign-in to be switched on (`HEYSTACK_AUTH=on`); otherwise `/mcp` answers `404` and the log says why.

```
claude mcp add --transport http heystack https://heystack.example.com/mcp \
  --header "Authorization: Bearer hsk_..."
```

Use the address Heystack answers at (`http://localhost:3001` on the same machine) and a token from **Settings,
Account**. Check with `claude mcp list`, or ask the agent "which Heystack tools do you have?".

**The agent only sees what its token allows.** Create the token for the job:

| The agent should | Token | Tools it gets |
|---|---|---|
| Look at projects, runbooks and runs | **Read only** | `list_projects`, `get_project`, `get_runbook`, `get_run`, `list_runs` |
| Ship a branch and see why it failed | **Read and deploy** + *See deployment output and script commands* | the above, `start_run`, `get_run_logs` |
| Also keep project records up to date | add *Edit projects* | `create_project`, `update_project` |
| Also change what a deployment runs | add *Edit runbook steps (shell access)* | `add_runbook_step`, `update_runbook_step`, `delete_runbook_step`, `reorder_runbook_steps` |

The last row is **shell access** to your machine: a script step is a command Heystack runs there. Give it only to an
agent you would trust with a login. Deploying still needs the project to allow it (**Settings** tab of the project, or
**Settings, General, Agent access**), and whether tokens may deploy a project is never something an agent can change.

A typical deployment, as the agent does it: `list_projects` to find the runbook id, `start_run` with that id (answers at
once), then `get_run` with `wait_seconds: 60` until `finished` is true, then `get_run_logs` if it failed. If the run is
Heystack's own deployment, Heystack restarts while it runs and the waiting request may fail: ask again a few seconds
later. When a tool is refused the agent gets the same plain sentence as the `/api/v1` routes give, for example which
permission to add.

**Through a Cloudflare Tunnel:** route the path `/mcp` to the API port, and if Cloudflare Access protects the address add
a policy for `/mcp` with the action *Bypass* (Heystack's own token protects it), or give the agent a Cloudflare service
token with two more `--header` flags (`CF-Access-Client-Id` and `CF-Access-Client-Secret`). The installing guide has the
steps.

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

## Are my processes fine?

```
heystack processes --json    # what runs under pm2 and systemd, and what is wrong with each, with a sentence
heystack processes --check   # exit code 1 when a process keeps restarting, points at missing files or has a runaway log
curl -H "Authorization: Bearer $HEYSTACK_TOKEN" http://localhost:3001/processes
```

A read-only token is enough to look. Restarting, stopping or emptying the logs of a pm2 process
(`POST /processes/{name}/restart`, `/stop`, `/flush-logs`) and changing what is watched need full access.

## Is it a good moment to restart?

```
heystack ready --json      # { safe, blockers, restartNeeded }
heystack ready && sudo reboot
curl -H "Authorization: Bearer $HEYSTACK_TOKEN" http://localhost:3001/host/readiness
```

`safe` is false while a deployment is running or queued or Heystack is running a job check; `blockers` says which. The exit
code of `heystack ready` is 1 when it is not safe. Heystack never restarts the machine, and cannot see a backup or another
program that runs outside Heystack. A read-only token is enough.

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
  comfortable with that, and prefer **Read only** for anything that only needs to look. Think twice before ticking
  **Edit runbook steps (shell access)**.
- Tokens do not expire by themselves. Revoke a token in **Settings, Account** when the agent is retired; it stops
  working at once.
- Heystack never sends anything outside your network unless you set up a channel or an integration yourself.
