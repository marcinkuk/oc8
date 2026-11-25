# Telegram approvals: how it works and how to run it

Covers the `telegram_approvals` capa: where the bot token lives, how messages
flow in both directions, why outbound HTTP is written the way it is, and how
to deploy and debug it on a live host. Written for whoever maintains it next —
human or agent.

---

## 1. The moving parts

| Piece | Where | What it does |
|---|---|---|
| Plugin | `capas/telegram_approvals/` | Contributes a real `ApprovalChannel` (`channel_id = "telegram"`). Code-carrying, `trust = first_party`, tenant-enabled. |
| Credential type | `capas/telegram_approvals/credential_types/telegram_bot.toml` | One password field, `bot_token`; validated via `channel.setup:validate` (`getMe`) when the credential is created/tested. |
| Secret | tenant secret store, ref `credential/<credential-id>/bot_token` | Encrypted at rest. The token is never in `plugin.toml`, env, or git — `plugin.toml` only carries the reference `telegram/bot_token`. |
| Binding | `approval_channel_binding` | Maps a Telegram sender id (`external_id`) to an oc8 user for one tenant. A button press from an unbound sender is refused. |
| Poll cursor | `channel_poll_cursor` (`last_update_id` per tenant+channel) | `getUpdates` offset. Survives restarts; Postgres owns durability, nothing is cached in the process. |
| Poll loop | `backend/src/oc8/channels/poll.py`, driven by `oc8/triggers/scheduler.py` (`run_scheduler`, 30 s tick) | Each tick polls every runnable tenant's polling channel. `timeout=0` short calls — the tick sets the cadence, not Telegram's long-poll. |
| Outbound guard | `backend/src/oc8/net/outbound.py` | `usable_addresses()` (all addresses, IPv6-first) and `SequentialTransport` (fall through addresses per request). |

Two inbound forms are recognised by `parse_update`: a button press
(`callback_data = "<approval-uuid>:approve|reject"`, ≤64 bytes — Telegram's
hard limit, enforced in the compose step) and a `/start <code>` / bare code
message, which is the personal-binding flow (`bind_from_link` in
`oc8/channels/dispatch.py`). Free text becomes `ChannelFreeText` and runs as a
chat-originated request.

Replies go back **into the chat the message arrived in** (`chat_id` from the
update, not the sender id) via `say()`/`deliver()`; approval notices carry
`✅ Approve` / `❌ Reject` buttons. Copy is English; the old German strings are
gone from shipped code (`i18n/de.po` remains for the translated UI).

Webhook mode exists but is fail-closed: without a configured
`webhook_secret`, `verify_inbound` refuses every webhook call. This deployment
runs **polling only** — no webhook is registered with Telegram
(`getWebhookInfo` empty), which is also why nothing competes for `getUpdates`.

## 2. The token: where it is and how to change it

- It is **per tenant**, not global: a credential of type `telegram_bot`
  inside the tenant (created in the UI "Connect Telegram" form or the
  credential picker). The bot belongs to the tenant so an inbound update is
  unambiguous about whose approval it answers.
- On `oc8.qkey.pro` (tenant AERO-GSERENT): credential "AgserBot",
  secret ref `credential/01a1f3cc-2758-7766-aeb4-c6a75462ca8a/bot_token`,
  `last_test_ok = true`.
- **Rotating a token**: create/re-test a new credential value in the UI; the
  channel picks the credential up on its next build. There is no env variable
  holding the token — do not "fix" it by putting a token in `.env`.
- Anyone with the token can talk to the bot, but only bound senders can move
  an approval: the decision path resolves the sender through the binding and
  fails closed otherwise.

## 3. Why outbound HTTP looks like this (and keep it this way)

The live host at `192.168.1.134` reached `api.telegram.org` unreliably:
the host resolved to two addresses, IPv6 answering ~0% of TCP connects (it
hangs until timeout, not refused) and IPv4 ~50%. httpx pins the first
`getaddrinfo` answer for the life of the client, so the channel looked
"intermittently broken" and failed calls were dropped silently. Three fixes,
in three commits — each solves a layer, don't collapse them:

1. **Try every address** (`usable_addresses` + `SequentialTransport`,
   commit `17a9763`): one failed connect is not a failed host.
2. **Bound the cost of a dead address** (`connect_timeout=3.0` per attempt
   via the per-request `timeout` extension, `on_attempt` callback,
   commit `1766e97`): a hanging connect used to eat the whole 10 s call
   timeout before the next address was tried.
3. **Rank by streak, retry the call** (commits `4ed7bac`, `9756b7c`):
   `_scores[ip]` is a connect streak (+1 success, −1 failure, floor −2);
   best streak first, DNS order on ties, so a *flaky-but-working* address is
   never locked out and a healthy network keeps RFC 8305 order untouched.
   `_post` then retries a call up to `_CALL_ATTEMPTS = 3` — but **only on
   `ConnectError`/`ConnectTimeout`**: the request never left the machine, so
   a retry cannot duplicate a Telegram message. A timeout after the request
   was sent, or Telegram's own `ok: false`, is never retried.

The scoreboard is in-process and deliberately so: it is a guess, and Postgres
holds the truth (cursor, bindings). Losing it on restart costs one slow call.

`SequentialTransport` gives up the DNS-rebinding pinning `PinnedTransport`
provides — that is acceptable only because Telegram is a fixed platform
endpoint, never a URL a person typed. Do not reuse it for user-supplied URLs;
those go through `guard_outbound_url`.

## 4. Deploying (the live Portainer stack)

**Deploy path is Portainer, always.** The stack is a Portainer git-linked compose
stack; `docker compose up` against the checkout creates containers Portainer does
not own and shows up as a second, unmanaged stack in the UI. Everything below goes
through Portainer.

### Stack record

| | |
|---|---|
| name / compose project | `oc8_agser` (Portainer uses the stack name as the project name) |
| id | 39 (endpoint 3 = the local environment) |
| type | 2 — Docker Compose stack |
| git | `https://github.com/marcinkuk/oc8.git` @ `refs/heads/main`, entry point `docker-compose.yml` |
| auto-update | interval `5m` + webhook, scheduler job registered |
| project path | `/data/compose/39` |
| env | 30 pairs stored on the stack record — Portainer writes them into `/data/compose/39/.env` on every deploy |

`git pull` in `/data/compose/39` is not the update path: Portainer re-clones the
repo on deploy (and on the 5-minute poll / webhook). `.env` in the project path is
generated from the stack record, so editing it by hand is lost on the next deploy —
change the **Env pairs on the stack** instead.

### Topology

```
browser → https://oc8.aero-gserent.com (443)
        → VPS 91.233.115.88 Caddy  — TLS terminates here (Let's Encrypt cert)
        → host 192.168.1.134:6441  — stack Caddy, plain HTTP, OC8_DOMAIN empty
        → frontend:3000 / backend:8099
```

`OC8_DOMAIN` stays empty: TLS is the VPS's job. Setting a domain makes the stack
Caddy run its own ACME issuance, which fails (no public 443 on this host).
`oc8.qkey.pro` points at this stack through the VPS with a stale upstream port →
`502` with an empty body, which the UI reports as "Unexpected end of JSON input"
(it parses the empty body). Use `https://oc8.aero-gserent.com`, or fix the qkey
vhost upstream on the VPS.

### Naming convention (one stack per company)

Everything derived comes from the stack name = compose project name. Nothing in
the compose file carries a hardcoded `oc8-` prefix, so the next company is simply
another stack with another name:

| object | rule | this stack |
|---|---|---|
| containers | `<project>-<service>-1` | `oc8_agser-backend-1` |
| volumes | `<project>_<key>` (keys: `pgdata`, `miniodata`, `caddy-data`, `caddy-config`, `ollama`) | `oc8_agser_pgdata` |
| compose network | `<project>_default` | `oc8_agser_default` |
| agent network | `name:` from `OC8_AGENT_RUNTIME_NETWORK`, default `<project>_agents` | `oc8_agser_agents` |
| images | `oc8-backend:latest`, `oc8-frontend:latest` — shared tags, not per-stack | unchanged |

The agent network is the one object that would drift on a rename, because the
backend reads it from an env var instead of from compose — hence the explicit
`networks.agents.name`, with the same value in the stack's Env pairs.

**A rename does not move data.** Compose derives volume names from the project, so
renaming means Postgres and Minio look for brand-new empty volumes and the app
starts from scratch. Rename = new stack + copy step, never just a new name:

```bash
docker volume create oc8_agser_pgdata
docker run --rm -v <old-project>_pgdata:/from -v oc8_agser_pgdata:/to alpine   sh -c 'cp -a /from/. /to/'      # same for miniodata, caddy-data, caddy-config
```

Verify with `du -sh` and file counts on both volumes before deploying, take a
`pg_dump` first, and keep the old volumes until the new stack is confirmed healthy.

### API routes (Portainer CE 2.45.4, upgraded 2026-11-25)

The create route is not `POST /api/stacks` — that returns `405` with an empty
body. From the handler routes:

| action | route |
|---|---|
| create compose stack from git | `POST /api/stacks/create/standalone/repository?endpointId=3` with `Name`, `RepositoryURL`, `RepositoryReferenceName`, `ComposeFile`, `AutoUpdate`, `Env` |
| create from literal file | `POST /api/stacks/create/standalone/string?endpointId=3` with `StackFileContent` |
| list / inspect / delete | `GET /api/stacks?endpointId=3`, `GET/DELETE /api/stacks/{id}?endpointId=3` (add `removeVolumes=false` to keep data) |
| redeploy from git | `PUT /api/stacks/{id}/git/redeploy?endpointId=3` |
| autoupdate + webhook | `PUT /api/stacks/{id}` with `AutoUpdate` |
| git webhook | `POST /api/stacks/webhooks/{webhookID}` (public) |
| auth | `POST /api/auth` → field `jwt` (lowercase), send as `Authorization: Bearer` |

Env pairs must be passed on create: compose interpolates `${OC8_*}` while loading
the file, so a stack created without them fails with `error while interpolating
x-backend-env.OC8_…: required variable … is missing`.

### Updating code

- `capas/` is bind-mounted read-only into backend/scheduler/workers at `/app/capas`.
  Plugin changes: push to `main`, then redeploy the stack (`PUT
  /api/stacks/39/git/redeploy`) or restart the services — no image rebuild.
- Core changes (`backend/src/oc8/...`, `frontend/`) need the image rebuilt, which
  Portainer does during deploy. `ghcr.io`/github.com are intermittently unreachable
  from this host over IPv4 — a failed build is normal, retry the deploy.
- Copies of this runbook live in `/data/oc8-docs/`, outside any project path,
  because a Portainer git redeploy with GitHub unreachable has previously wiped a
  stack's project directory entirely.
- Durability: cursor and bindings live in Postgres, the secret in the secret store —
  redeploys, rebuilds and renames never lose Telegram state.

### Health checks

```bash
docker ps --format '{{.Names}} {{.Status}}' | grep oc8_agser        # 11 up, 3 exited helpers
docker logs --since 10m oc8_agser-scheduler-1 2>&1 | grep -c 'poll failed'
docker exec oc8_agser-postgres-1 psql -U postgres -d oc8 -At -c \
  "SET app.tenant_id='ba599d16-2cae-4027-a487-58de294d75b0'; \
   SELECT (SELECT count(*) FROM credential WHERE credential_type='telegram_bot'), \
          (SELECT count(*) FROM approval_channel_binding), \
          (SELECT last_update_id FROM channel_poll_cursor);"
curl -s -o /dev/null -w '%{http_code}\n' https://oc8.aero-gserent.com/          # 200
curl -s -X POST https://oc8.aero-gserent.com/api/v1/auth/login \
  -H 'content-type: application/json' -d '{"email":"x@y.z","password":"x"}'      # 401 + JSON
```

## 5. Operating it

Health check (one command, on the host):

```bash
SCH=$(docker ps --format "{{.Names}}" | grep -E "scheduler-1$" | head -1)
PG=$(docker ps --format "{{.Names}}" | grep -E "postgres-1$" | head -1)
echo "fails:";  docker logs --since 10m "$SCH" 2>&1 | grep -c "poll failed" || true
echo "ok:";     docker logs --since 10m "$SCH" 2>&1 | grep -c 'getUpdates .*200 OK' || true
docker exec "$PG" psql -U postgres -d oc8 -At \
  -c "SELECT last_update_id, updated_at FROM channel_poll_cursor;"
```

- **Healthy** = successes climbing, failures ~0, cursor advancing. After the
  fix the measured live rate was ~90% first-attempt success with 0 poll
  failures per 8–10 min window (occasional `poll failed` is still possible on
  this link: all addresses down at once for >3 retries — the next tick recovers,
  the cursor is not lost).
- **Stuck cursor** (`updated_at` old): the channel isn't polling. Check the
  scheduler is up, then the credential (`last_test_ok`), then run the live
  probe below.
- **Live probe** (token comes from the secret store in the container; never
  paste it into shell history — read it via the secrets service or a temp
  env):

  ```bash
  docker exec -i "$SCH" python3 - <<'PY'
  import asyncio, importlib.util
  spec = importlib.util.spec_from_file_location(
      "tgch", "/app/capas/telegram_approvals/channel/channel.py")
  m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m)
  async def main():
      ch = m.TelegramChannel(token="<from secret store>")
      print(await ch._post("getMe", {}))
      print("scores:", m._scores)
  asyncio.run(main())
  PY
  ```

  `scores` shows the ranking live: the working address should carry a
  positive streak, the dead one −2. A fresh process starts at `0/0` — that is
  expected; ranking warms up after the first calls.
- **End-to-end**: bind a person by sending the link code (or `/start <code>`)
  in a private chat with the bot; `approval_channel_binding` should gain a
  row with their Telegram id. Raise an approval from any agent flow; the
  notice must arrive in the chat it is answered in.
- **German strings**: shipped copy is English; if German reappears in live
  messages, check `capas/telegram_approvals/i18n/de.po` consumers — do not
  reintroduce hardcoded German in `channel.py`.

## 6. Tests and review touchpoints

- `backend/tests/net/test_outbound.py` — transport fall-through, attempt
  reporting, per-attempt connect budget.
- The ranking/retry semantics were validated by driving the real module
  (`_record`/`_rank` streaks; connect retry; Telegram `ok: false` never
  retried) — keep those invariants if you touch `_post` or `_rank`.
- RLS note for ad-hoc SQL: tenant tables are row-filtered by the
  transaction-local `app.tenant_id` GUC; as `postgres` superuser some tables
  read empty unless set — `channel_poll_cursor` reads fine, approvals do not.
