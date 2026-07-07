# SMS Gateway Cost Breakdown — Semaphore vs Alternatives

**Scope:** OTP/2FA SMS delivery for customer + rider phone verification (C-1.2).
**Current implementation:** Semaphore, called via raw HTTP `fetch()` from `cartman-server/src/auth/otp.service.ts` (no SDK) and mirrored in `cartman-mobile/supabase/functions/otp-send`.
**Exchange rate used:** ~₱57 / USD (verify current rate before budgeting).
**Sources:** Provider pricing pages fetched 2026-07-02 — verify before large commitments, providers change rates without notice.

---

## Semaphore (current provider)

| Item | Rate |
|---|---|
| Cost per credit | **₱0.56**, VAT-inclusive |
| Regular SMS (`/api/v4/messages` — what `otp.service.ts` calls today) | 1 credit / 160 chars → **₱0.56/message** |
| Priority SMS (`/api/v4/priority`) | 2 credits → **₱1.12/message**, bypasses the 120 calls/min rate limit |
| Dedicated OTP endpoint (`/api/v4/otp`) | 2 credits → **₱1.12/message**, built-in `{otp}` templating, no rate limit |
| Long messages (>160 chars) | `ceil(length / 153)` credits |
| Sender name | 1st free; extra names ₱112/mo (200 credits) or ₱1,120/yr (2,000 credits) |
| Minimum purchase | None — pay-as-you-go |
| Payment method | Bank deposit (Unionbank); credits applied on receipt of deposit slip |

**Note:** the current implementation intentionally uses the cheaper `/messages` endpoint (1 credit) and relies on `OtpService`'s own in-app rate limit (3 codes / 15 min per identifier, see `otp.service.ts:17-18`) rather than paying 2x for the dedicated OTP endpoint's built-in rate-limit bypass. This is a reasonable tradeoff at current volume.

---

## Alternatives evaluated

| Provider | Rate to PH numbers | Notes |
|---|---|---|
| **Semaphore** (current) | ₱0.56/SMS | PH-local, already integrated, no minimum top-up |
| **PhilSMS** | **Starts at ₱0.35/SMS**, no minimum top-up | PH-local; free sender ID registration (vs. Semaphore charging ₱112+/mo for extras); credits valid 1 year; full tiered pricing table is dashboard-gated (requires free account signup to view) |
| **iTexMo** | Not retrievable at time of writing | Long-standing PH gateway commonly cited alongside Semaphore/PhilSMS; pricing page did not return content — check manually if evaluating |
| **Twilio** | ~$0.241 (**≈₱13.70**) per SMS to PH mobiles, + $0.001 failed-message fee, + $120/mo for a PH mobile number lease | ~24x more expensive than Semaphore. Already ruled out in [ARCHITECTURE.md §5](../../ARCHITECTURE.md) in favor of a PH-local gateway |

---

## Monthly cost by OTP volume

Cartman's rate limit caps each phone/email at 3 codes per 15 minutes (`otp.service.ts:17-18`), so cost scales with unique verification events (signups + re-verifications), not raw traffic.

| OTPs/month | Semaphore (₱0.56, current) | PhilSMS (₱0.35, if switched) | Twilio (₱13.70, for reference) |
|---|---|---|---|
| 100 (soft pilot) | ₱56 | ₱35 | ₱1,370 |
| 500 | ₱280 | ₱175 | ₱6,850 |
| 1,000 | ₱560 | ₱350 | ₱13,700 |
| 3,000 | ₱1,680 | ₱1,050 | ₱41,100 |
| 5,000 | ₱2,800 | ₱1,750 | ₱68,500 |

For a Phase 1 Antique Province pilot (few hundred active users, sign-up + occasional re-verification), realistic volume is **100–1,000 OTPs/month** → **₱56–₱560/mo** on Semaphore, or **₱35–₱350/mo** on PhilSMS.

---

## Recommendation

**Stay on Semaphore for now.** The integration is already built, tested, and deployed (`otp.service.ts`, `otp-send` Edge Function). PhilSMS is ~37% cheaper per message, but at this volume (under ₱600/mo either way) the savings don't justify a provider migration and re-verification of deliverability on PH carrier networks. Revisit if monthly OTP volume grows past ~5,000/mo (~₱1,050/mo savings potential) or if Semaphore deliverability/support becomes a problem.

If switching is ever warranted, the migration is small: `otp.service.ts`'s `sendSemaphoreSms` is a single private method behind a raw `fetch()` call — a `sendPhilSmsSms` sibling would follow the identical pattern (see `sendResendEmail` in the same file for a recent example of adding a second raw-HTTP provider without an SDK dependency).

---

## Related documents

| Document | Purpose |
|---|---|
| [README.md](./README.md) | Hostinger VPS sizing + full stack cost (uses a placeholder ₱200–800/mo SMS estimate — this doc supersedes that with sourced numbers) |
| [render-vs-railway.md](./render-vs-railway.md) | `cartman-server` hosting cost comparison (also uses a placeholder ₱500/mo SMS estimate) |
| [total-app-cost-breakdown.md](./total-app-cost-breakdown.md) | Full consolidated cost of the entire Cartman PH stack |
