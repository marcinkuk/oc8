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

Topology (as of 2026-11-21):

```
browser → https://oc8.aero-gserent.com (443)
        → VPS 91.233.115.88 Caddy  — TLS terminated here, Let's Encrypt cert
        → host 192.168.1.134:6441  — stack Caddy, plain HTTP, OC8_DOMAIN empty
        → frontend:3000 / backend:8099
```

- Stack: Portainer stack id **36**, compose checkout `/data/compose/36`,
  project/stack name **`oc8_agser`** (`docker-compose.yml` `name: oc8_agser`;
  Portainer uses the stack name as the project name, so the Portainer stack must
  carry the same name). Container names follow the project name
  (`oc8_agser-scheduler-1` after a redeploy; the pre-rename names were
  `oc8-scheduler-1` etc.).
- **Volume long names are pinned** in `docker-compose.yml` to the original
  `oc8_oc8-pgdata` / `oc8_oc8-miniodata` / caddy volumes. Without that pin a
  project rename would derive fresh `oc8_agser_*` volumes and Postgres/Minio
  would silently start empty. Keep those `name:` lines.
- `.env` in `/data/compose/36`: `OC8_HTTP_PORT=6441`, `OC8_HTTPS_PORT=443`
  (unused — no `OC8_DOMAIN`, so stack Caddy never binds TLS),
  `OC8_DOMAIN=` (empty on purpose: TLS is the VPS's job; setting a domain here
  makes the stack Caddy try its own ACME issuance and fail),
  `OC8_FRONTEND_BASE_URL=https://oc8.aero-gserent.com`,
  `OC8_OAUTH_REDIRECT_BASE_URL=https://oc8.aero-gserent.com`.
  **Portainer rewrites `.env` from the stack's Env pairs on every redeploy** —
  change values in the Portainer stack (PUT `/api/stacks/36` with `EnvPairs`),
  not only in the file, or the redeploy silently reverts them.
- **`capas/` is bind-mounted read-only** into backend/scheduler/workers at
  `/app/capas`. Plugin code therefore updates with `git pull` +
  `docker compose -p oc8_agser restart backend scheduler worker ingestion-worker`
  — no image rebuild for plugin-only changes. Core changes
  (`backend/src/oc8/...`) need the image rebuilt.
- **Image build**: `docker compose -p oc8_agser build --pull=false backend`
  (compose tags `oc8-backend:latest`). `ghcr.io` and github.com are
  intermittently unreachable from this host over IPv4 — a failed build is
  normal; retry (layer cache makes retries cheap). Portainer's auto-redeploy
  clones the repo itself; if GitHub is down mid-clone it can wipe
  `/data/compose/36` — restore by re-cloning and re-POSTing the stack's Env
  pairs (`POST /api/stacks/36/git`). Runbook copies live outside the checkout
  in `/data/oc8-docs/` for exactly that reason.
- `oc8.qkey.pro` still points at this same stack through the VPS but its
  upstream port is stale → `HTTP 502` with an empty body (the UI shows
  "Unexpected end of JSON input" because it parses that empty body). Use
  `https://oc8.aero-gserent.com`, or fix/retire the qkey vhost on the VPS.
- Durability: cursor and bindings live in Postgres, the secret in the secret
  store — restarting, rebuilding, renaming, or redeploying never loses the
  Telegram connection state. Pre-rename DB dump:
  `/data/oc8-backups/oc8-pre-rename-20261121T103341Z.dump`.

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
