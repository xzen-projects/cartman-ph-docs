# System Flows

Process and interaction flows extracted from [ARCHITECTURE.md](../../ARCHITECTURE.md). **Writes route through `cartman-server`** unless noted otherwise.

---

## Flow Index

| Flow | Type | Actors |
|------|------|--------|
| [Customer registration](#customer-registration-email--password) | Onboarding | Customer, Supabase Auth, `cartman-server`, Semaphore |
| [Rider registration](#rider-registration-no-approval-gate) | Onboarding | Rider, Supabase Auth |
| [Merchant registration](#merchant-registration-not-implemented) | Onboarding | **Not implemented** |
| [Food order lifecycle](#food-order-lifecycle) | Core | Customer, `cartman-server`, Ops (Swagger), Rider |
| [COD ID validation](#cod-id-validation-not-implemented) | Core | **Not implemented** |
| [Errand order](#errand-pabili-order) | Core | Customer, `cartman-server`, Rider |
| [Courier order](#courier-order-pickup_delivery) | Core | Customer, `cartman-server`, Rider |
| [Rider order claim](#rider-order-claim-race-condition) | Dispatch | Multiple riders, `cartman-server` |
| [Rider delivery progression](#rider-delivery-status-progression) | Dispatch | Rider, `cartman-server` |
| [Merchant order queue (ops interim)](#merchant-order-queue-ops-interim) | Fulfillment | Ops, `cartman-server` (Swagger) |
| [Admin cancel and reassign](#admin-cancel-and-reassign) | Dispatch | Admin, `cartman-server` (`admin-endpoints`, in progress) |
| [Wallet and lockout](#wallet-and-lockout) | Finance | Rider, `cartman-server`, Admin |
| [Support ticket and override](#support-ticket-and-override-not-implemented) | Admin | **Not implemented** |
| [Realtime subscriptions](#realtime-subscription-flows) | Events | All clients |
| [Push notification](#push-notification-flow) | Events | FCM — stub |

---

## Customer Registration (Email + Password)

**Refs:** C-1.1, C-1.2

```mermaid
sequenceDiagram
  participant C as Customer App
  participant Auth as Supabase Auth
  participant DB as PostgreSQL
  participant Server as cartman-server
  participant SMS as Semaphore SMS

  C->>Auth: signUp(email, password)
  Auth->>DB: create auth.users
  DB->>DB: handle_new_user() trigger creates profiles(role=customer)
  Auth-->>C: session (JWT) — no email verification required
  Note over C,Server: Phone OTP is a COD-checkout gate, not part of signup
  C->>Server: POST /auth/send-otp {phone}
  Server->>SMS: send verification code
  SMS-->>C: SMS delivered
  C->>Server: POST /auth/verify-otp {phone, code}
  Server->>DB: profiles.phone_verified = true
  Server-->>C: verification success
```

**Gates:**
- COD checkout blocked until `phone_verified = true` — signup/login/browsing are not blocked.
- `OTP_DEV_MODE=true` returns the code inline instead of sending SMS.
- Session tokens stored via the Supabase client SDK's secure storage.

---

## Rider Registration (No Approval Gate)

**Refs:** R-4.2

```mermaid
sequenceDiagram
  participant R as Rider App
  participant Auth as Supabase Auth
  participant DB as PostgreSQL

  R->>Auth: signUp(email, password)
  Auth->>DB: create auth.users
  DB->>DB: handle_new_user() trigger creates profiles(role=rider) + riders(is_active=false)
  Auth-->>R: session (JWT)
  R-->>R: unlock on-duty toggle immediately
```

**Feed gate:** `is_active = true` only. **No approval workflow exists** — the `verification_status: pending/approved/rejected` design (with admin document review) was never implemented; there is no `verification_status` column.

---

## Merchant Registration (Not Implemented)

There is no merchant app, no merchant login, and `merchants` has no FK to `auth.users`. The application/document-upload/admin-review flow originally designed here does not exist in code. Merchant rows (name, menu, delivery fee) are inserted directly by ops — seed data today is *Kusina ni Aling Nena* with 3 menu items. See [ARCHITECTURE.md §10.3](../../ARCHITECTURE.md#103-merchant-web-panel).

---

## Food Order Lifecycle

End-to-end from placement to delivery. Writes go through `cartman-server`; realtime broadcast is unchanged (Supabase WAL).

```mermaid
sequenceDiagram
  participant Cust as Customer App
  participant Server as cartman-server
  participant DB as PostgreSQL
  participant RT as Realtime
  participant Ops as Ops (Swagger)
  participant Rider as Rider App
  participant FCM as FCM

  Cust->>Server: POST /orders/food (COD)
  Server->>DB: INSERT order, status=pending
  DB->>RT: WAL event
  RT-->>Cust: stepper shows pending
  Ops->>Server: PATCH /orders/:id/accept
  Server->>DB: UPDATE preparing
  RT-->>Cust: stepper update
  Ops->>Server: PATCH /orders/:id/ready
  Server->>DB: UPDATE ready_for_pickup
  RT-->>Rider: feed change-signal → debounced GET /orders/feed
  Rider->>Server: PATCH /orders/:id/claim
  Server->>DB: conditional updateMany (race-safe)
  RT-->>Cust: rider assigned
  Rider->>Server: PATCH /orders/:id/status (chain to delivered)
  Server->>DB: legal-transition write + ledger credit
  RT-->>Cust: delivered
  Server->>FCM: webhook fan-out attempt
  Note over FCM: logging stub — no device actually receives this yet
```

### Order status state machine

Implemented `order_status` enum uses **`canceled`** (one L), not `cancelled`. Includes admin cancel (any pre-`delivered` status) and admin reassign (pre-pickup only) — see [ARCHITECTURE.md §8](../../ARCHITECTURE.md#8-order-lifecycle) for the full diagram with every admin-cancel edge.

```mermaid
stateDiagram-v2
  [*] --> pending: food_grocery_placed
  [*] --> ready_for_pickup: errand_pickup_delivery_ride_born_ready

  pending --> preparing: ops_accepts_via_swagger
  pending --> canceled: customer_or_admin_cancels

  preparing --> ready_for_pickup: ops_marks_ready_via_swagger
  preparing --> canceled: admin_cancels

  ready_for_pickup --> accepted: rider_claims_order
  accepted --> arrived_at_merchant: rider_taps_arrived
  arrived_at_merchant --> picked_up: rider_confirms_pickup
  picked_up --> in_transit: rider_starts_delivery
  in_transit --> delivered: rider_confirms_delivery

  delivered --> [*]
  canceled --> [*]
```

---

## COD ID Validation (Not Implemented)

The original design gated COD checkout on `profiles.id_document_url`/`id_verified`. **Neither column exists** in the implemented `profiles` table — there is no ID-upload popup, no document review, no COD-fraud gate beyond phone OTP verification (`phone_verified`). Treat this flow as unbuilt, not as a hidden/disabled feature.

---

## Errand (Pabili) Order

**Refs:** C-5.2

```mermaid
flowchart LR
  Form[Customer fills errand form] --> Validate[Store name + items + budget]
  Validate --> Post[POST /orders/errand on cartman-server]
  Post --> JSONB[errand_payload JSONB]
  JSONB --> Ready[Born ready_for_pickup immediately]
  Ready --> Dispatch[Rider claims from ranked feed]
```

| Field in `errand_payload` (implemented column name — original design called it `custom_description`) | Example |
|------------------------------|---------|
| Store name | "Antique Public Market" |
| Item list | "2kg rice, cooking oil" |
| Estimated budget | 500 PHP |

No `merchant_id` or `order_items`. Born `ready_for_pickup` directly — no ops accept/ready step, unlike food/grocery.

---

## Courier Order (`pickup_delivery`)

**Refs:** C-5.3

The implemented `order_type` enum value is `pickup_delivery`, not `courier`.

```mermaid
flowchart LR
  Pickup[Pin pickup on OSM map] --> Dropoff[Pin dropoff on OSM map]
  Dropoff --> Confirm[Customer confirms COD]
  Confirm --> Post[POST /orders/courier on cartman-server]
  Post --> Fee[Fee computed server-side in orders.service.ts]
  Fee --> Ready[Born ready_for_pickup immediately]
```

Stores `pickup_lat`/`pickup_lng`/`dropoff_lat`/`dropoff_lng` (not `point` columns). Fee is calculated server-side at placement — a `fare-calc` Edge Function file exists in `cartman-mobile/supabase/functions` but is unused; the client's local `fare.dart` is a display-only preview, not authoritative.

---

## Rider Order Claim (Race Condition)

**Refs:** R-1.1, R-1.2, R-4.1  
**Target latency:** under 50ms

```mermaid
sequenceDiagram
  participant R1 as Rider A App
  participant R2 as Rider B App
  participant Server as cartman-server
  participant DB as PostgreSQL
  participant RT as Supabase Realtime

  R1->>Server: PATCH /orders/:id/claim
  R2->>Server: PATCH /orders/:id/claim
  Server->>DB: conditional updateMany (order X)
  DB-->>Server: 1 row updated (winner: R1)
  Server-->>R1: 200 order assigned
  Server-->>R2: 409 already claimed
  DB->>RT: WAL broadcast accepted
  RT-->>R1: order-watch confirms
  RT-->>R2: feed change-signal → debounced refetch removes card
```

Gated server-side before the update runs: rider on-duty (`riders.is_active`), queue-depth cap **2** concurrent non-terminal orders, wallet not locked (`rider_net_cash > -₱2,000`).

**Decline flow:** Card removed locally (Hive); no server write.

---

## Rider Delivery Status Progression

**Refs:** R-5.1

Single contextual button advances status in order:

```
accepted → arrived_at_merchant → picked_up → in_transit → delivered
```

Each tap: `PATCH /orders/:id/status` on `cartman-server` (validates the legal-transition chain server-side, strict, no skipping) → Realtime broadcast to customer.

**Navigation (R-4.3):** Deep link to Google Maps / Apple Maps / Waze with target coords.

---

## Merchant Order Queue (Ops Interim)

No merchant panel exists — this is ops, by hand, via Swagger, standing in for it.

```mermaid
flowchart LR
  Incoming[New food/grocery order — pending] -.->|ops, PATCH accept| Accept[preparing]
  Accept -.->|ops, PATCH ready| Ready[ready_for_pickup]
  Ready --> Broadcast[Rider feed change-signal fires]
```

| Step | Endpoint | Status change |
|------|----------|----------------|
| Accept | `PATCH /orders/:id/accept` (ops, Swagger) | `pending` → `preparing` |
| Ready | `PATCH /orders/:id/ready` (ops, Swagger) | `preparing` → `ready_for_pickup` |

`errand`/`pickup_delivery`/`ride` skip this entirely — born `ready_for_pickup`.

---

## Admin Cancel and Reassign

**Status:** branch `admin-endpoints`, in progress — not yet merged.

```mermaid
sequenceDiagram
  participant Admin as Admin (via dashboard or Swagger)
  participant Server as cartman-server
  participant DB as PostgreSQL
  participant RT as Supabase Realtime

  Admin->>Server: PATCH /admin/orders/:id/cancel
  Server->>DB: UPDATE status=canceled (any pre-delivered status)
  DB->>RT: WAL broadcast canceled
  RT-->>Server: (customer/rider order-watch updates)
```

```mermaid
sequenceDiagram
  participant Admin as Admin (via dashboard or Swagger)
  participant Server as cartman-server
  participant DB as PostgreSQL
  participant RT as Supabase Realtime

  Admin->>Server: PATCH /admin/orders/:id/reassign {new_rider_id}
  Note over Server: Pre-pickup only (status = accepted); mirrors claim guards\n(on-duty, queue-depth cap 2, wallet not locked) for the new rider
  Server->>DB: UPDATE assigned_rider_id, status=accepted
  DB->>RT: WAL broadcast
  RT-->>Server: old rider's order-watch drops it; new rider's picks it up
```

Both are `@Roles('admin')`. Cancel accepts any status before `delivered`; reassign is pre-pickup only (before `arrived_at_merchant`).

---

## Wallet and Lockout

**Refs:** R-3.1, R-3.2

```mermaid
flowchart TB
  Delivered[Order status -> delivered] --> ServerWriter[cartman-server delivered-transition handler\ntransactional + idempotent]
  ServerWriter --> Credit[credit_delivery_reward INSERT]
  ServerWriter --> Debit[debit_cod_order INSERT if COD]

  AdminRemit[Admin: POST /ledger/transactions] --> Remit[credit_remittance INSERT]

  Credit --> Compute[rider_net_cash aggregate]
  Debit --> Compute
  Remit --> Compute

  Compute --> Check{rider_net_cash <= -2000?}
  Check -->|Yes| Lockout[GET /orders/feed returns 403; claim blocked server-side]
  Check -->|No| Normal[Rider can claim orders]
```

| Transaction | Writer | When | Status |
|-------------|--------|------|--------|
| `credit_delivery_reward` | `cartman-server` (delivered-transition handler) | On delivery | Implemented |
| `debit_cod_order` | Same handler | COD collected | Implemented |
| `credit_remittance` | `cartman-server` (`POST /ledger/transactions`, `@Roles('admin')`) | Rider remits cash | Implemented |
| `debit_commission` | — | Would apply commission | **Never written — not implemented** |

Rider app: **SELECT only**, via `GET /ledger/me/*` — never a direct table read/write.

---

## Support Ticket and Override (Not Implemented)

No `support_tickets` table exists in the implemented schema. The account-override flow described here (password reset / auth bypass gated by a ticket + user confirmation) is design-only — there is no ticket submission, no admin review queue, and no override execution path in code today.

---

## Realtime Subscription Flows

### Event catalog

| Event | Producer | Consumers | Status |
|-------|----------|-----------|--------|
| Order status change | `cartman-server` (write) | Customer/Rider order-watch channels | Implemented |
| Feed change-signal | `cartman-server` (write) | On-duty riders — triggers debounced `GET /orders/feed` refetch, not a raw row push | Implemented |
| GPS telemetry | Rider app, batched `POST /riders/me/telemetry` | Server-side storage only | No customer-tracking/admin-map consumer wired yet |
| Wallet txn | `cartman-server` (write) | Rider app, via `GET /ledger/me/*` reads (not a realtime push) | Implemented |
| Push | `cartman-server` webhook receiver | FCM to device | **Stub — logging only** |

### Customer — track single order (C-4.1, unchanged from original design)

```javascript
supabase
  .channel('order:' + orderId)
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'orders',
    filter: 'id=eq.' + orderId
  }, handleStatusChange)
  .subscribe()
```

### Rider — feed change-signal, not a direct feed push (R-1.1)

```javascript
supabase
  .channel('available-orders')
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'orders',
    filter: 'status=eq.ready_for_pickup'
  }, () => debouncedRefetchRankedFeed()) // GET /orders/feed on cartman-server
  .subscribe()
```

The event handler doesn't append the changed row directly to the feed (as the original design's `appendToFeed` implied) — it triggers a refetch of the server's ranked feed, which re-scores and re-sorts.

---

## Push Notification Flow

**Refs:** C-4.2 — **Not functional today.**

```mermaid
flowchart LR
  StatusChange[orders.status UPDATE] --> Webhook[cartman-server webhook receiver]
  Webhook --> Stub[FCM fan-out — logging stub]
  Stub -.->|no real send| Device[Customer or Rider device]
```

`firebase_messaging` is not in the mobile apps; no Firebase project is wired up. Device-token plumbing (`device_tokens` table, `POST` endpoints) exists, but nothing consumes it for real delivery.

---

## Background GPS Flow (Rider)

**Refs:** R-2.1, R-4.2

```mermaid
flowchart TB
  Toggle[On-Duty ON] --> Service[Android Foreground Service]
  Service --> GPS[geolocator, batched interval]
  GPS --> Post[POST /riders/me/telemetry on cartman-server]
  Post --> Insert[INSERT rider_locations]
  ToggleOff[On-Duty OFF] --> Stop[Stop service, save battery]
```

Telemetry is batched through the server (not a raw per-tick client insert to Supabase). No customer-tracking or admin-dispatch-map consumer of this data is wired up yet — it's write-only today.

---

## App Access Control Flow

Post-login role check on every client:

| App | Reject when |
|-----|-------------|
| Customer | `role != customer` |
| Rider | `role != rider` (no `verification_status` check — doesn't exist) |
| Merchant panel | N/A — no merchant app exists |
| Admin dashboard | No login exists yet; planned check is `role == admin` once dashboard auth ships |

Write-path enforcement is server-side (`JwtAuthGuard` + `RolesGuard` on `cartman-server`); the RLS-based enforcement in the original design applies only to the direct-read surfaces (§ schema.md).
