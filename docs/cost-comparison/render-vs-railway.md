# Render Pro vs Railway — NestJS Backend Cost Comparison

**Scope:** NestJS API server (cartman-server) hosting only.  
**Primary target:** 150–300 daily active users (Antique Province Phase 1 launch).  
**Extended reference:** Up to 1,000 DAU (growth planning).  
**Context:** Supabase Cloud handles PostgreSQL, Auth, Realtime, Storage, and Edge Functions — the NestJS server handles REST API logic, FCM dispatch, Semaphore OTP relay, and courier fee calculation.  
**Action item owner:** Louie Dale Cervera (assigned 2026-07-01 meeting)  
**Exchange rate used:** ~₱57 / USD (verify current rate before budgeting)

---

## Platform Pricing at a Glance

### Railway Pro — $25/month + compute

Railway Pro charges a **$25/month base seat fee** plus actual resource consumption billed per second on top. The base fee is **not** a credit — compute costs are additive.

| Resource | Rate |
|----------|------|
| **Base plan fee** | $25/month |
| **CPU** | $20 / vCPU / month ($0.000463/vCPU/min) |
| **RAM** | $10 / GB / month ($0.000231/GB/min) |
| **Egress (beyond included)** | $0.05/GB |
| **Bandwidth included** | 25 GB/month |
| **Volume storage** | $0.15/GB/month |

**Included features:** No service maximum · Full-stack previews · Horizontal autoscaling · Isolated environments · Private links · Workspace audit logs · AWS OIDC Integration · Chat support

**Sleep behavior:** None — containers stay always-on while deployed.  
**Scale-to-zero:** Available optionally (not recommended for production APIs).

> **Corrected from initial estimate:** Earlier pricing sources cited $20/month. The confirmed Pro plan is **$25/month + compute** (verified via Railway pricing screenshot, Jul 2026).

---

### Render — Fixed Instance Tiers

Flat monthly rate per web service instance. Always-on (no sleep on paid tiers). No per-second billing.

| Instance | Monthly (USD) | Monthly (PHP) | RAM | vCPU | Bandwidth included |
|----------|--------------|---------------|-----|------|-------------------|
| **Free** | $0 | ₱0 | 512 MB | 0.1 vCPU | 100 GB | Sleeps after 15 min idle |
| **Starter** | $7 | ~₱399 | 512 MB | 0.5 vCPU | 100 GB |
| **Standard** | $25 | ~₱1,425 | 2 GB | 1 vCPU | 100 GB |
| **Pro** | $85 | ~₱4,845 | 4 GB | 2 vCPU | 100 GB |
| **Pro Plus** | $175 | ~₱9,975 | 8 GB | 4 vCPU | 100 GB |

**Bandwidth overage:** $0.15/GB beyond 100 GB/month  
**Persistent disk:** $0.25/GB/month  
**Team workspace (optional):** $19/user/month for advanced team permissions  
**Sleep behavior:** Free tier only. Starter and above are always-on.

---

## Cartman PH NestJS Server — Resource Profile

The `cartman-server` (NestJS) is a **lightweight, I/O-bound API relay**. Supabase absorbs all heavy operations (Postgres writes, Realtime fanout, GPS inserts, Edge Functions). NestJS handles:

| Module | Load type |
|--------|-----------|
| AuthModule — OTP relay to Semaphore | Burst on sign-ups; low sustained |
| OrdersModule — race-safe claim, FCM push triggers | Moderate peaks at lunch/dinner rush |
| MerchantsModule — menu/stock read API | Low sustained |
| LedgerModule — wallet transaction API | Very low; admin-only writes |

### Estimated resource consumption

| DAU | Peak concurrent requests | Est. RAM | Est. CPU |
|-----|--------------------------|----------|----------|
| **150** | ~5–8 | ~200–350 MB | ~0.1–0.2 vCPU |
| **300** | ~10–18 | ~300–512 MB | ~0.2–0.35 vCPU |
| 600 | ~25–40 | ~400–768 MB | ~0.3–0.5 vCPU |
| 1,000 | ~45–65 | ~512 MB–1 GB | ~0.4–0.6 vCPU |

NestJS process baseline (idle): ~120–200 MB RAM. Under 150–300 DAU, 512 MB RAM with 0.5 vCPU is comfortable with headroom.

---

## Monthly Cost Estimates — NestJS Server Only

### Railway Pro (corrected: $25/month base + compute)

| Scenario | Est. provisioned | Compute cost | Base fee | **Total/month (USD)** | **Total (PHP)** |
|----------|-----------------|-------------|----------|----------------------|-----------------|
| **150 DAU** | 0.25 vCPU · 256 MB | $5 + $2.56 = **$7.56** | $25 | **~$33** | **~₱1,881** |
| **300 DAU** | 0.5 vCPU · 512 MB | $10 + $5 = **$15** | $25 | **~$40** | **~₱2,280** |
| 600 DAU | 0.5 vCPU · 1 GB | $10 + $10 = **$20** | $25 | **~$45** | **~₱2,565** |
| 1,000 DAU | 1 vCPU · 1 GB | $20 + $10 = **$30** | $25 | **~$55** | **~₱3,135** |

Egress: API JSON responses at 150–300 DAU will be well under the 25 GB/month included bandwidth — effectively **₱0 extra**.

### Render (flat monthly — no compute on top)

| Instance | DAU suitability | **Total/month (USD)** | **Total (PHP)** | Notes |
|----------|----------------|----------------------|-----------------|-------|
| Starter ($7) | **150–300 DAU** | **$7** | **~₱399** | 512 MB RAM; NestJS fits at this DAU with no async workers |
| Standard ($25) | **150–1,000 DAU** | **$25** | **~₱1,425** | 2 GB RAM; comfortable across full Phase 1 range |
| Pro ($85) | 1,000+ DAU | **$85** | **~₱4,845** | 4 GB RAM / 2 vCPU; significantly over-provisioned for Phase 1 |
| Pro Plus ($175) | High scale | **$175** | **~₱9,975** | Not applicable Phase 1 |

Bandwidth: 100 GB/month included on all paid tiers. API traffic at 150–300 DAU will not breach this.

---

## Side-by-Side Comparison

| Dimension | Render Starter | Render Standard | Render Pro | **Railway Pro** |
|-----------|---------------|-----------------|------------|-----------------|
| **Monthly cost (USD)** | $7 fixed | $25 fixed | $85 fixed | $25 base + ~$8–30 compute |
| **Est. cost at 150 DAU (USD)** | **$7** | **$25** | **$85** | **~$33** |
| **Est. cost at 300 DAU (USD)** | **$7** | **$25** | **$85** | **~$40** |
| **Est. cost at 1,000 DAU (USD)** | Risky (RAM limit) | **$25** | **$85** | **~$55** |
| **Monthly cost PHP (300 DAU)** | ~₱399 | ~₱1,425 | ~₱4,845 | ~₱2,280 |
| **RAM** | 512 MB (hard cap) | 2 GB (hard cap) | 4 GB (hard cap) | Configurable; scales to need |
| **vCPU** | 0.5 (hard cap) | 1 (hard cap) | 2 (hard cap) | Configurable; billed per second |
| **Bandwidth included** | 100 GB | 100 GB | 100 GB | **25 GB** |
| **Bandwidth overage rate** | $0.15/GB | $0.15/GB | $0.15/GB | $0.05/GB |
| **Billing model** | Flat — pay for max | Flat — pay for max | Flat — pay for max | Base + actual usage |
| **Overpay at low traffic** | Yes | Yes | Yes | Partial (base fee fixed; compute scales down) |
| **Horizontal autoscaling** | No | No | Yes | **Yes** |
| **Sleep / cold start** | No (paid) | No | No | No |
| **Singapore region** | Available | Available | Available | **Available** |
| **Chat support** | No | No | No | **Yes (Pro)** |
| **Workspace audit logs** | No | No | No | **Yes (Pro)** |
| **Team access** | $19/user extra | $19/user extra | $19/user extra | **Included per seat** |

---

## Total Monthly Stack Cost (NestJS server + Supabase + SMS)

At **150–300 DAU primary target**, combined with Supabase Pro (~$25/month ≈ ₱1,425) and Semaphore SMS (~₱500/month):

| Hosting choice | Server/month | Supabase Pro | SMS (est.) | **Monthly total** | **Annual total** |
|---------------|-------------|-------------|-----------|-------------------|-----------------|
| **Render Starter** (150–300 DAU) | ₱399 | ₱1,425 | ₱500 | **₱2,324** | **~₱27,888** |
| **Render Standard** (150–300 DAU) | ₱1,425 | ₱1,425 | ₱500 | **₱3,350** | **~₱40,200** |
| **Railway Pro** (150 DAU) | ₱1,881 | ₱1,425 | ₱500 | **₱3,806** | **~₱45,672** |
| **Railway Pro** (300 DAU) | ₱2,280 | ₱1,425 | ₱500 | **₱4,205** | **~₱50,460** |
| Render Pro (150–300 DAU) | ₱4,845 | ₱1,425 | ₱500 | **₱6,770** | **~₱81,240** |

**Extended reference — 1,000 DAU:**

| Hosting choice | Server/month | Supabase Pro | SMS (est.) | **Monthly total** |
|---------------|-------------|-------------|-----------|-------------------|
| Render Standard | ₱1,425 | ₱1,425 | ₱500 | **₱3,350** |
| Railway Pro | ₱3,135 | ₱1,425 | ₱500 | **₱5,060** |
| Render Pro | ₱4,845 | ₱1,425 | ₱500 | **₱6,770** |

---

## Key Tradeoffs

### Railway Pro — Advantages

- **Scales with actual traffic** — compute cost drops at 3am when no orders are coming in.
- **Horizontal autoscaling** — handles peak lunch/dinner bursts automatically without manual scaling.
- **No fixed RAM ceiling** — you're not capped at 512 MB on Starter or 2 GB on Standard if a busy day pushes past it.
- **Chat support included** — useful during initial launch instability.
- **Workspace audit logs** — team visibility on deployments.
- **Team access included in seat fee** — no extra $19/user/month.
- **Competitive egress** — $0.05/GB vs Render's $0.15/GB (matters at higher DAU).
- **Future database hosting** — if Supabase is ever replaced or supplemented, Railway can host Postgres natively within the same billing.

### Railway Pro — Disadvantages

- **More expensive at 150–300 DAU than Render** — ₱1,881–₱2,280/month vs ₱399–₱1,425/month on Render's fixed tiers.
- **Variable billing** — requires setting spend alerts and hard caps to avoid cost surprises from a runaway service.
- **Less included bandwidth** — 25 GB/month vs Render's 100 GB (not a real concern at current DAU, but worth noting).
- **Relatively newer platform** — less legacy documentation and third-party tutorials than Render.

### Render — Advantages

- **Cheapest option at 150–300 DAU** — Starter at $7/month (₱399) or Standard at $25/month (₱1,425) is significantly cheaper than Railway Pro for the primary launch target.
- **Fully predictable cost** — flat rate means no billing surprises.
- **Proven, mature platform** — extensive documentation and community resources.
- **100 GB bandwidth included** — 4× more than Railway Pro's 25 GB.

### Render — Disadvantages

- **No autoscaling on Starter or Standard** — if the lunch rush spikes beyond instance capacity, requests queue or error without manual intervention.
- **Pay for provisioned capacity even at idle** — Starter pays $7/month whether serving 1 or 500 requests/hour.
- **Team access costs extra** — $19/user/month workspace fee if multi-member dashboard needed.
- **Render Pro ($85/month) is wasteful** for this load — 4 GB RAM / 2 vCPU is 4–8× more than needed at 150–300 DAU.

---

## Cost Curve (150 DAU to 1,000 DAU)

```
Monthly server cost (USD)

$90 |                                         [Render Pro $85]──────────
    |
$55 |                                                     [Railway Pro ~$55]
$50 |
$45 |                                         [Railway Pro ~$45]
$40 |                    [Railway Pro ~$40]
$33 |  [Railway Pro ~$33]
$25 |  [Render Standard $25]────────────────────────────────────────────
    |
 $7 |  [Render Starter $7]────────────╮  (RAM risk at 600+ DAU)
    |
    └────────────────────────────────────────────────────────────────────
         150 DAU       300 DAU       600 DAU       1,000 DAU
```

Render Standard ($25) is flat and cheaper than Railway Pro across the entire 150–1,000 DAU range. Railway Pro only justifies itself through its feature set (autoscaling, support), not raw price.

---

## Recommendation

### At 150–300 DAU (Phase 1 Antique Province launch target)

**Render Standard at $25/month (~₱1,425) is the most cost-effective choice.**

Railway Pro costs **₱456–₱855/month more** than Render Standard at this load level, and the autoscaling benefit is marginal when peak concurrent requests are only 5–18 (well within a fixed 1 vCPU / 2 GB instance).

| Option | Cost at 300 DAU | Verdict |
|--------|----------------|---------|
| Render Starter ($7) | ~₱399/mo | Viable; tight RAM at peak. Use as staging env. |
| **Render Standard ($25)** | **~₱1,425/mo** | **Best value for Phase 1 launch** |
| Railway Pro (~$40) | ~₱2,280/mo | Overpays for features unused at this scale |
| Render Pro ($85) | ~₱4,845/mo | Do not use — severely over-provisioned |

### At 1,000 DAU (growth reference)

Render Standard ($25/month) **still beats Railway Pro (~$55/month)** on price. However, Railway Pro's **horizontal autoscaling** becomes genuinely valuable here — Render Standard (1 vCPU fixed) can start queuing requests at 45–65 concurrent users during a peak delivery window without manual intervention.

At 1,000 DAU, evaluate whether the autoscaling advantage of Railway ($55/month) is worth the ₱1,710/month premium over Render Standard ($25/month).

### If Railway was confirmed in the meeting

The meeting confirmed Railway as the preferred platform — that may account for non-cost factors (developer experience, database consolidation, future-proofing). If sticking with Railway:

- **Set a spend cap** of $60/month in Railway dashboard to catch runaway services.
- **Monitor the first 30 days** (target: backend launch ~2026-07-08) to validate actual compute costs against these estimates.
- **Do not provision idle resources** — Railway's per-second billing means you only pay for what runs, so right-size at 0.5 vCPU / 512 MB initially and scale up only if needed.

### Practical suggestion

Use **Render Standard as the production host at launch**, and **Railway as the staging environment** (lower traffic = lower Railway compute cost). Evaluate migrating production to Railway at the 600–1,000 DAU mark when autoscaling value becomes real.

---

## Related Documents

| Document | Purpose |
|----------|---------|
| [Hostinger KVM comparison](./README.md) | VPS option for web panel hosting |
| [strategy.md](../breakdown/strategy.md) | Confirmed stack decisions (Railway selected 2026-07-01) |
| [ARCHITECTURE.md](../../ARCHITECTURE.md) | Full system design and container map |
| [meeting-notes-2026-07-01.md](../../references/meeting-notes-2026-07-01.md) | Source of this action item |
