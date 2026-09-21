# VIP Membership Telegram Bot

A production-oriented Telegram bot for managing paid VIP memberships and a
private VIP channel, built with Python 3.12, aiogram 3.x, PostgreSQL,
SQLAlchemy 2.x (async), Alembic, Redis, and APScheduler.

## Features

- Dynamic, database-driven subscription plans (no hardcoded plans)
- Telegram native Payments integration, provider-agnostic architecture
  (`PaymentProvider` abstraction — add Stripe or another provider later
  without touching the rest of the system)
- Idempotent payment confirmation: VIP access is **only** ever granted after
  a verified confirmation from the payment provider (Telegram's
  `successful_payment` update), never from a user's claim. Duplicate charge
  IDs and duplicate invoice payloads are rejected at the database level
  (unique constraints) and at the service level.
- Membership stacking: renewing before expiry adds the new plan's duration
  on top of remaining time instead of discarding it.
- Automatic background expiration worker (checks every N seconds,
  independent of user activity) that expires memberships, revokes VIP
  channel access, and notifies the user.
- Configurable pre-expiration notifications (e.g. 3 days / 24h / 1h before
  expiry), each sent exactly once per subscription per threshold.
- Referral system with self-referral and duplicate-referral protection, and
  configurable VIP-day rewards.
- Promo code system: percentage or fixed discounts, validity windows,
  total/per-user usage limits, plan restrictions.
- Full in-Telegram admin panel: dashboard, user management (search,
  suspend/unsuspend, manual VIP grant/removal), plan management, promo code
  management, broadcast (text/photo/video/document with progress and rate
  limiting), referral leaderboard, dynamic settings, audit log.
- Structured JSON logging with automatic secret redaction.
- Global error handler covering Telegram API errors, network errors, and
  database errors so no single failure crashes the bot.

## Project layout

```
app/
├── main.py                  # Entrypoint: bot, dispatcher, scheduler wiring
├── config/                  # Settings (pydantic) and logging setup
├── database/
│   ├── models/               # SQLAlchemy ORM models
│   ├── repositories/         # Data access layer
│   ├── base.py, session.py
├── services/                 # Business logic (subscriptions, payments, ...)
│   └── payment_providers/    # Payment provider abstraction + Telegram impl
├── bot/
│   ├── handlers/              # aiogram routers (user-facing + admin/)
│   ├── keyboards/              # Inline keyboards
│   ├── middlewares/            # DB session injection, admin auth, throttling
│   ├── filters/, states/
├── workers/                   # Background jobs (expiration, notifications)
└── utils/                     # Datetime, code generation, pagination helpers
alembic/                      # Database migrations
tests/                        # Pytest suite (SQLite in-memory, async)
```

## Requirements

- Docker and Docker Compose (recommended), or Python 3.12+ with a local
  PostgreSQL 16+ and Redis 7+ if running without Docker.
- A Telegram bot token from [@BotFather](https://t.me/BotFather).
- A Telegram Payments provider token, obtained from BotFather under
  `/mybots` → your bot → `Payments` → choose a provider (e.g. Stripe's
  Telegram integration) and connect a **test** provider first.

## Setup

### 1. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set at minimum:

- `BOT_TOKEN` — from BotFather.
- `ADMIN_IDS` — comma-separated Telegram numeric user IDs who should have
  admin access (get your ID from e.g. [@userinfobot](https://t.me/userinfobot)).
- `VIP_CHANNEL_ID` — the numeric ID of your private VIP channel/group
  (starts with `-100...`). **Add the bot to that channel as an admin** with
  at least "Invite Users via Link" and "Ban Users" permissions, so it can
  create invite links and remove expired members.
- `TELEGRAM_PAYMENTS_PROVIDER_TOKEN` — your payment provider token.
- `POSTGRES_*` / `DATABASE_URL` — set a strong password; keep the host as
  `db` if using Docker Compose, or point it at your own PostgreSQL instance
  otherwise.
- `REDIS_URL` — keep as-is for Docker Compose, or point at your Redis
  instance.

### 2. Run with Docker Compose (recommended)

```bash
docker compose up --build
```

This will:
1. Start PostgreSQL and Redis and wait for them to be healthy.
2. Run `alembic upgrade head` via the one-shot `migrate` service to create
   all tables.
3. Start the bot, which begins polling Telegram and starts the background
   expiration/notification scheduler.

Check logs with:

```bash
docker compose logs -f bot
```

### 3. Run locally without Docker (alternative)

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Point DATABASE_URL / REDIS_URL in .env at your local Postgres/Redis
alembic upgrade head
python -m app.main
```

## First steps after startup

1. Open a chat with your bot and send `/start`. Since your Telegram ID is
   in `ADMIN_IDS`, you'll see a "🛠 Admin Panel" button on the main menu.
2. Go to **Admin Panel → Plans → Create Plan** and create your first
   subscription plans (e.g. VIP 7/30/90/365 Days).
3. Optionally create promo codes under **Admin Panel → Promo Codes**.
4. Test a payment: as a regular (non-admin, or any) user, choose
   **💎 Subscribe**, pick a plan, and pay using Telegram's test payment
   card details for your configured test provider. On confirmation, VIP
   access and an invite link are granted automatically.
5. Adjust notification timing or referral rewards under
   **Admin Panel → Settings** at any time — no redeploy needed.

## Running the tests

```bash
pip install -r requirements.txt
pytest
```

The test suite uses an in-memory SQLite database (via `aiosqlite`) so it
runs without any external services. It covers:

- Subscription creation, renewal stacking, expiration, suspend/unsuspend.
- Payment idempotency: re-confirming the same payment is a safe no-op, and
  a provider charge ID reused across two different pending payments is
  rejected.
- Promo code validation (expiry, plan restriction, per-user and total usage
  limits, discount computation).
- Referral abuse prevention (self-referral, duplicate referral) and reward
  granting.
- Datetime formatting utilities.

## Database migrations

To create a new migration after changing models:

```bash
alembic revision --autogenerate -m "describe your change"
alembic upgrade head
```

The initial migration (`alembic/versions/0001_initial.py`) was written by
hand to exactly match the models in `app/database/models/`, so
autogeneration will produce a clean diff for any changes you make on top of
it.

## Security notes

- The bot token, payment provider token, and database credentials all live
  only in `.env` / environment variables — never in source code — and are
  redacted from structured logs.
- VIP access is granted exclusively by `PaymentService.confirm_payment_from_provider`,
  which is only ever called from the `successful_payment` handler
  (`app/bot/handlers/payment.py`). No other code path grants VIP access
  based on payment.
- `pre_checkout_query` is validated against the stored pending payment's
  amount/currency before approval, so a tampered client-side price can
  never be charged.
- All admin-only callbacks and commands are enforced server-side by
  `AdminOnlyMiddleware`, not merely by hiding buttons client-side.
- Every admin action (suspend, VIP grant/removal, plan/promo changes,
  broadcasts, setting changes) is written to the `audit_logs` table.

## Extending to a new payment provider

Implement `app/services/payment_providers/base.py`'s `PaymentProvider`
interface (see `telegram_payments.py` for a reference implementation), then
pass an instance of it into `PaymentService` instead of
`TelegramPaymentsProvider`. No changes to `SubscriptionService`,
repositories, or handlers are required — `PaymentService` is the only
caller of the provider interface.
