# Render Pro vs Railway — NestJS Backend Cost Comparison

**Scope:** NestJS API server (cartman-server) hosting only.  
**Primary target:** 150–300 daily active users (Antique Province Phase 1 launch).  
**Extended reference:** Up to 1,000 DAU (growth planning).  
**Context:** Supabase Cloud handles PostgreSQL, Auth, Realtime, Storage, and Edge Functions — the NestJS server handles REST API logic, FCM dispatch, Semaphore OTP relay, and courier fee calculation.  
**Action item owner:** Louie Dale Cervera (assigned 2026-07-01 meeting)  
**Exchange rate used:** ~₱57 / USD (verify current rate before budgeting)

---

## Platform Pricing at a Glance

### Railway — Plans and Compute Rates

> **Pricing discrepancy — verify before committing:**  
> Railway's official documentation (docs.railway.com) lists the Pro plan at **$20/month with $20 of compute credit included**. The Railway pricing page screenshot (captured Jul 2026) shows **$25/month + compute** as an additive charge. These produce materially different costs. Both models are calculated below. Confirm the current model at railway.com/pricing before signing up.

**Confirmed compute rates (both sources agree):**

| Resource | Rate |
|----------|------|
| CPU | $20 / vCPU / month ($0.000463/vCPU/min) |
| RAM | $10 / GB / month ($0.000231/GB/min) |
| Network egress | $0.05/GB |
| Volume storage | $0.15/GB/month |

**Plan structures in question:**

| Plan | Per docs (docs.railway.com) | Per pricing page screenshot (Jul 2026) |
|------|-----------------------------|----------------------------------------|
| Hobby | $5/month + $5 compute credit | — |
| **Pro** | **$20/month + $20 compute credit included** | **$25/month + compute (additive, no credit)** |

**Under the docs model ($20 + credit):** The $20 monthly fee acts as a prepaid compute credit. If your containers consume $15 in resources, you pay $20. If they consume $35, you pay $35 ($20 credit absorbs the first $20, $15 overage billed on top).

**Under the screenshot model ($25 + additive):** The $25/month is a seat fee. Compute costs are fully additional on top — e.g., $15 of compute = $25 + $15 = $40 total.

**Pro plan confirmed features** (from pricing page screenshot):
- All Hobby features, plus:
- No service maximum
- 25 GB bandwidth included
- Full-stack previews
- Horizontal autoscaling
- Isolated environments
- Private links
- Workspace audit logs
- AWS OIDC Integration (Beta)
- Chat support

**Hobby plan features** (baseline inherited by Pro, per docs):
- Up to 6 replicas, 48 GB RAM, 48 vCPU pool
- 5 GB volume storage
- 100 GB ephemeral storage
- 72-hour image retention

**Pro plan resource limits** (per docs):
- Up to 42 replicas, 1 TB RAM, 1,000 vCPU pool
- 1 TB volume storage (user-resizable)
- Unlimited image size, 120-hour image retention

**Sleep behavior:** None — containers run continuously while deployed.

---

### Render — Fixed Instance Tiers

Flat monthly rate per web service instance. Billing is prorated per second (only relevant if a service is created or destroyed mid-month; for a running production service it is effectively flat monthly). Always-on for all paid tiers.

| Instance | Monthly (USD) | Monthly (PHP) | RAM | vCPU | Bandwidth included |
|----------|--------------|---------------|-----|------|-------------------|
| Free | $0 | ₱0 | 512 MB | 0.1 vCPU | 100 GB | Sleeps after 15 min idle (~30s cold start) |
| **Starter** | **$7** | **~₱399** | 512 MB | 0.5 vCPU | 100 GB |
| **Standard** | **$25** | **~₱1,425** | 2 GB | 1 vCPU | 100 GB |
| **Pro** | **$85** | **~₱4,845** | 4 GB | 2 vCPU | 100 GB |
| Pro Plus | $175 | ~₱9,975 | 8 GB | 4 vCPU | 100 GB |
| Pro Max | $225 | ~₱12,825 | — | — | — |
| Pro Ultra | $450 | ~₱25,650 | — | — | — |

> Pro Max and Pro Ultra are not applicable to Phase 1. Listed for completeness.

**Bandwidth overage:** $0.15/GB beyond 100 GB/month  
**Persistent disk:** $0.25/GB/month  
**Team workspace (optional):** $19/user/month for advanced team access and permissions  
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

### Railway — Model A: $20/month + $20 credit (per official docs)

Under this model, the $20 monthly fee includes $20 of compute credit. You only pay extra when compute exceeds $20.

| Scenario | Est. compute | Credit covers | Overage | **Total/month (USD)** | **Total (PHP)** |
|----------|-------------|--------------|---------|----------------------|-----------------|
| **150 DAU** (0.25 vCPU · 256 MB) | ~$7.56 | Full coverage | $0 | **$20** | **~₱1,140** |
| **300 DAU** (0.5 vCPU · 512 MB) | ~$15 | Full coverage | $0 | **$20** | **~₱1,140** |
| 600 DAU (0.5 vCPU · 1 GB) | ~$20 | Full coverage | $0 | **$20** | **~₱1,140** |
| 1,000 DAU (1 vCPU · 1 GB) | ~$30 | $20 covered | $10 | **$30** | **~₱1,710** |

### Railway — Model B: $25/month + compute additive (per pricing page screenshot)

Under this model, the $25 is a base seat fee. All compute costs are added on top.

| Scenario | Est. compute | Base fee | **Total/month (USD)** | **Total (PHP)** |
|----------|-------------|----------|----------------------|-----------------|
| **150 DAU** (0.25 vCPU · 256 MB) | ~$7.56 | $25 | **~$33** | **~₱1,881** |
| **300 DAU** (0.5 vCPU · 512 MB) | ~$15 | $25 | **~$40** | **~₱2,280** |
| 600 DAU (0.5 vCPU · 1 GB) | ~$20 | $25 | **~$45** | **~₱2,565** |
| 1,000 DAU (1 vCPU · 1 GB) | ~$30 | $25 | **~$55** | **~₱3,135** |

**The difference between models at 150–300 DAU: ₱741–₱1,140/month.** Verify the active billing model before committing.

### Render (flat monthly — no compute on top)

| Instance | DAU suitability | **Total/month (USD)** | **Total (PHP)** | Notes |
|----------|----------------|----------------------|-----------------|-------|
| Starter ($7) | **150–300 DAU** | **$7** | **~₱399** | 512 MB RAM; NestJS fits at this load with no async workers |
| Standard ($25) | **150–1,000 DAU** | **$25** | **~₱1,425** | 2 GB RAM; comfortable across the full Phase 1 range |
| Pro ($85) | 1,000+ DAU | **$85** | **~₱4,845** | 4 GB / 2 vCPU; over-provisioned for Phase 1 |

Bandwidth: 100 GB/month included. API JSON traffic at 150–300 DAU will not breach this.

---

## Side-by-Side Comparison

| Dimension | Render Starter | Render Standard | Render Pro | Railway (Model A) | Railway (Model B) |
|-----------|---------------|-----------------|------------|-------------------|-------------------|
| **Base fee** | $7 fixed | $25 fixed | $85 fixed | $20 + credit | $25 + compute |
| **Est. cost at 150 DAU** | **$7** | **$25** | **$85** | **$20** | **~$33** |
| **Est. cost at 300 DAU** | **$7** | **$25** | **$85** | **$20** | **~$40** |
| **Est. cost at 1,000 DAU** | Risky (RAM) | **$25** | **$85** | **$30** | **~$55** |
| **300 DAU cost (PHP)** | ~₱399 | ~₱1,425 | ~₱4,845 | ~₱1,140 | ~₱2,280 |
| **RAM** | 512 MB (fixed) | 2 GB (fixed) | 4 GB (fixed) | Configurable | Configurable |
| **vCPU** | 0.5 (fixed) | 1 (fixed) | 2 (fixed) | Configurable | Configurable |
| **Bandwidth included** | 100 GB | 100 GB | 100 GB | 25 GB | 25 GB |
| **Bandwidth overage rate** | $0.15/GB | $0.15/GB | $0.15/GB | $0.05/GB | $0.05/GB |
| **Billing model** | Flat | Flat | Flat | Credit absorbs compute | Seat fee + compute |
| **Overpay at idle** | Yes | Yes | Yes | No (credit covers low usage) | Partial (base fixed) |
| **Horizontal autoscaling** | No | No | Yes | Yes | Yes |
| **Sleep / cold start** | No (paid) | No | No | No | No |
| **Singapore region** | Yes | Yes | Yes | Yes | Yes |
| **Chat support** | No | No | No | Yes (Pro) | Yes (Pro) |
| **Workspace audit logs** | No | No | No | Yes (Pro) | Yes (Pro) |
| **Team access** | $19/user extra | $19/user extra | $19/user extra | Included | Included |

---

## Total Monthly Stack Cost (NestJS server + Supabase + SMS)

At **150–300 DAU primary target**, combined with Supabase Pro (~$25/month ≈ ₱1,425) and Semaphore SMS (~₱500/month):

### Under Railway Model A ($20 + credit) — best case

| Hosting choice | Server/month | Supabase Pro | SMS (est.) | **Monthly total** | **Annual total** |
|---------------|-------------|-------------|-----------|-------------------|-----------------|
| Render Starter | ₱399 | ₱1,425 | ₱500 | **₱2,324** | **~₱27,888** |
| **Railway Pro** (150–600 DAU) | ₱1,140 | ₱1,425 | ₱500 | **₱3,065** | **~₱36,780** |
| Render Standard | ₱1,425 | ₱1,425 | ₱500 | **₱3,350** | **~₱40,200** |
| Railway Pro (1,000 DAU) | ₱1,710 | ₱1,425 | ₱500 | **₱3,635** | **~₱43,620** |
| Render Pro | ₱4,845 | ₱1,425 | ₱500 | **₱6,770** | **~₱81,240** |

### Under Railway Model B ($25 + compute additive) — screenshot pricing

| Hosting choice | Server/month | Supabase Pro | SMS (est.) | **Monthly total** | **Annual total** |
|---------------|-------------|-------------|-----------|-------------------|-----------------|
| **Render Starter** (150–300 DAU) | ₱399 | ₱1,425 | ₱500 | **₱2,324** | **~₱27,888** |
| **Render Standard** (150–300 DAU) | ₱1,425 | ₱1,425 | ₱500 | **₱3,350** | **~₱40,200** |
| Railway Pro (150 DAU) | ₱1,881 | ₱1,425 | ₱500 | **₱3,806** | **~₱45,672** |
| Railway Pro (300 DAU) | ₱2,280 | ₱1,425 | ₱500 | **₱4,205** | **~₱50,460** |
| Render Pro (any DAU) | ₱4,845 | ₱1,425 | ₱500 | **₱6,770** | **~₱81,240** |

**Extended reference — 1,000 DAU:**

| Hosting choice | Server/month | Supabase Pro | SMS | **Monthly total** |
|---------------|-------------|-------------|-----|-------------------|
| Render Standard | ₱1,425 | ₱1,425 | ₱500 | **₱3,350** |
| Railway Model A | ₱1,710 | ₱1,425 | ₱500 | **₱3,635** |
| Railway Model B | ₱3,135 | ₱1,425 | ₱500 | **₱5,060** |
| Render Pro | ₱4,845 | ₱1,425 | ₱500 | **₱6,770** |

---

## Key Tradeoffs

### Railway Pro — Advantages

- **Scales with actual traffic** — compute drops at 3am; you're not paying for a fully provisioned instance while idle (especially on Model A).
- **Horizontal autoscaling** — handles lunch/dinner peak bursts automatically.
- **No fixed RAM ceiling** — not capped at 512 MB or 2 GB if a busy day pushes past it.
- **Chat support included** — valuable during initial launch instability.
- **Workspace audit logs** — team visibility on deployments and incidents.
- **Team access included in seat fee** — no extra $19/user/month.
- **Competitive egress** — $0.05/GB vs Render's $0.15/GB.
- **Postgres on same platform** — if Supabase is ever supplemented, Railway can host it within the same billing.

### Railway Pro — Disadvantages

- **Pricing model needs verification** — docs and screenshot disagree. At Model B ($25 + additive), Railway is the most expensive option at 150–300 DAU.
- **Variable billing** — requires setting hard spend caps to avoid surprises from a misconfigured service.
- **Less included bandwidth** — 25 GB/month vs Render's 100 GB (not a concern at current DAU).
- **Newer platform** — fewer third-party tutorials and community support than Render.

### Render — Advantages

- **Cheapest option regardless of model** — Starter ($7) or Standard ($25) beats Railway Pro under either pricing interpretation at 150–300 DAU.
- **Fully predictable billing** — flat rate, no surprises.
- **4× more bandwidth included** — 100 GB vs Railway's 25 GB.
- **Proven, mature platform** — broader documentation and community resources.

### Render — Disadvantages

- **No autoscaling on Starter or Standard** — fixed instance; a lunch rush spike that exceeds the instance limit queues or errors without intervention.
- **Pay for provisioned capacity 24/7** — Starter pays $7/month whether serving zero or 500 requests/hour.
- **Team access costs extra** — $19/user/month workspace fee if multi-member dashboard is needed.
- **Render Pro ($85/month) is wasteful** — 4 GB RAM / 2 vCPU is 4–8× more than needed for this load.

---

## Cost Curve (150 DAU to 1,000 DAU)

```
Monthly server cost (USD)

$90 |                                           [Render Pro $85]─────────
    |
$55 |                                                       [Railway B ~$55]
$45 |                              [Railway B ~$45]
$40 |               [Railway B ~$40]
$33 |  [Railway B ~$33]
$30 |                                                       [Railway A $30]
$25 |  [Render Standard $25]──────────────────────────────────────────────
$20 |  [Railway A $20]──────────────────────────╮
    |                                            │ (overage kicks in)
 $7 |  [Render Starter $7]──────────╮
    |                                └── (RAM risk at 600+ DAU)
    └─────────────────────────────────────────────────────────────────────
          150 DAU        300 DAU        600 DAU        1,000 DAU

  A = Railway docs model ($20 + $20 credit)
  B = Railway screenshot model ($25 + compute additive)
```

---

## Recommendation

### At 150–300 DAU (Phase 1 Antique Province launch target)

| Option | Cost at 300 DAU | Verdict |
|--------|----------------|---------|
| Render Starter ($7) | ~₱399/mo | Viable; tight RAM at peak — best as staging environment |
| **Render Standard ($25)** | **~₱1,425/mo** | **Best value — safe choice regardless of Railway model** |
| Railway Pro — Model A ($20) | ~₱1,140/mo | Cheapest if docs model is correct; verify before signing up |
| Railway Pro — Model B ($40) | ~₱2,280/mo | Overpays for features unused at this scale |
| Render Pro ($85) | ~₱4,845/mo | Do not use — severely over-provisioned |

**If Railway billing model can be confirmed as Model A (credit-included):** Railway Pro at ~₱1,140/month becomes the best value — cheaper than Render Standard and includes autoscaling.

**If Railway billing model is Model B (screenshot, additive):** Render Standard at ₱1,425/month is better value for 150–300 DAU. Railway only wins on feature set (autoscaling, support), not price.

**Safe default without confirming:** Use **Render Standard ($25/month)**. It is the most predictable and cost-effective option that is not dependent on resolving the Railway pricing ambiguity.

### At 1,000 DAU (growth reference)

- **Railway Model A ($30/month)** edges out Render Standard ($25/month) slightly in cost while adding autoscaling — worth switching to Railway at this scale if the Model A pricing is confirmed.
- **Railway Model B ($55/month)** is ₱1,710/month more than Render Standard — only justified if autoscaling is operationally critical.

### If Railway was confirmed in the meeting

Non-cost factors (developer experience, future Postgres consolidation, autoscaling) likely drove that preference. If proceeding with Railway:

- **Confirm the billing model immediately** (Model A vs B) by creating a test service and checking the first invoice.
- **Set a hard spend cap** of $60/month in Railway's billing settings.
- **Start right-sized:** 0.5 vCPU / 512 MB RAM. Scale up only after 30 days of production data.
- **Monitor the first billing cycle** (~2026-07-08 backend launch target) against these estimates.

### Practical setup suggestion

| Environment | Host | Cost | Reasoning |
|-------------|------|------|-----------|
| Production | Render Standard | ~₱1,425/mo | Predictable; sufficient for Phase 1 |
| Staging / dev | Railway Hobby | ~₱285/mo ($5) | Low traffic = low compute bill; doubles as Railway familiarity |

Evaluate moving production to Railway at 600–1,000 DAU when autoscaling value becomes real.

---

## Related Documents

| Document | Purpose |
|----------|---------|
| [Hostinger KVM comparison](./README.md) | VPS option for web panel hosting |
| [strategy.md](../breakdown/strategy.md) | Confirmed stack decisions (Railway selected 2026-07-01) |
| [ARCHITECTURE.md](../../ARCHITECTURE.md) | Full system design and container map |
| [meeting-notes-2026-07-01.md](../../references/meeting-notes-2026-07-01.md) | Source of this action item |
