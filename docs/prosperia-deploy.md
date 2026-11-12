# Prosperia deploy runbook (OpenReply on Railway)

Replaces ManyChat's Instagram comment-to-DM automation. Fork of `diwenne/openreply`, deployed from this fork only.

## Pinned upstream

- Upstream: `diwenne/openreply`
- Base SHA: `c78b2d2f68c9764ef7a7dde51dfa5bfa99304a09` (recorded 2026-11-12)
- Upstream changes enter only through a reviewed PR in this fork.

## Railway project

- Project: `openreply` (separate from Prosperia's production project; do not add services there).
- Public hostname: `reply.prosperiaapp.com` (Cloudflare DNS, SSL mode Full).
- Postgres: dedicated to OpenReply. Not shared with `wa-expense`.
- Redis: dedicated to OpenReply.

## Services

All three app services build from the root `Dockerfile` (base `node:22-slim`, Node runtime >= 20.19 required by Prisma 7 and Next 16). Each sets its own Railway config-as-code path.

| Service | Config path | Start command | Restart policy | Public domain |
|---|---|---|---|---|
| `web` | `railway/web.json` | `npx prisma migrate deploy && npm start` | ON_FAILURE, max 5 | yes |
| `worker` | `railway/worker.json` | `npm run worker` | ALWAYS | no |
| `cron` | `railway/cron.json` | `sh scripts/cron.sh` | ALWAYS | no |

- Migrations run only on `web`. `worker` and `cron` never run migrations.
- `cron` calls `/api/cron/attach-next-reel` every 5 minutes and `/api/cron/refresh-tokens` plus `/api/cron/snapshot-followers` daily at 05:00 UTC, with header `Authorization: Bearer $CRON_SECRET`. If `CRON_SECRET` is unset, `cron.sh` falls back to `NEXTAUTH_SECRET`.

## Environment variables (names only, values live in the password manager)

Store values under `OpenReply / Railway`. Never commit values.

Legend: `All` = web, worker and cron. `Shared` = project shared variable.

| Variable | Services | Status |
|---|---|---|
| `NEXTAUTH_URL` | All | Set to `https://reply.prosperiaapp.com` |
| `NEXTAUTH_SECRET` | All | Set |
| `CRON_SECRET` | All | Set |
| `ENCRYPTION_KEY` | All | Set. Exactly 64 hex chars, identical on all three |
| `WEBHOOK_VERIFY_TOKEN` | All | Set |
| `DATABASE_URL` | All | `${{Postgres.DATABASE_URL}}` |
| `REDIS_URL` | All | `${{Redis.REDIS_URL}}` (add `?family=0` only if private-network DNS requires it) |
| `ALLOWED_EMAILS` | All | Set. Required. Upstream ships it commented out in `.env.example`. Value: the three addresses listed in the password manager entry, not in this repo |
| `EMAIL_SERVER` | All | PENDING: AWS SES SMTP URL (dedicated IAM SMTP user, region host). Upstream ships it commented out |
| `EMAIL_FROM` | All | `noreply@prosperiaapp.com` (must be SES-verified) |
| `RESEND_API_KEY` | None | Not used. Replaced by `EMAIL_SERVER` |
| `META_GRAPH_API_VERSION` | All | Set (`v25.0`) |
| `INSTAGRAM_APP_ID` | All | PENDING: Meta (Task 7 Step 6) |
| `INSTAGRAM_APP_SECRET` | All | PENDING: Meta (Task 7 Step 6) |
| `FACEBOOK_APP_SECRET` | All | PENDING: Meta (Task 7 Step 6) |
| `CRON_BASE_URL` | cron only | `https://reply.prosperiaapp.com` |

The app boots without the three Meta variables. Webhooks and DM sends do not work until they are set.

## Meta app

- Status: PENDING. Created in Task 7.
- App ID: pending (not a secret, record here once created).
- Connected account: Prosperia's own Instagram account only, as an Instagram tester. No App Review needed for this scope. Other accounts require App Review (see `META_APP_REVIEW.md`).
- Webhook callback: `https://reply.prosperiaapp.com/api/webhook`, fields `comments` and `messages`.
- OAuth redirect: `https://reply.prosperiaapp.com/api/instagram/callback`.
- Legal URLs: `/privacy`, `/terms`, `/data-deletion` exist in the app. Fallback: `https://prosperiaapp.com/privacy` and `/terms` if the app pages are not used.

## Meta limits (cannot be changed)

- One private reply per comment, within 7 days of the comment.
- 750 private replies per hour per account. OpenReply queues the overflow.
- Follow-up DMs only within 24 h of the user's last message.
- Follow-gate fails open: if Meta returns no follow status, the link is sent.

## Campaign inventory

Pending Task 9 Step 1. One row per ManyChat automation: post, keywords, DM text, buttons and URLs, public reply.

## Rollback

Stub. Full procedure is in Task 9 Step 5 of the plan.

1. Deactivate all OpenReply campaigns.
2. Reconnect Instagram in ManyChat, set ManyChat as conversation owner in Business Suite, resume its automations.
3. Record the failure here and debug from `web` and `worker` logs.

ManyChat stays live until the cutover passes. Its subscription is cancelled only after 7 clean days.

## Upgrades

Monthly: `git fetch https://github.com/diwenne/openreply main`, review the diff (especially `worker/`, `prisma/migrations/`, webhook handlers), merge through a PR in this fork, watch the Railway deploy, then rerun the test-post check.
