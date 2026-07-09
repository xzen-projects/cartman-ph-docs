# Cartman Server - Required Updates for Admin Dashboard V2

> **Status: implemented on `cartman-server` branch `admin-v2` (pending merge).** All 4 Prisma models and all 13 endpoints below shipped as specified and were review-approved as spec-compliant. See [`docs/api/cartman-server-endpoints.md`](./api/cartman-server-endpoints.md), section "Admin Dashboard V2 — incidents, config, users, verifications, tickets", for the verified endpoint reference (request/response shapes confirmed against `admin.controller.ts` + the `admin-*.service.ts` files) and [`ARCHITECTURE.md` §10.4](../ARCHITECTURE.md#104-admin-dashboard) for how this maps to dashboard pages. Known gap carried forward from this spec: `POST /admin/config` is **store-and-serve only** — nothing in `orders.service.ts` reads `global_config` yet, so pricing/rider/merchant parameters set here have no effect on live orders. This spec document is kept for historical reference; do not treat it as the current source of truth if it and the endpoint reference disagree — the endpoint reference wins.

This document details the newly added frontend dashboard features that require corresponding backend implementations. The Cartman server agent should use this specification to create the missing API endpoints, add the new Prisma models, and apply the required business logic.

## 1. New Prisma Data Models

The following tables need to be created in the database and mapped via the Prisma schema.

### Incidents
For tracking operational issues and system alerts.
```prisma
model incidents {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  type          String
  severity      String
  status        String   @default("Open") // Enum: Open, Investigating, Resolved
  related_order String?
  assignee      String?
  reported_at   DateTime @default(now()) @db.Timestamptz()
}
```

### Verifications
For reviewing COD ID requirements and Merchant Business Permits.
```prisma
model verifications {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  user_id       String   @db.Uuid
  document_type String
  document_url  String
  status        String   @default("Pending") // Enum: Pending, Approved, Rejected
  submitted_at  DateTime @default(now()) @db.Timestamptz()
}
```

### Tickets
For managing multi-channel support requests from customers, riders, and merchants.
```prisma
model tickets {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  user_id       String   @db.Uuid
  category      String   // General, Dispute, Technical, Billing
  subject       String
  priority      String   @default("Low") // Low, Medium, High
  status        String   @default("Open") // Open, In Progress, Closed
  created_at    DateTime @default(now()) @db.Timestamptz()
}
```

### Global Configuration (Key-Value or Single Row)
For dynamic business logic variables (Pricing, Penalties, Merchant Rules).
```prisma
model global_config {
  id                      Int      @id @default(1) // Singleton
  base_rate               Decimal  @default(39.0)
  base_radius             Decimal  @default(2.0)
  surcharge_per_km        Decimal  @default(15.0)
  strike_limit            Int      @default(3)
  auto_assign             Boolean  @default(true)
  pro_exposure_multiplier Decimal  @default(2.0)
  allow_ads               Boolean  @default(true)
  updated_at              DateTime @default(now()) @updatedAt @db.Timestamptz()
}
```

---

## 2. New Admin API Endpoints

All endpoints are prefixed with `/admin` and require standard Admin Supabase JWT authentication.

### A. Incidents & System Logs
Monitor and manage operational issues.
- **GET** `/admin/incidents`
  - **Query Params**: `limit`, `offset`
  - Returns paginated list of `incidents`.
- **POST** `/admin/incidents`
  - Create a new incident log.
  - **Body**: `{ type, severity, relatedOrder, assignee }`
- **PATCH** `/admin/incidents/:id`
  - Update incident status.
  - **Body**: `{ "status": "Investigating" }`

### B. Global Business Configuration
Manage system-wide parameters (Adaptive Pricing & Rider Penalties).
- **GET** `/admin/config`
  - Retrieve the current singleton `global_config` record.
- **POST** `/admin/config`
  - Update the `global_config` record.
  - **Body**: 
    ```json
    {
      "pricingConfig": { "baseRate": 39, "baseRadius": 2, "surchargePerKm": 15 },
      "riderConfig": { "strikeLimit": 3, "autoAssign": true },
      "merchantConfig": { "proExposureMultiplier": 2.0, "allowAds": true }
    }
    ```

### C. Advanced User Profiling & Overrides
Enhancements to the existing `/admin/users` routes for executing manual admin interventions.
- **GET** `/admin/users`
  - **Query Params**: `limit`, `offset`, `role`
  - Retrieve a list of users across the platform, including their current active/suspended status and lock state.
- **POST** `/admin/users/:id/bypass-auth`
  - Removes Supabase authentication blocks/locks on a specific user.
- **POST** `/admin/users/:id/reset-2fa`
  - Triggers an immediate, forced password or 2FA reset workflow for the user.
- **POST** `/admin/users/:id/toggle-status`
  - Toggles a user's `is_active` or `status` state (ban/unban).

### D. Verification Pipeline
Approve or reject uploaded credentials.
- **GET** `/admin/verifications`
  - **Query Params**: `limit`, `offset`, `status`
  - Retrieve pending compliance documents.
- **POST** `/admin/verifications/:id/approve`
  - Updates the `verifications` table to "Approved" and flags the related user/rider as verified.
- **POST** `/admin/verifications/:id/reject`
  - Updates the `verifications` table to "Rejected" and triggers a re-upload notification.

### E. Support Tickets
Manage customer, rider, and merchant support communications.
- **GET** `/admin/tickets`
  - **Query Params**: `limit`, `offset`, `status`
  - Retrieves paginated list of `tickets`.
- **PATCH** `/admin/tickets/:id/status`
  - Updates ticket state.
  - **Body**: `{ "status": "In Progress" }`
