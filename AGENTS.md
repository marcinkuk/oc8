# Agent instructions

Guidance for AI coding agents working in this repository.

- Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a pull request —
  it covers the DCO sign-off, coding guidelines, and PR process.
- Before committing: `cd backend && python -m pytest && python -m ruff check src tests`
  and `cd frontend && npm run lint && npm run test:unit && npm run build`.
- Follow existing patterns in the module you're touching rather than
  introducing a new one — see [docs/contributing/coding-guidelines.md](docs/contributing/coding-guidelines.md).
- Telegram approvals: [docs/TELEGRAM_APPROVALS.md](docs/TELEGRAM_APPROVALS.md)
  covers where the bot token lives, the outbound-transport design (address
  ranking + connect-only retry — do not simplify it away), deploying the live
  Portainer stack, and how to health-check polling.
- Known failures that exist on plain upstream too (don't chase them): backend
  `tests/approvals/test_reads_go_through_the_scoped_repository.py::test_no_module_loads_an_approval_or_a_clarification_outside_the_repo`
  (`agents/retire.py` loads `ApprovalRequest`); frontend `tsc --noEmit` reports
  8 errors (mission-automation, capa-export-wizard); `totp-challenge`
  resend-timer test fails intermittently under full-suite load.
- Upstream is `oc8-ai/oc8`; contributions need Conventional Commits plus a
  `Signed-off-by` DCO line (`git commit -s`).

## Ship a fix here AND upstream

This fork is what the live instance builds from, so every fix needs two steps,
in this order — the PR alone changes nothing for the running tenant.

1. Branch from `upstream/main`, never from this fork's `main`:
   `git fetch upstream && git checkout -b fix/<name> upstream/main`.
2. Commit with Conventional Commits and a DCO sign-off for the human author,
   not the agent default in `git config`:
   `git -c user.name="Marcin Kuk" -c user.email="marcin.kuk@users.noreply.github.com" commit -s`.
3. Push to origin, then `gh pr create --repo oc8-ai/oc8 --base main --head marcinkuk:<branch>`.
   The token has no admin rights on `oc8-ai/oc8`, so you cannot set issue/PR labels.
4. Make it live locally: `git checkout main && git merge --no-ff fix/<name>` and push `main`.
   `--no-ff` keeps the branch commit's SHA identical to the PR head, so when upstream
   merges it, a later `git merge upstream/main` into our `main` adds no duplicate commit
   and no conflict on the same file. Cherry-pick or rebase would fork the SHA and re-conflict.
5. Never rebase or force-push this `main` — it is published and deployed. Rebase only a
   feature branch before its merge (`git rebase upstream/main fix/<name>`).
6. Deploy is `git pull && docker compose up -d --build` (`docs/DEPLOY.md`); a frontend-only
   change needs only `docker compose up -d --build frontend`. Do not restart the live stack
   without the user confirming which Portainer stack serves `oc8.aero-gserent.com`.

`git push` fails with "Invalid username or token" because no credential helper is
configured; `gh auth setup-git` does not help either. Push with
`git push "https://x-access-token:${GITHUB_TOKEN}@github.com/<remote>.git" <branch>`
and keep the token out of the stored remote URLs.

## Portainer stack `oc8_agser` — NEVER detach it from GitHub

The live tenant builds from this fork's `main` through Portainer stack
`oc8_agser` (current id 39, endpoint 3). The stack record MUST stay a
standalone/repository stack: `https://github.com/marcinkuk/oc8.git` @
`refs/heads/main`, entry point `docker-compose.yml`, AutoUpdate interval
`5m`. Pushing a commit to `origin/main` is the only update path — Portainer's
5-minute poll re-clones the repo and rebuilds backend/frontend images itself.

The rule covers this stack only. Git redeploy, stack create/update/delete and
compose operations are legitimate for every other stack and environment — do
not avoid or refuse them there.

Hard rules for this stack:

- Never convert this stack to an App Template / web-editor stack and never
  paste compose content over it. That drops the git link: the stack record
  then shows `FromAppTemplate: true`, `GitConfig: null`, `AutoUpdate: null`
  and the 5-minute poll silently stops forever. This happened twice — once on
  stack 38 (2026-11-23) — and each time the user had to notice the live
  instance stopped updating.
- Never clear or shorten `AutoUpdate`, never delete the stack without the
  user's explicit confirmation.
- Never run `docker compose up/build` against `/data/compose/39` and never
  hand-edit `/data/compose/39/.env` — Portainer re-clones the repo and
  regenerates `.env` from the stack's Env pairs on every deploy.
- After any deploy-related change, verify the link is intact:
  `GET /api/stacks/{id}` must show a non-null `GitConfig` (URL
  `marcinkuk/oc8.git`, ref `refs/heads/main`) and `AutoUpdate.Interval: "5m"`.

`pull_policy: build` on the backend/frontend images is load-bearing: without it Portainer aborts every auto-update deploy at `compose pull` ("pull access denied" for the locally-built images). Do not remove it.

If the record ever shows `GitConfig: null`, restore it (data survives — the
named volumes are not removed): dump the stack's Env pairs, then
`DELETE /api/stacks/{id}?endpointId=3&force=true&removeVolumes=false` (the
delete refuses without `endpointId`), then
`POST /api/stacks/create/standalone/repository?endpointId=3` with `Name`,
`RepositoryURL`, `RepositoryReferenceName`, `ComposeFile`, the same `Env`
pairs, and `AutoUpdate: {"Interval":"5m","CreateWebhook":true}`.
Verify: stack record has GitConfig+AutoUpdate, all 11 containers healthy, and
the running image carries the newest main commit.

## Reading schedules on the live tenant

- An agent's cron schedule is a `Trigger` row, not the free-text `Agent.schedule` column.
  `copilot_get_agent` reports `schedule: ""` for an agent that does have a live trigger —
  an empty `schedule` proves nothing. The authority is `GET /api/v1/agents/{id}/triggers`
  and the "Schedule / trigger" card on the agent page (Configuration tab).
- Each scheduled run's task is `trigger.task_text`, and `open_run_task` titles the board
  task with it — so a run card titled "Scheduled run for X" is a trigger left on the UI's
  placeholder default.
- The MCP connection to oc8 works. Direct REST does not: the member API key returns 401 on
  `/api/v1`, and admin login demands TOTP enrollment (the issued token only carries
  `totp:enroll`). Do not burn turns re-probing it — use MCP, or ask the user for a usable
  token.
