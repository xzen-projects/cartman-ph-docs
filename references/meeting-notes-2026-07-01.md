# Meeting Notes — Jul 1, 2026 at 20:30 PST

**Attendees:** Louie Dale Cervera, Benjamen Ford Cepe, Teddie John Rajeev, Franz Eliezer Samilo  
**Source:** Gemini AI notes from Google Meet recording

---

## Decisions

| Decision | Detail |
|----------|--------|
| Android initial release account | Franz Eliezer Samilo's Google Play Console account; transfer to Cartman PH account later |
| iOS access (Phase 1) | Web app as temporary solution while native iOS is finalized |
| Hosting provider | **Railway** — pay-as-you-go, supports databases, single Singapore server sufficient for PH user volume |
| Cartman Pro features | Deferred — focus on core operational functionality for initial launch |
| Phone number verification | Mandatory via SIM registration for all customer accounts (OTP) |
| COD ID validation | Valid ID **not** required at registration (skip button allowed); mandatory popup when user attempts to finalize a COD transaction |
| Rider drop-off points | Riders can input additional custom drop-off addresses beyond preloaded locations |
| Delivery fee structure | Global threshold-based system; managed as adaptive variables in admin dashboard (flag-down rate for 2 km radius baseline) |
| Support ticket + override | Platform implements support tickets; admins can perform password resets and auth bypasses after user confirmation |

---

## Architecture Confirmations

| Topic | Confirmed Detail |
|-------|-----------------|
| Mobile stack | **Flutter** (cross-platform, confirmed by Louie) |
| Mapping service | **OpenStreetMap** (confirmed; Google Maps API avoided due to geocoding costs) |
| Backend | NestJS hosted on **Railway** |
| Server region | Single server in **Singapore** — no CDN needed for current PH user volume |
| API security | Standard API keys + rate limiting to protect endpoints |
| Image caching | Flutter `cached_network_image` / image network package for merchant images |
| Docker | Development environment consistency only — not for production exports |

---

## Competitive Context

| Competitor | Context |
|------------|---------|
| Coco Express | Direct competitor in Antique Province |
| Food Panda | Competitor; also offers web-based ordering |
| Grab | Competitor in region |

Current volume: **300+ transactions/day** on existing platform.

---

## Timeline

| Milestone | Target |
|-----------|--------|
| Backend completion | ~2026-07-08 (1 week, Louie) |
| Testing phase start | ~2026-07-08 to 2026-07-10 |
| Launch window | 7–10 days from meeting (to prevent losing merchants/customers) |
| Meeting cadence | Every Monday, Wednesday, and Friday |

---

## Action Items

| Assignee | Task |
|----------|------|
| Benjamen Ford Cepe | Research Google/Apple organization account requirements to bypass 14-day testing |
| Louie Dale Cervera | Cost analysis: Render vs Railway (hosting) |
| Louie Dale Cervera | Connect backend to frontend within 1 week |
| Louie Dale Cervera | Research 2FA and email-based OTP implementation |
| Louie Dale Cervera | Break down delivery fee structure and pricing |
| Benjamen Ford Cepe | Implement COD ID verification logic |
| Benjamen Ford Cepe | Share brand kit (logos, marketing icons) |
| Teddie John Rajeev | Grant Louie Dale Cervera access to project environments |
| Teddie John Rajeev | Build support ticket system with admin override capability |
| The group | Prepare iOS web app (production-ready for Apple devices) |
| The group | Finalize Android app for Play Store deployment |
| The group | Advanced user profiling in admin dashboard |
| The group | Add drop-off points to rider mapping interface |
| The group | Implement application health monitoring |

---

## Additional Notes

- **Merchant order management:** Small local shops manage orders via mobile app; PDF printing for larger merchants is under investigation.
- **Rider allocation algorithm:** Weighted by distance, active order count, and rider success history; adaptive threshold rather than fixed limit.
- **Order grace period:** Merchants can cancel orders if items are out of stock; global variables manage rider penalties for repeated order declines.
- **Authentication:** 2FA and email OTP for password resets under investigation; email service free tier ~3,000 emails/month considered sufficient.
- **Operational dashboard:** Needed for monitoring active riders, system health, and end-to-end ops.
- **Pro plan / Cartman Pro:** Deferred to future phase; would include merchant promotional activities and higher platform visibility.
- **COD registration UX:** Skip button shown during initial registration profile; valid ID popup triggered only when user attempts to finalize a COD transaction.
