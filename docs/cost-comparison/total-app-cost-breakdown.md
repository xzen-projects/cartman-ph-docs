# Cartman PH — Total App Cost Breakdown (Phase 1)

**Scope:** Every recurring and one-time cost to run Cartman PH in Phase 1 (Antique Province, 300–1,000 DAU) — backend, hosting, third-party APIs, and distribution.
**This document consolidates:** [README.md](./README.md) (Hostinger VPS + Supabase), [render-vs-railway.md](./render-vs-railway.md) (`cartman-server` hosting), and [sms-gateway-cost-breakdown.md](./sms-gateway-cost-breakdown.md) (SMS OTP), plus email, push, and maps costs not previously broken out on their own.
**Exchange rate used:** ~₱57 / USD (verify current rate before budgeting).
**Status of decisions referenced:** Railway (server hosting) and Flutter (mobile framework) are **RESOLVED** per `cartmanph-docs/docs/breakdown/strategy.md` (confirmed 2026-07-01). Costs below reflect resolved decisions where available.

---

## 1. Where every component runs

| Component | Runs on | Cost driver |
|---|---|---|
| PostgreSQL, Auth, Realtime, Storage, Edge Functions | **Supabase Cloud** | Supabase Pro plan |
| `cartman-server` (NestJS REST API) | **Railway** | Compute usage (Pro plan credit) |
| Merchant Web, Admin Dashboard, Financial Ledger (SPAs) | **Hostinger VPS** | Fixed VPS plan |
| SMS OTP delivery | **Semaphore** | Per-credit, pay-as-you-go |
| Email OTP delivery (C-1.2, resolved 2026-07-02) | **Resend** | Free tier, then per-email |
| Push notifications | **FCM (Firebase Cloud Messaging)** | Free |
| Map tiles | **OpenStreetMap** | Free — no API key, no Google Maps/Mapbox (non-negotiable, see `cartman-mobile/CLAUDE.md` §3.1) |
| Mobile app distribution | **Sideload APK** → Google Play Console (Franz's account) → Cartman PH account at scale | Free during pilot; $25 one-time when migrating to Cartman's own Play Console account |
| iOS | Not in Phase 1 — web app access only | Deferred; Apple Developer Program ($99/yr) not budgeted until Phase 2 |

---

## 2. Recurring monthly costs

| Item | Low estimate | High estimate | Source |
|---|---|---|---|
| Hostinger KVM 1 (web panels, renewal rate) | ₱679 | ₱679 | [README.md](./README.md) |
| Railway Pro (`cartman-server`, 150–600 DAU) | ₱1,140 | ₱1,710 (1,000 DAU) | [render-vs-railway.md](./render-vs-railway.md) |
| Supabase Pro | ₱1,400 | ₱1,425 | [README.md](./README.md), [render-vs-railway.md](./render-vs-railway.md) |
| Semaphore SMS OTP (100–1,000 OTPs/mo) | ₱56 | ₱560 | [sms-gateway-cost-breakdown.md](./sms-gateway-cost-breakdown.md) |
| Resend email OTP | **₱0** (free tier: 3,000 emails/mo, 100/day cap) | ₱0 | resend.com/pricing, fetched 2026-07-02 |
| FCM push | ₱0 | ₱0 | Firebase free tier, no cap for this use case |
| OSM map tiles | ₱0 | ₱0 | No API key required |
| **Total (monthly)** | **₱3,275** | **₱4,374** | — |

### Annual total

| Scenario | Monthly | **Annual** |
|---|---|---|
| Low estimate (150–300 DAU, light OTP volume) | ₱3,275 | **~₱39,300/yr** |
| High estimate (1,000 DAU, heavier OTP volume) | ₱4,374 | **~₱52,488/yr** |

---

## 3. One-time / deferred costs (not in Phase 1 monthly budget)

| Item | Cost | When it applies |
|---|---|---|
| Domain registration (.com or .ph) | ~₱700–1,500/yr (estimate — not yet locked in project docs; verify with registrar before budgeting) | Ongoing, but not yet documented as a resolved line item |
| Google Play Console (Cartman PH's own account) | $25 one-time | When migrating off Franz's personal account per the resolved distribution plan (`strategy.md:173-174`) |
| Apple Developer Program | $99/yr | Deferred to Phase 2 (iOS native) — `ARCHITECTURE.md:248` explicitly excludes this from Phase 1 for cost/timeline reasons |
| Extra Semaphore sender name | ₱112/mo or ₱1,120/yr | Only if a second SMS sender ID is needed; first is free |

---

## 4. If email volume outgrows Resend's free tier

Resend's free tier covers 3,000 emails/month (capped at 100/day) — comfortably covers OTP-only usage at current projected volume (see `strategy.md:205`, which assumed this exact ~3,000/month free-tier ceiling when scoping the email OTP decision). If usage grows past that:

| Resend tier | Monthly | Emails included |
|---|---|---|
| Pro | $20 (~₱1,140) | 50,000/mo |
| Pro | $35 (~₱1,995) | 100,000/mo |
| Scale | $90–$1,150 (~₱5,130–₱65,550) | 100K–2.5M/mo |

Not expected to trigger in Phase 1 — email is OTP-only (2FA/password reset), not general notifications.

---

## 5. What's driving cost vs what's free

**Paid, unavoidable:**
- Supabase Pro (Postgres, Realtime, Auth, Edge Functions) — the backbone of the whole architecture
- Railway (API server compute)
- Hostinger VPS (web panel hosting)
- Semaphore (SMS OTP — carriers charge per message, no free alternative for PH-local delivery at volume)

**Paid but currently ₱0 in practice:**
- Resend (free tier covers current OTP-only volume)
- FCM (no cost at any realistic Phase 1 volume)

**Structurally free (architectural decisions, not just current usage):**
- OSM map tiles — enforced as a non-negotiable in `cartman-mobile/CLAUDE.md` §3 rule 1, specifically to avoid Google Maps/Mapbox billing risk
- Sideload distribution — avoids Play Store review cycles and (temporarily) the $25 console fee

---

## 6. Bottom line

Cartman PH's Phase 1 recurring cost is **~₱3,275–₱4,374/month (~₱39,300–₱52,488/year)**, dominated by Supabase Pro and server/VPS hosting — not by third-party messaging APIs, which stay under ₱600/month combined (SMS + email) at pilot volume. The architecture's insistence on free-tier-friendly choices (OSM over Google Maps, FCM over paid push, Resend's free OTP allotment) keeps the marginal cost of *user growth* low; the fixed Supabase + hosting costs are what scale with DAU, not the messaging layer.

---

## Related documents

| Document | Purpose |
|---|---|
| [README.md](./README.md) | Hostinger VPS sizing detail, Supabase tier guidance by DAU |
| [render-vs-railway.md](./render-vs-railway.md) | Full Railway vs Render comparison for `cartman-server` |
| [sms-gateway-cost-breakdown.md](./sms-gateway-cost-breakdown.md) | Semaphore vs PhilSMS vs Twilio, per-message and per-volume detail |
| [ARCHITECTURE.md](../../ARCHITECTURE.md) | Canonical system design and the OSM/no-paid-maps decision |
| [strategy.md](../breakdown/strategy.md) | Stack decisions, resolved vs open items |
