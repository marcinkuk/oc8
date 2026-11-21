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

### Topology (as of 2026-11-21)

```
browser → https://oc8.aero-gserent.com (443)
        → VPS 91.233.115.88 Caddy  — TLS terminates here (Let's Encrypt cert)
        → host 192.168.1.134:6441  — stack Caddy, plain HTTP, OC8_DOMAIN empty
        → frontend:3000 / backend:8099
```

`OC8_DOMAIN` stays empty on purpose: TLS is the VPS's job. Setting a domain here
makes the stack Caddy run its own ACME issuance, which fails (the host has no
public 443) and takes the proxy down.
`oc8.qkey.pro` still points at this stack through the VPS with a stale upstream
port → `HTTP 502` with an **empty body**; the UI shows "Unexpected end of JSON
input" because it parses that empty body. Use `https://oc8.aero-gserent.com`, or
fix/retire the qkey vhost on the VPS.

### Naming convention (one stack per company)

Everything derived comes from the compose **project name** = the Portainer
**stack name** = `oc8_agser`. Nothing in the compose file carries a hardcoded
`oc8-` prefix, so a second company is a second stack with its own prefix:

| object | rule | this stack |
|---|---|---|
| containers | `<project>-<service>-1` | `oc8_agser-backend-1` |
| volumes | `<project>_<key>` (keys: `pgdata`, `miniodata`, `caddy-data`, `caddy-config`, `ollama`) | `oc8_agser_pgdata` |
| compose network | `<project>_default` | `oc8_agser_default` |
| agent network | `name:` set from `OC8_AGENT_RUNTIME_NETWORK`, default `<project>_agents` | `oc8_agser_agents` |
| images | `oc8-backend:latest`, `oc8-frontend:latest` — shared, **not** per-stack | unchanged |

The agent network is the one object that must not drift with a rename, because
the backend reads it from an env var rather than from compose: hence the explicit
`networks.agents.name: ${OC8_AGENT_RUNTIME_NETWORK:-oc8_agser_agents}` — one
variable drives both.

**A project rename does not move data.** Compose derives volume names from the
project, so `oc8` → `oc8_agser` means Postgres and Minio look for brand-new,
empty volumes and the app silently starts from scratch. Rename therefore needs
the copy step below, not just a redeploy.

### Renaming / migrating an existing stack

```bash
cd /data/compose/36 && git pull        # must carry `name: oc8_agser`
docker compose -p oc8 down
for k in pgdata miniodata caddy-data caddy-config; do
  docker volume create oc8_agser_$k
  docker run --rm -v oc8_oc8-$k:/from -v oc8_agser_$k:/to alpine     sh -c 'cp -a /from/. /to/ 2>/dev/null || true'
done
# the agent network holds only running sandboxes; recreate rather than rename
docker network rm oc8_agents 2>/dev/null || true
docker compose -p oc8_agser up -d --build backend frontend   # images are shared tags
```

Then re-point the agent network env if it was overridden, and check
`docker volume ls | grep oc8_agser` shows non-empty `pgdata`. Keep the old
`oc8_oc8-*` volumes until the new stack is confirmed healthy — they are the
rollback. A logical dump before any of this is cheaper than a restore:
`/data/oc8-backups/oc8-pre-rename-20261121T103341Z.dump` is the pre-rename one.

### Portainer stack 36 — git link vs. manual deploy, and why it matters

Stack 36 **is** git-linked (`Type 2` compose, `GitConfig` =
`https://github.com/marcinkuk/oc8.git` @ `refs/heads/main`, entry point
`docker-compose.yml`, `AutoUpdate` = every 5m + webhook). Two things about it
are worth knowing before relying on it:

- **The updater is not running.** The stack record references `JobID "1"`, but
  no stack-update job exists in the scheduler (`grep` of the bolt DB finds zero),
  so the 5-minute poll never fires. The git link is currently declarative only.
- **The stack's `Env` pairs are stale** and Portainer rewrites
  `/data/compose/36/.env` from them on *every* Portainer-initiated deploy. As of
  2026-11-21 the stored pairs still say `OC8_HTTP_PORT=80`,
  `OC8_FRONTEND_BASE_URL=http://192.168.1.134`,
  `OC8_OAUTH_REDIRECT_BASE_URL=http://192.168.1.134`, plus an unused
  `OC8_IMAGE_TAG=cf243f7…`. A Portainer "Deploy the stack" click therefore
  reverts the port to 80 and the base URLs to the LAN IP — the exact state that
  produced the broken login and the VPS `502`. That is why the current containers
  were deployed by hand (`docker compose -p oc8 up`) from the checkout after
  editing `.env`: it is the only path that keeps the intended port and URLs until
  the stack's Env pairs are corrected.

Fix, and the git link becomes the normal deploy path again — either in the UI
(Duplicate/Edit → rename to `oc8_agser`, set the env pairs, redeploy) or through
the API (`PUT /api/stacks/36` with the corrected `EnvPairs`). Set `Name` and
`Env` together: Portainer uses `Name` as the compose project name, so the name
change *is* the rename, and the volume copy above must happen around it.

- **`capas/` is bind-mounted read-only** into backend/scheduler/workers at
  `/app/capas`. Plugin code updates with `git pull` +
  `docker compose -p oc8_agser restart backend scheduler worker ingestion-worker`
  — no image rebuild. Core changes (`backend/src/oc8/...`) need a rebuild:
  `docker compose -p oc8_agser build --pull=false backend` (compose tags
  `oc8-backend:latest`). `ghcr.io`/github.com are intermittently unreachable
  from this host over IPv4 — a failed build is normal, retries are cheap.
- `/data/compose/36` is Portainer's own clone (7 commits, origin = github);
  `.env` there is untracked and survives `git pull`. Copies of this runbook live
  outside the checkout in `/data/oc8-docs/`, because a Portainer git redeploy
  with GitHub unreachable has previously wiped `/data/compose/36` entirely.
- Durability: cursor and bindings live in Postgres, the secret in the secret
  store — restarts, rebuilds, renames and redeploys never lose Telegram state.

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
