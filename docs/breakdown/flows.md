# System Flows

Process and interaction flows extracted from [ARCHITECTURE.md](../../ARCHITECTURE.md).

---

## Flow Index

| Flow | Type | Actors |
|------|------|--------|
| [Customer registration](#customer-registration-otp) | Onboarding | Customer, Auth, Edge, Semaphore |
| [Rider registration](#rider-registration-approval) | Onboarding | Rider, Admin |
| [Merchant registration](#merchant-registration-approval) | Onboarding | Merchant, Admin, Storage |
| [Food order lifecycle](#food-order-lifecycle) | Core | Customer, Merchant, Rider |
| [Errand order](#errand-pabili-order) | Core | Customer, Rider, Ops |
| [Courier order](#courier-order) | Core | Customer, Rider |
| [Rider order claim](#rider-order-claim-race-condition) | Dispatch | Multiple riders |
| [Rider delivery progression](#rider-delivery-status-progression) | Dispatch | Rider |
| [Merchant order queue](#merchant-order-queue) | Fulfillment | Merchant |
| [Wallet and lockout](#wallet-and-lockout) | Finance | Rider, Admin |
| [Realtime subscriptions](#realtime-subscription-flows) | Events | All clients |
| [Push notification](#push-notification-flow) | Events | FCM |

---

## Customer Registration (OTP)

**Refs:** C-1.1, C-1.2

```mermaid
sequenceDiagram
  participant C as Customer App
  participant Auth as Supabase Auth
  participant Edge as Edge Function
  participant SMS as Semaphore SMS
  participant DB as PostgreSQL

  C->>Auth: signUp(phone or email)
  Auth->>DB: create auth.users + profiles role customer
  Auth-->>C: session pending verification
  C->>Edge: POST send-otp
  Edge->>SMS: send verification code
  SMS-->>C: SMS delivered
  C->>Edge: POST verify-otp
  Edge->>DB: profiles.phone_verified = true
  Edge-->>C: verification success
  C->>Auth: refresh session
  Auth-->>C: JWT with customer role
```

**Gates:**
- Checkout blocked until `phone_verified = true`
- Session stored in encrypted device storage

---

## Rider Registration (Approval)

**Refs:** R-4.2 (post-approval on-duty)

```mermaid
sequenceDiagram
  participant R as Rider App
  participant Auth as Supabase Auth
  participant DB as PostgreSQL
  participant Admin as Admin Dashboard

  R->>Auth: signUp(phone)
  Auth->>DB: profiles role rider + riders pending
  R->>DB: upload ID docs via Storage
  Admin->>DB: review application
  alt Approved
    Admin->>DB: verification_status = approved
    R->>Auth: login
    R-->>R: unlock on-duty toggle
  else Rejected
    Admin->>DB: verification_status = rejected
    R-->>R: show rejection screen
  end
```

**Feed gate:** `verification_status = approved` AND `is_active = true`

---

## Merchant Registration (Approval)

```mermaid
sequenceDiagram
  participant M as Merchant Panel
  participant Auth as Supabase Auth
  participant DB as PostgreSQL
  participant Storage as Supabase Storage
  participant Admin as Admin Dashboard

  M->>Auth: signUp(email)
  Auth->>DB: profiles role merchant + merchants pending
  M->>Storage: upload business permit
  M->>DB: save document refs
  Admin->>DB: review application
  alt Approved
    Admin->>DB: merchants.status = active
    M-->>M: unlock menu + order queue
  else Rejected
    Admin->>DB: merchants.status = suspended
    M-->>M: show restricted screen
  end
```

---

## Food Order Lifecycle

End-to-end from placement to delivery.

```mermaid
sequenceDiagram
  participant Cust as Customer App
  participant DB as PostgreSQL
  participant RT as Realtime
  participant Merch as Merchant Panel
  participant Rider as Rider App
  participant FCM as FCM

  Cust->>DB: INSERT order pending COD
  DB->>RT: WAL event
  RT-->>Merch: new order
  Merch->>DB: UPDATE preparing
  RT-->>Cust: stepper update
  Merch->>DB: UPDATE ready_for_pickup
  RT-->>Rider: available order in feed
  Rider->>DB: race-safe claim
  RT-->>Cust: rider assigned
  Rider->>DB: status to delivered
  RT-->>Cust: delivered
  DB->>FCM: push order arrived
  FCM-->>Cust: lock-screen alert
```

### Order status state machine

```mermaid
stateDiagram-v2
  [*] --> pending: customer_submits_cod_order

  pending --> preparing: merchant_accepts
  pending --> cancelled: customer_or_admin_cancels

  preparing --> ready_for_pickup: merchant_marks_ready
  preparing --> cancelled: merchant_or_admin_cancels

  ready_for_pickup --> accepted: rider_claims_order
  accepted --> arrived_at_merchant: rider_taps_arrived
  arrived_at_merchant --> picked_up: rider_confirms_pickup
  picked_up --> in_transit: rider_starts_delivery
  in_transit --> delivered: rider_confirms_delivery

  delivered --> [*]
  cancelled --> [*]
```

---

## Errand (Pabili) Order

**Refs:** C-5.2

```mermaid
flowchart LR
  Form[Customer fills errand form] --> Validate[Store name + items + budget]
  Validate --> Insert[INSERT order type errand]
  Insert --> JSONB[custom_description JSONB]
  JSONB --> Dispatch[Ops / rider dispatch]
```

| Field in `custom_description` | Example |
|------------------------------|---------|
| Store name | "Antique Public Market" |
| Item list | "2kg rice, cooking oil" |
| Estimated budget | 500 PHP |

No `merchant_id` or `order_items`. Rider claims from feed when status reaches `ready_for_pickup` (may be set by admin/ops for non-merchant flow).

---

## Courier Order

**Refs:** C-5.3

```mermaid
flowchart LR
  Pickup[Pin pickup on OSM map] --> Dropoff[Pin dropoff on OSM map]
  Dropoff --> Fee[Edge Function calculate fee]
  Fee --> Confirm[Customer confirms COD]
  Confirm --> Insert[INSERT order type courier]
```

Stores `pickup_coords` and `dropoff_coords`. Fee calculated server-side for consistency.

---

## Rider Order Claim (Race Condition)

**Refs:** R-1.1, R-1.2, R-4.1  
**Target latency:** under 50ms

```mermaid
sequenceDiagram
  participant R1 as Rider A App
  participant R2 as Rider B App
  participant DB as PostgreSQL
  participant RT as Supabase Realtime

  R1->>DB: UPDATE claim order X
  R2->>DB: UPDATE claim order X
  DB-->>R1: 1 row winner
  DB-->>R2: 0 rows conflict
  DB->>RT: WAL broadcast accepted
  RT-->>R1: order assigned
  RT-->>R2: order unavailable
  R2-->>R2: remove card already claimed
```

**Decline flow:** Card removed locally (Hive/AsyncStorage); no server write.

---

## Rider Delivery Status Progression

**Refs:** R-5.1

Single contextual button advances status in order:

```
accepted → arrived_at_merchant → picked_up → in_transit → delivered
```

Each tap: `UPDATE orders SET status = :next` → Realtime broadcast to customer, merchant, admin.

**Navigation (R-4.3):** Deep link to Google Maps / Apple Maps / Waze with target coords.

---

## Merchant Order Queue

```mermaid
flowchart LR
  Incoming[Incoming Orders Realtime] --> Accept[Accept Order]
  Accept --> Prepare[Mark Preparing]
  Prepare --> Ready[Mark Ready for Pickup]
  Ready --> Broadcast[Rider Feed Triggered]
```

| Step | Status change | Side effect |
|------|---------------|-------------|
| Accept | `pending` → `preparing` | Customer stepper updates |
| Ready | `preparing` → `ready_for_pickup` | Rider feed subscription fires |

---

## Wallet and Lockout

**Refs:** R-3.1, R-3.2

```mermaid
flowchart TB
  Delivered[Order marked delivered] --> Trigger[DB trigger or admin]
  Trigger --> Credit[credit_delivery_reward INSERT]
  Trigger --> Debit[debit_cod_order INSERT if COD]

  AdminRemit[Admin logs remittance] --> Remit[credit_remittance INSERT]

  Credit --> Compute[Compute W_r aggregate]
  Debit --> Compute
  Remit --> Compute

  Compute --> Check{W_r <= -2000?}
  Check -->|Yes| Lockout[Hide feed block accept]
  Check -->|No| Normal[Rider can accept orders]
```

| Transaction | Writer | When |
|-------------|--------|------|
| `credit_delivery_reward` | DB trigger / admin | On delivery |
| `debit_cod_order` | Admin ledger | COD collected |
| `credit_remittance` | Admin ledger | Rider remits cash |

Rider app: **SELECT only** on `rider_wallet_transactions`.

---

## Realtime Subscription Flows

### Event catalog

| Event | Producer | Consumers |
|-------|----------|-----------|
| Order status change | Merchant, Rider, Admin | Customer, Rider, Merchant, Admin |
| `ready_for_pickup` | Merchant | On-duty riders |
| GPS telemetry | Rider app | Customer tracking, Admin map |
| Wallet txn | Admin ledger | Rider earnings view |
| Push | Edge / trigger | FCM to device |

### Customer — track single order (C-4.1)

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

### Rider — available orders feed (R-1.1)

```javascript
supabase
  .channel('available-orders')
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'orders',
    filter: 'status=eq.ready_for_pickup'
  }, appendToFeed)
  .subscribe()
```

---

## Push Notification Flow

**Refs:** C-4.2

```mermaid
flowchart LR
  StatusChange[orders.status UPDATE] --> Webhook[DB webhook or trigger]
  Webhook --> Edge[send-push-notification]
  Edge --> FCM[FCM API]
  FCM --> Device[Customer or Rider device]
```

Triggers on key transitions (e.g. rider assigned, delivered). Works when app is backgrounded or killed.

---

## Background GPS Flow (Rider)

**Refs:** R-2.1, R-4.2

```mermaid
flowchart TB
  Toggle[On-Duty ON] --> Service[Android Foreground Service]
  Service --> GPS[geolocator interval 10-30s]
  GPS --> Insert[INSERT rider_location_logs]
  ToggleOff[On-Duty OFF] --> Stop[Stop service save battery]
```

Customer sees assigned rider position via Realtime/poll on logs. Admin dispatch map uses same data.

---

## App Access Control Flow

Post-login role check on every client:

| App | Reject when |
|-----|-------------|
| Customer | `role != customer` |
| Rider | `role != rider` OR `verification_status != approved` |
| Merchant | `role != merchant` OR `status != active` |
| Admin / Ledger | `role != admin` |

Enforced client-side (route guards) and server-side (RLS).
