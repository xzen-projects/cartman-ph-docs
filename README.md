# Cartman PH — Antique Doc Review

Documentation repository for the **Cartman PH** delivery platform (Phase 1: Antique Province). This repo holds architecture specs and source backlogs — **not** application source code.

---

## Quick start — what to read

| If you are… | Start here | Then read |
|-------------|------------|-----------|
| **New to the project** | [ARCHITECTURE.md](./ARCHITECTURE.md) | [docs/breakdown/README.md](./docs/breakdown/README.md) |
| **Backend / DB engineer** | [docs/breakdown/schema.md](./docs/breakdown/schema.md) | [docs/breakdown/flows.md](./docs/breakdown/flows.md), [docs/api/](./docs/api/cartman-server-endpoints.md) |
| **Mobile developer** | [ARCHITECTURE.md §10](./ARCHITECTURE.md#10-application-architecture) | [docs/api/](./docs/api/cartman-server-endpoints.md), [docs/breakdown/flows.md](./docs/breakdown/flows.md), source PDFs below |
| **Product / ops** | [docs/breakdown/domains.md](./docs/breakdown/domains.md) | [docs/breakdown/strategy.md](./docs/breakdown/strategy.md) |
| **Tech lead / architect** | [ARCHITECTURE.md](./ARCHITECTURE.md) | All of `docs/breakdown/` |

### Recommended reading order

1. **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Canonical full-platform design (customer + rider apps, merchant panel, admin, ledger). Read this first unless you only need one slice.
2. **[docs/breakdown/](./docs/breakdown/)** — Same content reorganized by concern:
   - [domains.md](./docs/breakdown/domains.md) — Bounded contexts and who owns what
   - [schema.md](./docs/breakdown/schema.md) — Tables, enums, RLS, wallet rules
   - [flows.md](./docs/breakdown/flows.md) — Onboarding, orders, dispatch, wallet sequences
   - [strategy.md](./docs/breakdown/strategy.md) — Stack, deployment, roadmap, open decisions
3. **[docs/api/](./docs/api/cartman-server-endpoints.md)** — cartman-server HTTP endpoint reference: every route, auth/token usage, request/response shapes, worked curl examples. Read this before wiring any client to the server.
4. **Source PDFs** (optional, for task-level acceptance criteria):
   - [Cartman PH - Customer-Side Mobile Application.pdf](./references/Cartman%20PH%20-%20Customer-Side%20Mobile%20Application.pdf) — Tasks C-1.1 through C-7.2
   - [Cartman PH — Rider Side Mobile Application Task List.pdf](./references/Cartman%20PH%20—%20Rider%20Side%20Mobile%20Application%20Task%20List.pdf) — Tasks R-1.1 through R-6.3
5. **[docs/proposals/](./docs/proposals/)** — Designs written ahead of a decision. Each proposal carries its own status header — some have since (partially) shipped.
   - [rider-dispatch-weighting.md](./docs/proposals/rider-dispatch-weighting.md) — Weighted dispatch for delivery + Pasakay orders. **A simplified version shipped in Phase 1** (`GET /orders/feed`, single ranked list, env-tunable weights); the staggered-wave, 5-factor design remains proposal-only — see the doc's "Proposal vs. shipped" table.

---

## Document authority

| Document | Role |
|----------|------|
| **ARCHITECTURE.md** | Single source of truth for system design. Do not fork divergent specs. |
| **docs/breakdown/** | Views of the same design — convenient slices, not alternate truth. If breakdown and ARCHITECTURE disagree, **ARCHITECTURE.md wins**. |
| **docs/api/** | Endpoint contracts for `cartman-server` — must track the controllers in the `cartman-server` repo. **Code wins on conflict**; update the doc in the same change that alters a controller. |
| **PDF backlogs** | Original mobile scope and acceptance criteria. Web panels (merchant, admin, ledger) are inferred in ARCHITECTURE.md only. |

---

## What to ignore

| Path | Why |
|------|-----|
| **`.git/`** | Version control metadata — not project documentation. |
| **`.cursor/`**, **agent transcripts** (if present locally) | IDE / AI session data — not part of the product spec. |

There is **no runnable app**, database migration, or CI config in this repository — implementation lives in sibling repos: **`cartman-mobile`** (Flutter melos workspace: `apps/`, `packages/`, `supabase/` migrations + functions) and **`cartman-server`** (NestJS + Prisma API). Active development is on the `mvp-readiness` branch in both.

---

## Repo layout

```
.
├── README.md                          ← You are here
├── ARCHITECTURE.md                    ← Main architecture (read first)
├── references/
│   ├── Cartman PH - Customer-Side …pdf ← Customer mobile backlog (reference)
│   ├── Cartman PH — Rider Side …pdf    ← Rider mobile backlog (reference)
│   └── meeting-notes-2026-07-01.md
└── docs/
    ├── api/
    │   └── cartman-server-endpoints.md ← Endpoint + auth usage reference
    ├── breakdown/
    │   ├── README.md                  ← Index for breakdown docs
    │   ├── domains.md
    │   ├── schema.md
    │   ├── flows.md
    │   └── strategy.md
    ├── cost-comparison/               ← Hosting / SMS cost worksheets
    └── proposals/
        └── rider-dispatch-weighting.md
```

---

## Phase 1 at a glance

- **Region:** Antique Province, Philippines  
- **Clients:** Android customer + rider apps (Flutter). Merchant panel and admin dashboard are deferred — ops run via Swagger + SQL for MVP  
- **Backend:** `cartman-server` (NestJS + Prisma, deployed on Render) is the authoritative writer; Supabase provides PostgreSQL, Auth (GoTrue), Realtime, and Storage. Reads + realtime stay direct-to-Supabase under RLS; all writes and money reads go through the server ([endpoint reference](./docs/api/cartman-server-endpoints.md))  
- **Payments:** Cash on Delivery (COD) only  
- **Maps:** OpenStreetMap in-app; external map apps for rider turn-by-turn  

Details: [ARCHITECTURE.md §1](./ARCHITECTURE.md#1-executive-summary) and [strategy.md](./docs/breakdown/strategy.md).

---

## Previewing Markdown

Mermaid diagrams require a compatible renderer (e.g. VS Code Markdown Preview, GitHub, or [markdownlivepreview.com](https://markdownlivepreview.com)). If a diagram fails, check Mermaid 11 syntax — auth ER diagrams must not use `PK_FK` as a key constraint.

---

## Contributing

When updating design:

1. Edit **ARCHITECTURE.md** for canonical changes.
2. Update the relevant **docs/breakdown/** file(s) to stay in sync.
3. If a change touches a `cartman-server` controller/DTO, update **docs/api/** in the same change — code wins on conflict.
4. Do not treat PDFs as living documents — note deltas in ARCHITECTURE.md instead.

Implementation code lives in the sibling `cartman-mobile` and `cartman-server` repos (see [ARCHITECTURE.md §4](./ARCHITECTURE.md#4-repository-layout)).
