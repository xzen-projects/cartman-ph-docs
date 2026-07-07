# Cartman PH — Antique Doc Review

Documentation repository for the **Cartman PH** delivery platform (Phase 1: Antique Province). This repo holds architecture specs and source backlogs — **not** application source code.

---

## Quick start — what to read

| If you are… | Start here | Then read |
|-------------|------------|-----------|
| **New to the project** | [ARCHITECTURE.md](./ARCHITECTURE.md) | [docs/breakdown/README.md](./docs/breakdown/README.md) |
| **Backend / DB engineer** | [docs/breakdown/schema.md](./docs/breakdown/schema.md) | [docs/breakdown/flows.md](./docs/breakdown/flows.md) |
| **Mobile developer** | [ARCHITECTURE.md §10](./ARCHITECTURE.md#10-application-architecture) | [docs/breakdown/flows.md](./docs/breakdown/flows.md), source PDFs below |
| **Product / ops** | [docs/breakdown/domains.md](./docs/breakdown/domains.md) | [docs/breakdown/strategy.md](./docs/breakdown/strategy.md) |
| **Tech lead / architect** | [ARCHITECTURE.md](./ARCHITECTURE.md) | All of `docs/breakdown/` |

### Recommended reading order

1. **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Canonical full-platform design (customer + rider apps, merchant panel, admin, ledger). Read this first unless you only need one slice.
2. **[docs/breakdown/](./docs/breakdown/)** — Same content reorganized by concern:
   - [domains.md](./docs/breakdown/domains.md) — Bounded contexts and who owns what
   - [schema.md](./docs/breakdown/schema.md) — Tables, enums, RLS, wallet rules
   - [flows.md](./docs/breakdown/flows.md) — Onboarding, orders, dispatch, wallet sequences
   - [strategy.md](./docs/breakdown/strategy.md) — Stack, deployment, roadmap, open decisions
3. **Source PDFs** (optional, for task-level acceptance criteria):
   - [Cartman PH - Customer-Side Mobile Application.pdf](./Cartman%20PH%20-%20Customer-Side%20Mobile%20Application.pdf) — Tasks C-1.1 through C-7.2
   - [Cartman PH - Rider Side Mobile Application Task List.pdf](./Cartman%20PH%20-%20Rider%20Side%20Mobile%20Application%20Task%20List.pdf) — Tasks R-1.1 through R-6.3
4. **[docs/proposals/](./docs/proposals/)** — Not-yet-decided designs for future phases (Phase 2+). Not canonical, not implemented.
   - [rider-dispatch-weighting.md](./docs/proposals/rider-dispatch-weighting.md) — Weighted priority-wave dispatch for delivery + Pasakay orders

---

## Document authority

| Document | Role |
|----------|------|
| **ARCHITECTURE.md** | Single source of truth for system design. Do not fork divergent specs. |
| **docs/breakdown/** | Views of the same design — convenient slices, not alternate truth. If breakdown and ARCHITECTURE disagree, **ARCHITECTURE.md wins**. |
| **PDF backlogs** | Original mobile scope and acceptance criteria. Web panels (merchant, admin, ledger) are inferred in ARCHITECTURE.md only. |

---

## What to ignore

| Path | Why |
|------|-----|
| **`.git/`** | Version control metadata — not project documentation. |
| **`test.mmd`**, **`test.svg`** | Leftover Mermaid validation artifacts (safe to delete). |
| **`.cursor/`**, **agent transcripts** (if present locally) | IDE / AI session data — not part of the product spec. |
| **`apps/`**, **`packages/`**, **`supabase/`** | Not in this repo yet. Described in ARCHITECTURE as the *planned* monorepo layout for implementation elsewhere. |

There is **no runnable app**, database migration, or CI config in this repository today.

---

## Repo layout

```
.
├── README.md                          ← You are here
├── ARCHITECTURE.md                    ← Main architecture (read first)
├── Cartman PH - Customer-Side …pdf    ← Customer mobile backlog (reference)
├── Cartman PH - Rider Side …pdf       ← Rider mobile backlog (reference)
└── docs/
    └── breakdown/
        ├── README.md                  ← Index for breakdown docs
        ├── domains.md
        ├── schema.md
        ├── flows.md
        └── strategy.md
```

---

## Phase 1 at a glance

- **Region:** Antique Province, Philippines  
- **Clients:** Android customer + rider apps; merchant, admin, and ledger web panels  
- **Backend:** Supabase (PostgreSQL, Auth, Realtime, Storage, Edge Functions)  
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
3. Do not treat PDFs as living documents — note deltas in ARCHITECTURE.md instead.

Implementation code belongs in a separate `cartman-ph` monorepo when scaffolding begins (see [ARCHITECTURE.md §4](./ARCHITECTURE.md#4-repository-layout)).
