# cartman-server — Endpoint & Usage Reference

`cartman-server` (NestJS + Prisma) is the **authoritative writer** for the Cartman PH platform: all order placement, dispatch, status transitions, ledger writes, OTP, and money reads go through this HTTP API. The Flutter apps call it via `CartmanApiClient` (dio + Supabase bearer token); reads and Realtime subscriptions stay direct-to-Supabase under RLS.

| Environment | Base URL |
|---|---|
| Local dev | `http://localhost:3000` |
| Android emulator → local dev | `http://10.0.2.2:3000` |
| Production (Render) | `https://<cartman-server>.onrender.com` *(placeholder — substitute the deployed Render URL)* |

**Swagger UI:** `/api/docs` — always on in dev; in production only when `SWAGGER_ENABLED=true` (kept available because interim merchant ops run through Swagger until the merchant panel exists).

**Source of truth:** the controllers and services in `cartman-server/src/`. This document reflects branch `admin-endpoints` (tip `4aa30b3`, PRing into `mvp-readiness`). **Code wins on any conflict — update this doc when controllers change.**

---

## Authentication & security

### Identity: Supabase GoTrue (email + password)

cartman-server does **not** mint sessions. Supabase Auth (GoTrue) does; the server only validates the resulting JWT (`SUPABASE_JWT_SECRET`, HS256) and resolves the caller's role from `profiles.role` per request.

**Step 1 — obtain a token** (password grant against the Supabase project):

```bash
curl -s -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"seed-customer@cartman.ph","password":"CartmanDev1!"}'
```

The response contains `access_token` (the JWT you send to cartman-server), `refresh_token`, `expires_in` (~3600 s), and the `user` object. Extract the token, e.g.:

```bash
TOKEN=$(curl -s -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"seed-customer@cartman.ph","password":"CartmanDev1!"}' | jq -r .access_token)
```

**Step 2 — call cartman-server.** Every request carries `Authorization: Bearer <access_token>`:

```bash
curl -s http://localhost:3000/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

### Security rules

- **Never put tokens in URLs, query strings, or logs.** The token goes in the `Authorization` header only. Query strings end up in server logs, proxy logs, and browser history.
- **HTTPS only** outside local dev. The production Render deployment terminates TLS; never send a bearer token over plain HTTP to a non-localhost host.
- **Tokens expire** (~1 hour). Refresh via the Supabase SDK (the mobile apps do this automatically) or the refresh-token grant: `POST {SUPABASE_URL}/auth/v1/token?grant_type=refresh_token` with `{"refresh_token": "..."}` and the same `apikey` header.
- **The anon key is client-safe** — it only grants what RLS policies allow. **The service-role key is server-only** (used by cartman-server for Storage writes) and must NEVER ship in any client, dashboard bundle, or repo.
- **`@Public` routes are the only tokenless surface.** Enumerated exhaustively:
  - `GET /` — liveness (returns a hello string)
  - `GET /health` — liveness (`{"status":"ok"}`)
  - `POST /webhooks/supabase` — guarded by the `x-supabase-webhook-secret` header instead of a JWT (rejects 401 if the header is missing/wrong, or if `SUPABASE_WEBHOOK_SECRET` is unset server-side)
  - `POST /auth/send-email-otp` — **dormant** (email verification is deferred scope)
  - `POST /auth/verify-email-otp` — **dormant**

  Everything else requires a valid Supabase JWT — `JwtAuthGuard` is registered globally (`app.module.ts`), so an unlisted route without a token is a 401 by default, not an accidental hole.

### Role model

`profiles.role` ∈ `customer` | `rider` | `merchant` | `admin`. `JwtStrategy` looks the role up from the database on **every request** (not from the token claims), so a role change takes effect on the next call without re-login. Guard order is JwtAuthGuard → RolesGuard:

- **401** — missing, expired, or invalid token; or no `profiles` row for the token's `sub`.
- **403** — valid token, wrong role for the route's `@Roles(...)` (also used for the rider feed's duty/wallet gates — see the orders table).

### Error shape

All errors are JSON via the global exception filter + `ValidationPipe`:

```json
{ "statusCode": 400, "message": "Order must contain items" }
```

`message` is a string, or a **string array** for validation failures (one entry per failed constraint; validation errors also include `"error": "Bad Request"`). Unhandled exceptions are logged server-side and returned as a generic `{"statusCode": 500, "message": "Internal server error"}` — never a stack trace. Note: `ValidationPipe({whitelist: true})` silently **strips** unknown body fields rather than rejecting them.

### Rate limits

- **Global:** 100 requests/min per IP (`ThrottlerModule`; `trust proxy` is set so Render's `X-Forwarded-For` gives real client IPs).
- **Phone OTP routes** (`POST /auth/send-otp`, `POST /auth/verify-otp`): 10 requests/min per IP, plus a DB-backed limit of **3 codes per 15 min per phone number** inside `OtpService`.
- Breaching either returns **429**.

### Money conventions

- All money values are **integer centavos** (₱1.00 = 100). No floats, ever.
- Most routes return money as JSON **Numbers** (`serializeOrder` / `serializeProfile` / admin serializers convert BigInt → Number at the response boundary).
- **Ledger routes return string centavos**: `amount` and `id` are strings on `POST /ledger/transactions`, `GET /ledger/me/transactions`, `GET /admin/ledger/transactions`, the `balance` field of `GET /ledger/me/wallet`, and the `ledger_entries[]` of `GET /admin/orders/:id`. Called out per table below.
- Ledger is **append-only**; `amount` is always positive — sign is inferred from `type` (credits: `credit_remittance`, `credit_delivery_reward`; debits: `debit_cod_order`, `debit_commission`). Wallet balance is always **derived** (never stored); lockout floor is net cash ≤ −₱2,000 (−200 000 centavos).

### Pagination (admin lists)

All `GET /admin/*` list endpoints share the envelope:

```json
{ "items": [...], "total": 123, "limit": 20, "offset": 0 }
```

Query params `limit` (1–100, default 20) and `offset` (≥0, default 0).

---

## Endpoints

Roles legend: **any** = any authenticated user (no `@Roles`). "Consumed by" values: customer app / rider app / dashboard (planned) / Swagger ops / Supabase webhook / — dormant.

### Root (liveness)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `GET /` | public | Liveness hello | — | `"Hello World!"` string | uptime checks |
| `GET /health` | public | Health check | — | `{"status":"ok"}` | uptime checks / Render |

```bash
curl -s http://localhost:3000/health
```

### Auth (`/auth`)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /auth/send-otp` | any | Send 6-digit SMS OTP via Semaphore. Throttled 10/min/IP + 3/15min/phone. | `{phone}` — PH mobile, normalized to `+639XXXXXXXXX` (accepts `09…`, `639…`, `+639…`) | `{message}`; **`OTP_DEV_MODE=true` only:** also `{code}` (echoes the OTP instead of sending SMS — dev only, never in prod) | customer app |
| `POST /auth/verify-otp` | any | Verify OTP; sets `profiles.phone_verified=true` + saves `phone`. Max 5 attempts/code, 10-min expiry. Throttled 10/min/IP. | `{phone, code}` (code = 6 chars) | `{verified: true}` (errors are 400: invalid/expired/too many attempts) | customer app |
| `POST /auth/send-email-otp` | public | **Dormant** — email verification is deferred scope | `{email}` | `{message}` (+`code` in dev mode) | — dormant |
| `POST /auth/verify-email-otp` | public | **Dormant** | `{email, code}` | `{verified: true}` | — dormant |
| `GET /auth/me` | any | Current user's profile incl. nested `rider` row (or `null`) | — | profiles row: `{id, full_name, phone, phone_verified, role, avatar_url, membership_tier, cart_points, wallet_centavos, created_at, rider}` (points/wallet as Numbers) | customer + rider apps |

```bash
curl -s -X POST http://localhost:3000/auth/send-otp \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"phone":"09171234567"}'
```

### Profiles (`/profiles`)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `GET /profiles/me` | any | Own profile (same wire shape as `/auth/me`, minus the `rider` include) | — | profiles row (404 if no row) | customer + rider apps |
| `GET /profiles/me/addresses` | any | Saved addresses, newest first | — | `[{id, user_id, label, text_address, lat, lng, is_default, created_at}]` | customer app |
| `POST /profiles/me/addresses` | any | Create address. `is_default: true` atomically unsets other defaults. | `{label, text_address, lat, lng, is_default?}` | created address row | customer app *(screen buttons not yet wired)* |
| `PUT /profiles/me/addresses/:id` | any | Partial update (all fields optional; same default-swap atomicity) | `{label?, text_address?, lat?, lng?, is_default?}` | updated address row | customer app *(not yet wired)* |
| `DELETE /profiles/me/addresses/:id` | any | Delete own address | — | deleted address row | customer app *(not yet wired)* |
| `POST /profiles/me/device-tokens` | any | Upsert FCM device token (unique per user+token) | `{token, platform?}` (`platform` ∈ android\|ios, defaults `android`) | device_tokens row | customer + rider apps |
| `DELETE /profiles/me/device-tokens/:token` | any | Remove own registration(s) for a token (sign-out path) | — | `{count}` | customer + rider apps |

```bash
curl -s -X POST http://localhost:3000/profiles/me/addresses \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"label":"Home","text_address":"San Jose, Antique","lat":10.7327,"lng":121.9418,"is_default":true}'
```

### Orders (`/orders`)

Order rows on the wire: money fields (`subtotal`, `delivery_fee`, `discount`, `total`, `delivery_payout`, `order_items[].unit_price`) as **Number centavos**; plus `id`, `ref_no` (`CMN-000123`), `customer_id`, `assigned_rider_id`, `type`, `status`, `merchant_id`, `merchant_name`, `payment` (default `cod`), pickup/dropoff lat/lng/text, `errand_payload`, `notes`, `recipient_name`, `recipient_phone`, timestamps.

**Customer routes:**

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /orders/food` | customer | Place food order — born `pending`; prices read server-side from `menu_items` (out-of-stock rejected) | `{merchant_id, items: [{menu_item_id, qty, modifiers?}]}` | order row incl. `order_items` | customer app |
| `POST /orders/grocery` | customer | Place grocery order — same body/flow as food | same as food | order row incl. `order_items` | customer app |
| `POST /orders/errand` | customer | Place Pabili errand — born `ready_for_pickup` (no merchant prep); flat ₱100 fee | `{errand_payload?, notes?}` (mobile folds store/items/budget into `notes`) | order row | customer app |
| `POST /orders/courier` | customer | Place pickup & delivery — born `ready_for_pickup`; haversine fare ₱40 base + ₱12/km, min ₱50 | `{pickup_lat, pickup_lng, pickup_text?, dropoff_lat, dropoff_lng, dropoff_text?, notes?, recipient_name?, recipient_phone?}` | order row | customer app |
| `POST /orders/ride` | customer | Place Pasakay ride — born `ready_for_pickup`; same fare curve as courier (placeholder rate) | same as courier minus recipient fields (ignored if sent) | order row | customer app |
| `GET /orders/history` | customer | Own orders, newest first, last 20 | — | `[order row incl. merchant {name, image_url}, order_items, order_stops]` | customer app |
| `PATCH /orders/:id/cancel` | customer | Cancel own order — only while **`pending` or `ready_for_pickup` AND unassigned** (race-safe conditional update) | — | canceled order row (400 `"Order can no longer be cancelled"` otherwise) | customer app |

**Merchant/ops routes** (interim: driven via Swagger until the merchant panel exists):

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `PATCH /orders/:id/accept` | merchant, admin | Advance to `preparing` (food/grocery prep start) | — | order row | Swagger ops |
| `PATCH /orders/:id/ready` | merchant, admin | Advance to `ready_for_pickup` (enters rider feed pool) | — | order row | Swagger ops |

**Rider routes:**

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `GET /orders/feed` | rider | Weighted ranked feed of unassigned `ready_for_pickup` orders (top 20 of oldest-50 pool). Scoring: age 0.4 + proximity 0.3 + payout 0.2 + type 0.1, env-tunable via `FEED_W_*`. Rider position = freshest telemetry ≤10 min old, else the query params. **403 gates:** off-duty → `"Go on duty to see orders"`; net cash ≤ −₱2,000 → `"Wallet locked — remit cash before accepting orders"`. | query: `lat?`, `lng?` (floats) | `[{order, score, distance_km}]` (`distance_km` null without a position) | rider app |
| `PATCH /orders/:id/claim` | rider | **Atomic claim** (first-writer-wins conditional update) → `accepted`. Guards (400): on-duty, wallet lock, queue depth (max 2 active), already-claimed. | — | order row + **`queued: boolean`** (true = rider already had an active order; this one queues behind it) | rider app |
| `PATCH /orders/:id/status` | rider | Advance own order one legal step: `accepted → arrived_at_merchant → picked_up → in_transit → delivered` (no skipping; 400 `"Illegal status transition"`). On `delivered`: idempotently credits `credit_delivery_reward` (delivery_payout) + debits `debit_cod_order` (total, COD) in one transaction. | `{status}` ∈ accepted\|arrived_at_merchant\|picked_up\|in_transit\|delivered | order row | rider app |

```bash
curl -s -X PATCH "http://localhost:3000/orders/$ORDER_ID/claim" \
  -H "Authorization: Bearer $RIDER_TOKEN"
```

### Riders (`/riders`)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /riders` | rider, admin | Create the caller's `riders` row (id = caller's user id) | `{vehicle?, plate_no?}` (both default `""`) | riders row | Swagger ops *(no mobile call site — riders provisioned via seed/signup trigger)* |
| `PATCH /riders/:id/activate` | admin | Force a rider's duty flag (ops override) | `{is_active}` | riders row | Swagger ops / dashboard (planned) |
| `PATCH /riders/me/duty` | rider | Toggle own on-duty flag | `{is_active}` | riders row | rider app |
| `POST /riders/me/telemetry` | rider | Batched GPS fixes (background location flush) | `{locations: [{lat, lng, order_id?, recorded_at?}]}` (`recorded_at` ISO 8601, defaults server-side) | `{count}` (rows inserted) | rider app |
| `GET /riders/me/history` | rider | Own **delivered** orders, newest first, last 20 | — | `[order row incl. order_items, order_stops]` | rider app |

```bash
curl -s -X POST http://localhost:3000/riders/me/telemetry \
  -H "Authorization: Bearer $RIDER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"locations":[{"lat":10.7327,"lng":121.9418,"recorded_at":"2026-07-08T09:30:00.000Z"}]}'
```

### Merchants (`/merchants`)

Money fields (`delivery_fee`, `min_order`, `price`) are centavos. Mobile **browsing** of merchants/menus is direct-to-Supabase (RLS) — these routes are the write path.

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /merchants` | merchant, admin | Create merchant | `{name, category?, delivery_fee?, min_order?, eta_minutes?, is_open?, image_url?, promo_text?}` | merchants row | Swagger ops |
| `PATCH /merchants/:id/status` | admin | Open/close a merchant | `{is_open}` | merchants row | Swagger ops / dashboard (planned) |
| `POST /merchants/:id/menu-items` | merchant, admin | Add menu item | `{name, price?, category?, in_stock?, description?, modifiers?}` | menu_items row | Swagger ops |
| `GET /merchants/:id/menu-items` | any | List a merchant's menu items | — | `[menu_items row]` | Swagger ops *(apps read menus direct from Supabase)* |
| `PUT /merchants/:id/menu-items/:itemId` | merchant, admin | Update menu item (all fields optional / partial) | same fields as create, all optional | menu_items row | Swagger ops |
| `PATCH /merchants/:id/menu-items/:itemId/stock` | merchant, admin | Toggle stock flag | `{in_stock}` | menu_items row | Swagger ops |
| `DELETE /merchants/:id/menu-items/:itemId` | merchant, admin | Delete menu item | — | deleted menu_items row | Swagger ops |

```bash
curl -s -X POST "http://localhost:3000/merchants/$MERCHANT_ID/menu-items" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Chicken Inasal","price":15000,"category":"Mains","in_stock":true}'
```

### Ledger (`/ledger`) — string centavos

**All routes in this table return `amount`/`id` (and `balance`) as strings**, unlike the Number convention elsewhere.

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /ledger/transactions` | admin | Manual wallet entry (e.g. record a cash remittance). `amount` = **positive** centavos; sign inferred from `type`. | `{rider_id, type, amount, order_id?}` — `type` ∈ credit_remittance\|debit_cod_order\|credit_delivery_reward\|debit_commission | txn row with `id`, `amount` as **strings** | Swagger ops / dashboard (planned) |
| `GET /ledger/me/wallet` | rider | Derived net-cash balance + lock flag | — | `{balance: "<signed centavos string>", locked: boolean}` (locked at ≤ −200000) | rider app |
| `GET /ledger/me/transactions` | rider | Own wallet transactions, newest first, last 50 | — | `[{id: string, rider_id, type, amount: string, order_id, created_at}]` | rider app |
| `GET /ledger/me/earnings-today` | rider | Today's `credit_delivery_reward` sum (Asia/Manila calendar day) | — | `{earnings_today: <Number centavos>}` *(Number — computed sum, not a ledger row)* | rider app |

```bash
curl -s -X POST http://localhost:3000/ledger/transactions \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"rider_id":"'$RIDER_ID'","type":"credit_remittance","amount":50000}'
```

### Storage (`/storage`)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /storage/upload` | admin, merchant, customer, rider | Upload a file to Supabase Storage (server uses the service-role key; upsert on) | **multipart/form-data**: file field **`file`** (required, 400 without it), text fields `bucket` (default `cartman-assets`), `path` (default `<timestamp>-<originalname>`) | `{path, publicUrl}` | customer + rider apps *(avatar/POD plumbing — UI not wired yet)* |

```bash
curl -s -X POST http://localhost:3000/storage/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@avatar.jpg" \
  -F "bucket=cartman-assets" \
  -F "path=avatars/$USER_ID.jpg"
```

### Webhooks (`/webhooks`)

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `POST /webhooks/supabase` | public + secret header | Supabase Database Webhook for `orders` changes → FCM push fan-out (**currently a logging stub** — no real push is sent). Requires header `x-supabase-webhook-secret: <SUPABASE_WEBHOOK_SECRET>`; 401 if wrong, missing, or the env var is unset (endpoint disabled until configured). | Supabase webhook payload `{type, record, old_record, ...}` | `{success: true}` | Supabase webhook |

```bash
curl -s -X POST http://localhost:3000/webhooks/supabase \
  -H "x-supabase-webhook-secret: $SUPABASE_WEBHOOK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"UPDATE","record":{"id":"...","status":"accepted","customer_id":"..."},"old_record":{"status":"ready_for_pickup"}}'
```

### Admin (`/admin`) — all admin-only, all on branch `admin-endpoints`

Class-level `@Roles('admin')`. Lists use the `{items, total, limit, offset}` envelope (limit 1–100, default 20; offset default 0).

| Method & path | Roles | Purpose | Request | Response | Consumed by |
|---|---|---|---|---|---|
| `GET /admin/stats` | admin | Ops dashboard roll-up ("today" = Asia/Manila) | — | `{orders: {active, unassigned, delivered_today, canceled_today}, fleet: {on_duty, total}, cod_today: {expected, remitted}, remittance: {riders_owing, outstanding_total}, avg_delivery_secs_today: number\|null}` (money as Numbers) | dashboard (planned) |
| `GET /admin/orders` | admin | Filterable order list, newest first | query: `status` (comma-separated `order_status` list), `type` (one `order_type`), `rider_id`, `customer_id`, `merchant_id` (UUIDs), `limit`, `offset` | envelope; items = order rows incl. `customer: {id, full_name, phone}`, `rider: {id, full_name, phone}` | dashboard (planned) |
| `GET /admin/orders/:id` | admin | Full order detail | — | order row + `order_items`, `order_stops`, `customer`, `rider` (+ `vehicle`, `plate_no`, `rating` when assigned), `last_location: {lat, lng, recorded_at}\|null`, `ledger_entries: [{id: string, type, amount: string, created_at}]` | dashboard (planned) |
| `PATCH /admin/orders/:id/cancel` | admin | Cancel any **pre-terminal** order (any status except `delivered`/`canceled`; race-safe). 404 unknown id; 400 `"Order can no longer be canceled"`. | — | canceled order row | dashboard (planned) |
| `PATCH /admin/orders/:id/reassign` | admin | Reassign to a rider — legal only from `ready_for_pickup`/`accepted`/`arrived_at_merchant` (pre-pickup), resets status to `accepted`. Mirrors claim guards on the target rider (400): must exist + be on duty, not wallet-locked, queue not full (max 2 active), not already assigned this order. | `{rider_id}` (UUID) | updated order row | dashboard (planned) |
| `GET /admin/riders` | admin | Fleet list with computed per-rider fields (batched queries — no N+1) | query: `on_duty` (`true`/`false`), `limit`, `offset` | envelope; items: `{id, full_name, phone, is_active, vehicle, plate_no, rating, acceptance_rate, total_trips, active_orders, net_cash (Number, signed), wallet_locked, last_location: {lat, lng, recorded_at}\|null}` | dashboard (planned) |
| `GET /admin/merchants` | admin | Merchant list + active (non-terminal) order counts | query: `is_open` (`true`/`false`), `limit`, `offset` | envelope; items = merchants row (money as Numbers) + `active_orders` | dashboard (planned) |
| `GET /admin/ledger/transactions` | admin | Cross-rider ledger list, newest first | query: `rider_id`, `order_id` (UUIDs), `type` (one `wallet_txn_type`), `limit`, `offset` | envelope; items: `{id: string, rider_id, rider_name, type, amount: string, order_id, created_at}` — **string centavos** (ledger convention) | dashboard (planned) |

```bash
curl -s "http://localhost:3000/admin/orders?status=ready_for_pickup,accepted&type=food&limit=50" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Authority & status notes

- **Code wins.** This reference is derived from the controllers/DTOs/services in `cartman-server/src/`; if it disagrees with the code, the code is right — fix this doc in the same change that alters a controller.
- **Branch status:** the `/admin/*` routes (and the global-guard ordering fix they rely on) live on `admin-endpoints` (tip `4aa30b3`), PRing into `mvp-readiness`. Everything else is on `mvp-readiness`.
- **Deferred / not implemented:** incidents, commissions (`debit_commission` exists in the enum but nothing writes it), merchant panel (ops via Swagger), admin dashboard UI (endpoints exist; the Next.js prototype is planned), real FCM fan-out (`webhooks.service.ts` is a logging stub; `firebase_messaging` not in the apps), `multi_stop` orders (enum value exists; no endpoint, UI entry hidden), email verification (email-OTP routes dormant).

**Route count: 52** (root 2, auth 5, profiles 7, orders 12, riders 5, merchants 7, ledger 4, storage 1, webhooks 1, admin 8).
