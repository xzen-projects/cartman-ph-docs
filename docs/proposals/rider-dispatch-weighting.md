# Rider Dispatch Weighting — Delivery + Pasakay

**Status:** Implemented (Phase 1) — `cartman-server/src/orders/feed.service.ts`. A single-factor-sum, advisory ranked feed (`GET /orders/feed`) shipped instead of Phase 2, with env-tunable weights. See the delta table below for what shipped vs. what's still proposal-only.

**Origin:** Write-up of the "Rider allocation algorithm" idea noted in [meeting-notes-2026-07-01.md](../../references/meeting-notes-2026-07-01.md) ("Weighted by distance, active order count, and rider success history; adaptive threshold rather than fixed limit").

## Proposal vs. shipped

| Aspect | This proposal | Shipped (Phase 1, `feed.service.ts`) |
|---|---|---|
| Mechanism | Staggered waves — ranked candidates revealed to a quantile band at a time, expanding on timeout (§4) | **Single ranked feed** — `GET /orders/feed` returns the full top-20 candidate list at once, ordered by score; no wave gating |
| Score formula | 5 factors: `active_workload`, `daily_throughput`, `proximity`, `trip_length`, `reliability`, × `type_modifier` (§3) | **4 factors**: age (0.40) + proximity (0.30) + payout (0.20) + type weight (0.10) — no workload/throughput/trip-length/reliability terms |
| Weight tuning | Admin-configurable `w1..w5` + `type_modifier` via a dashboard surface | **Env vars** (`FEED_W_AGE`, `FEED_W_PROX`, `FEED_W_PAYOUT`, `FEED_W_TYPE`) — no admin UI, no per-type modifier |
| Candidate pool filters | Hard filters: on-duty, verified, not wallet-locked, max proximity radius, stacking cap (§2) | On-duty + wallet-lock check happen at **claim** time, not feed-scoring time; no max-radius filter (proximity is a score term, not a hard cutoff) beyond the 10 km scoring range; no stacking-cap filter in the feed query itself (queue-depth cap 2 is enforced at claim) |
| `reliability` factor (decline tracking) | Proposed — requires persisting decline events server-side (§3, "Dependency this proposal introduces") | **Not built** — declines remain local-only (Hive), never sent to the server |
| `pasakay`/`ride` type | Proposed as a same-shape addition to `order_type` (§1) | **Already shipped independently** — `ride` is a real `order_type` enum value with `TYPE_WEIGHTS.ride = 1.0` (highest priority) in `feed.service.ts` |
| Claim mechanic | Unchanged — race-safe DB update (§ Relationship below) | **Unchanged as designed** — `PATCH /orders/:id/claim`, conditional `updateMany`, first-writer-wins |

Everything below this point (§1–§6) is the **original proposal as written** — it describes the staggered-wave, 5-factor design that was *not* built. Read it as "what a Phase 2 iteration could look like," not as a description of the shipped feed.

**Relationship to current system:** The claim mechanic is unchanged from what this proposal assumed: whoever taps first wins via a race-safe conditional update (see [flows.md § Rider Order Claim](../breakdown/flows.md#rider-order-claim-race-condition), target latency <50ms; implemented in `cartman-server`, not a raw client SQL statement). What *did* change from the "broadcast to every on-duty rider at once" baseline this proposal was written against: the feed is no longer a flat broadcast — `GET /orders/feed` now returns a **weighted-ranked** list (§ shipped column above), though as a single list, not the staggered waves this proposal describes. This proposal does **not** replace the claim mechanic. It (still, as originally proposed) describes a layer in front of it that would control *who sees the order and when* — the shipped feed is a simpler version of that layer, using ranking instead of wave-gated visibility. The underlying claim stays a race-safe DB update either way — weighting only ever affects visibility/ordering, never grants an order outright.

---

## 1. Scope

Covers order-to-rider distribution for all dispatchable order types:

| Order type | Job shape |
|---|---|
| `food`, `errand`, `courier` | Pickup point → dropoff point, rider carries a parcel |
| `pasakay` (hitch-a-ride) | Pickup point → dropoff point, rider carries a passenger |

Pasakay is not yet in the `orders.order_type` enum or anywhere else in `schema.md` — this proposal treats it as a same-shape addition to that enum (pickup/dropoff coordinates, same status progression), distinguished from parcel types only by dispatch weighting behavior (§3) and stacking rules (§2).

**Out of scope:** batched/look-ahead matching across multiple pending orders, surge pricing, cross-province dispatch. See Anti-Patterns (§5) for why batching is named but not solved here.

---

## 2. Candidate pool (hard filters)

Before any weight is computed, the same eligibility filtering that exists today narrows the pool. A rider must satisfy all of:

- On-duty, in the order's service zone (existing rider-feed filter).
- `verification_status = approved`.
- Not wallet-locked (`W_r` above `wallet_lockout_threshold`).
- Within the order type's **max proximity radius** (new, admin-configurable per type — tighter for `pasakay`/`food`, looser for `courier`/`errand`).
- Not at their **stacking cap** (new, admin-configurable max concurrent active orders per rider; riders may hold multiple parcel deliveries concurrently, but a `pasakay` order can never be stacked with anything else — a passenger occupies the whole trip).

Only riders passing all filters are scored. This keeps proximity as a hard boundary rather than something a weighted sum could be outvoted on (see §5, "Greedy nearest-only assignment").

---

## 3. Weight factors

Each factor normalizes to `[0, 1]`, where **0 is best** (most available/suitable). The total score is a weighted sum; **lowest score = highest priority**.

```
score = w1·active_workload + w2·daily_throughput + w3·proximity
      + w4·trip_length + w5·reliability
      × type_modifier(order_type)
```

All `w*` weights and `type_modifier` values are admin-configurable (mirrors the existing `delivery_fee_thresholds`-style adaptive config pattern), not hardcoded.

| Factor | Definition | Source data |
|---|---|---|
| `active_workload` | Rider's current active order count ÷ their stacking cap | `orders` where `assigned_rider_id = rider` and status not terminal |
| `daily_throughput` | Orders completed today ÷ a rolling daily norm, reset at midnight PH time | `orders.status = delivered`, `delivered_at` today |
| `proximity` | Normalized ETA from rider's last location ping → order's pickup point | `rider_location_logs` (most recent row) |
| `trip_length` | Normalized estimated pickup → dropoff duration | order's `pickup_coords`/`dropoff_coords` |
| `reliability` | Normalized recent decline rate (declines ÷ offers, trailing window) | **new** — see dependency note below |
| `type_modifier` | Per-order-type multiplier, applied most heavily to `proximity` | admin config, keyed by `order_type` |

**Dependency this proposal introduces:** `reliability` requires decline events to exist server-side. `domains.md` currently states declines are local-only (Hive/AsyncStorage on the rider app, no server write). Implementing this factor requires persisting at least a decline counter server-side — called out here as an open prerequisite, not assumed to already exist.

### 3.1 Edge case: proximity vs. trip length on the same order

These are kept as **separate factors**, not merged into one "distance" number, because they answer different questions and conflating them produces the wrong assignment in either direction:

- **Proximity** answers "how long until the rider reaches pickup" — this is a near-hard UX constraint (food goes cold, a passenger waits at the curb), so it's enforced primarily via the max-radius **filter** in §2, not left to the weighted sum to possibly overrule.
- **Trip length** answers "how long will this order occupy the rider" — it does not gate eligibility for *this* order (a nearby rider with a long haul ahead is still the correct match). Instead it feeds an **estimated-occupied-until** projection, combined with `active_workload`, that lowers that rider's priority for the *next* dispatch while they're mid-trip. The fairness effect surfaces on the following assignment, not this one.
- **`type_modifier` skews this per type:** `pasakay` weights `proximity` more heavily than parcel types (a live passenger's wait tolerance is lower than a bag of groceries), while `courier` tolerates a larger radius and lower proximity weight since cross-town trips are expected.

---

## 4. Staggered-wave dispatch mechanism

1. **Pool** — apply §2 filters to get eligible candidates.
2. **Score** — compute §3's weighted sum per candidate using their latest location ping and current order state.
   - **Staleness guard:** a rider whose last location ping is older than a threshold (e.g. 2 minutes) is scored as if farther away, so a stale GPS row can't win a wave they're no longer actually near.
3. **Bucket into waves** — sort ascending by score, split into quantile bands (e.g. lowest ~20% → Wave 1, next ~20% → Wave 2, ...).
4. **Reveal Wave 1 only** — only those riders see the order in their feed. Everyone else does not yet.
5. **Race-claim within the wave** — unchanged from today: first `UPDATE ... WHERE assigned_rider_id IS NULL` wins.
6. **Expand on timeout** — if unclaimed after a short window (e.g. 8–12s), Wave 2 is revealed *additionally* (cumulative, Wave 1 stays visible), and so on.
7. **Exhaustion fallback** — if the entire pool times out unclaimed, the order falls through to the existing admin/ops manual-assign queue (`ARCHITECTURE.md` §10.3 "Manual intervention").

Scoring must stay off the realtime claim hot path (today's target: <50ms) — waves are computed once at dispatch trigger time (e.g. when an order flips to `ready_for_pickup`, or on `pasakay` request creation), not recomputed per claim attempt.

---

## 5. Anti-patterns

Failure modes this design is explicitly built to avoid, or knowingly defers:

1. **Greedy nearest-only assignment** — optimizing per-order proximity in isolation degrades fleet-wide efficiency over time. Avoided by treating proximity as an eligibility filter (§2), not the dominant score term.
2. **Rider starvation** — riders consistently farther from hotspots, or with more logged deliveries, never surface. Mitigated by `daily_throughput` fairness with a daily reset.
3. **Notification storm / thundering herd** — revealing to too many riders at once causes claim contention. Wave-based reveal narrows the "at once" pool; wave *expansion* transitions still need their own debounce so they don't recreate the storm at each timeout boundary.
4. **Stale-score race** — a score is a snapshot; rider state (location, active orders) can change between scoring and claim. Never treated as a reservation — the underlying claim remains a race-safe DB update regardless of how scoring was computed.
5. **Metric gaming** — toggling on/off duty to reset counters, or idling near merchant hotspots purely to look "available." Mitigated by calendar-day resets (not session-based) and by keeping `daily_throughput` based on completed orders, not presence.
6. **Cold-start penalty** — a brand-new rider has no `reliability` history and must default to a neutral/mid score, not the worst-case score, or they'd never get selected.
7. **Single-factor dominance** — one factor overwhelming the sum causes degenerate routing (same rider always wins). Weights are bounded, normalized, and admin-tunable; live score distributions should be reviewed periodically, not set once and forgotten.
8. **Non-deterministic tie-breaks** — identical scores need a deterministic tie-break (e.g. hash of `rider_id + order_id`), not re-rolled randomness, to avoid flaky or unfair repeat selection.
9. **No look-ahead / batch neglect** — scoring one order at a time ignores the rest of the pending queue (a classic online bipartite-matching limitation). Named here as a known limitation for potential future batched-matching work; not solved by this proposal.
10. **Latency budget creep** — today's claim path targets <50ms; scoring must stay pre-computed/async relative to that path, or it risks blowing the budget under peak load (300+ transactions/day today, per meeting notes, and growing).
11. **Type-blind mixing** — a shared queue that isn't type-aware lets short parcel jobs systematically outscore `pasakay` (or vice versa). This is what `type_modifier` (§3) exists to prevent.

---

## 6. Open dependencies / follow-ups

- Persist rider decline events server-side (currently local-only) — required for `reliability` (§3).
- Add `pasakay` to `orders.order_type` and define its status progression (assumed identical to existing types per §1, but not yet ratified in `schema.md`).
- Admin dashboard surface for tuning `w1..w5`, `type_modifier`, max proximity radius, and stacking cap per order type.
- Decide wave count, quantile band sizes, and per-wave timeout — placeholder values above (~20% bands, 8–12s) need validation against real order volume once Phase 2 planning starts.
