# GoTools Roadmap

**Status:** Draft for review
**Author:** Generated for PlayologyEntertainment / GoTools
**Date:** 2026-07-01
**Covers:** Version 1.0 recap → Version 2.0 direction

---

## 1. Version 1.0 — where we are today

GoTools shipped essentially the full catalog laid out in `TripTools_DesignDocument.html`, plus a data-integrity overhaul (`TripTools_DataConsolidation_Plan.md`, Phases 0–7, all ✅ done) and several platform features that weren't in the original spec at all. Consider everything below **v1.0, locked**.

### 1.1 The 77-tool catalog (`GoTools.html` → `TOOLS`)

| Phase | Tools | Count |
|---|---|---|
| 🔍 Discovery & Planning | Destination Picker, Destination Comparison, Best Time to Visit, Travel Style Quiz, Bucket List Builder, Trip Countdown, Trip Duration Calculator, Travel Budget Planner, Group Trip Organizer, Flight Cost Estimator, Hotel vs. Vacation Rental, Road Trip Route Planner, Itinerary Builder, Travel Insurance Worksheet | 14 |
| 📋 Documents & Logistics | Passport Expiry Checker, Visa Requirement Guide, Travel Document Checklist, International Driving Guide, Flight Delay Compensation, Customs Declaration Helper, Frequent Flyer Tracker, Vaccination Requirements, TSA & Security Guide | 9 |
| 🧳 Packing | Smart Packing List, Baggage Fee Calculator, Carry-On Size Checker, Luggage Weight Calculator, Weather-Based Wardrobe, Power Plug & Adapter Guide, What NOT to Pack | 7 |
| 💱 Money & Finance | Currency Converter, Tip Calculator, Daily Expense Tracker, Bill Splitter, ATM Fee Estimator, Cost of Living Comparison, VAT & Tax Refund Guide, Cruise Gratuity Calculator | 8 |
| 🌍 At Destination | Time Zone Converter, Jet Lag Calculator, Universal Unit Converter, Language Phrasebook, Public Holidays, eSIM & Mobile Data, Safety & Advisories, Emergency Numbers, UV Index & Sun Safety, Altitude Sickness Guide, Weather Forecast, Tipping Culture Guide, Driving & Road Rules, Local Quiet Hours Guide | 14 |
| 🚗 Road Trip | Fuel Cost Calculator, Rest Stop Planner, Road Trip Playlist Timer, Border Crossing Guide, Car Emergency Kit Checklist, Toll Cost Estimator | 6 |
| 🚢 Cruise | Shore Excursion Planner, Cruise Line Comparison, Sea Sickness Guide, Cruise Day Planner | 4 |
| 🎢 Theme Park | Theme Park Budget, Park Day Optimizer, Ride Height Checker, Park Dining Strategy | 4 |
| 💼 Business Travel | Per Diem Calculator, Expense Report Builder, Meeting Time Zone Planner, Airport Lounge Guide, Business Trip Packing | 5 |
| 🏠 Returning Home | Customs Duty Calculator, Trip Cost Recap, Souvenir & Gift Tracker, Post-Trip Health Checklist, Trip Journal, Review Drafter | 6 |
| **Total** | | **77** |

Three of these — **Public Holidays**, **eSIM & Mobile Data**, and **Safety & Advisories** — didn't exist in the original 75-tool design document. They were added as net-new data domains during the consolidation effort (design doc §Phase 7) and are now first-class tools in the hub.

### 1.2 The data foundation (finished, not to be re-litigated)

The single biggest v1.0 achievement isn't a tool — it's that GoTools no longer has ~14 independent, drifting country/city datasets. `TripTools_DataConsolidation_Plan.md` Phases 0–7 delivered:

- **`TT_COUNTRIES`** — one canonical registry, 202 countries/territories, identity + currency + calling code + languages + phrasebook link.
- **`TT_DESTINATIONS`** — one canonical destination list, 332 rows (113 curated "inspire" destinations + anchor cities for every remaining country), each linked to a country code.
- **`resolveDestination()`** — the one matcher every tool uses to turn `trip.destination` free text into a country/city, replacing ~11 bespoke regex/matcher implementations.
- **Full 202-country coverage** on Power Plug, Emergency Numbers, Visa, Vaccination, Customs, IDP, Driving Rules, Quiet Hours, Tipping, VAT Refund, Safety & Advisories, eSIM, Public Holidays, and Languages. **ATM Fees is the one intentional exception** — 165 countries have precise surcharge data, 37 volatile/cash-based economies carry a "bring cash" caveat instead of a false-precise number.
- A load-time self-check (`ttCountriesSelfCheck()`) that guards against identity drift and referential-integrity breaks reappearing.

### 1.3 Platform features beyond the original design doc

Built after the design document, on top of the 77-tool grid:

- **MyTrip profile** — the active-trip object ~20 tools read via a `trip:true` flag to auto-fill/auto-detect.
- **Trip Sync** — BYO GitHub personal-access-token, private Gist per trip, companion merge, reset-sync-code flow. Zero GoTools-hosted backend.
- **Trip Briefing** — an aggregating dashboard: countdown banner (with a "heading home" flip and a C/F toggle), itinerary box, customizable/scrollable boxes.
- **Favorites** — starred tools, surfaced as a gold category above the phase grid.

### 1.4 Explicit v1.0 principles (from the design doc — worth restating so v2.0 doesn't accidentally violate them)

- No accounts, no tracking, nothing leaves the browser unless a tool explicitly needs it.
- Not a booking engine — every flight/hotel/lounge tool is an *estimator or comparator*, never a live search or affiliate funnel.
- Offline-first: offline tools must never spinner/error; online tools must declare connectivity before use.
- Zero build step, zero framework, one HTML file per surface.

---

## 2. Version 2.0 — direction (confirmed with you)

Four decisions locked in before drafting the workstreams below:

1. **Architecture:** move from purely backend-free to **optional lightweight cloud sync** (opt-in account layer for cross-device sync/notifications), while every tool must remain fully usable with zero account — this is additive, not a replacement for the local-first model.
2. **Investment mix:** all four of — new applets, deepening existing tools, platform/UX, and data completeness.
3. **AI features:** yes, via **BYO API key** (same trust pattern as the existing Gist sync token — user supplies their own key, stored locally, called directly from the browser).
4. **New traveler segments** (pet travel, accessibility & mobility, digital nomad): **backlog/stretch only** — not committed v2.0 scope. Listed in §7 for later prioritization, not built against in the phases below.

---

## 3. Workstream A — New applets & unified destination view

Rather than bolting on isolated new tools, the highest-leverage move is consolidating the **At Destination** phase's 8 country-lookup tools (Emergency, Tipping, Driving Rules, Quiet Hours, Safety, eSIM, Holidays, Phrasebook) — which already all key off the same `TT_COUNTRIES`/`resolveDestination()` — into one skimmable view:

- **Destination Dossier** — a single-screen brief for the active trip's country (or any looked-up country) pulling one line each from Safety level, Emergency numbers, Tipping norms, Driving side, Quiet hours, eSIM status, next public holiday, and phrasebook language. Each line deep-links to its full tool. This is a natural extension of Trip Briefing, not a new dataset — pure UI/aggregation work.
- **EV / hybrid road trip support** — Fuel Cost Calculator currently assumes gas; add an EV mode (cost-per-kWh, charging stop planning) as a genuine gap inside the existing Road Trip phase.
- **International Customs Duty** — Customs Duty Calculator is US-only (the $800 exemption); extend to Canada/UK/EU/Australia duty-free allowances so non-US travelers get value from the Home phase.
- **Multi-trip history / "countries visited" map** — a new lightweight view (not a full tool) that reads past MyTrip records and renders a visited-countries map using the existing `TT_COUNTRIES` alpha-2 keys. No new data domain required.

---

## 4. Workstream B — Deepen existing tools

Phase-by-phase, concrete (not speculative) enhancements to tools that already exist:

- **Itinerary Builder** — `.ics` calendar export; per-day budget rollup tying into Travel Budget Planner instead of living separately.
- **Group Trip Organizer** — real-time voting via Trip Sync's existing Gist channel instead of local-only state.
- **Smart Packing List / Weather-Based Wardrobe** — these are two separate tools solving overlapping problems; unify their state so a forecast pull feeds the rules-based packing generator directly.
- **Park Day Optimizer** — layer in a live wait-time source (e.g., a free queue-times-style API) as a new *online* enhancement, consistent with the existing connectivity-badge pattern — falls back to the current static strategy guide when offline.
- **Trip Journal + Review Drafter** — combined "trip recap" export (journal entries + photos + spend summary) as a shareable artifact, feeding the gamification ideas in Workstream C.
- **Frequent Flyer Tracker / Passport Expiry Checker** — these already compute expiry dates; they just can't *alert* you. Pairs directly with the notification work in Workstream C.

---

## 5. Workstream C — Platform & UX

- **Installable PWA** — manifest + service worker for true offline caching (today "offline" means "no network call," not "installable/cacheable app shell").
- **Local notifications** — passport/visa/frequent-flyer-point expiry reminders and trip-countdown milestones, via the Notification API — works with **zero account**, so it ships independently of the cloud-sync decision.
- **Multi-trip dashboard** — past-trips archive, stats (countries visited, days traveled, total spent by rolling up Trip Cost Recap history), feeding the visited-countries map from Workstream A.
- **Light gamification** — milestone badges (countries visited, tools used across all 10 phases) — additive, cosmetic, no new data domain.
- **Optional cloud sync (the architecture decision from §2.1):** the current Trip Sync (BYO GitHub token + Gist) proved the sync *concept* works without a backend. Two credible paths to "optional lightweight cloud sync," in increasing order of commitment:
  - **C1 — Lower friction on the existing model.** Replace manual PAT entry with GitHub OAuth device flow. Still zero GoTools-hosted infrastructure.
  - **C2 — A minimal opt-in backend** (e.g., Cloudflare Workers + KV, or Supabase) with magic-link auth, used *only* for cross-device MyTrip sync and push delivery. Every tool must keep working with no account, per the v1.0 principle in §1.4.
  - **Recommendation:** treat this the way the data-registry work was treated — write a dedicated short design doc (architecture, auth provider, data retention, cost) before any code lands, given it's the one workstream that touches the "no accounts" principle. Don't start C2 without that doc reviewed.

---

## 6. Workstream D — AI-assisted features (BYO API key)

Mirrors the trust model already established by the Gist sync token: user pastes their own key (OpenAI/Anthropic/etc.) into a settings panel, it's stored in `localStorage` only, and calls go straight from the browser to the provider — no GoTools-side key, no proxy, no cost to us.

Candidate features, in order of value/risk:

1. **Smart Packing List, AI-augmented** — free-text trip description ("5 days in Tokyo in March with a toddler, mostly museums") augments, not replaces, the existing rules-based generator. Tool still works fully with zero key.
2. **Itinerary auto-draft** — seed a first-pass day-by-day plan from MyTrip + Bucket List + destination data, which the user then edits in the existing drag-and-drop Itinerary Builder.
3. **"Ask GoTools"** — a scoped Q&A box that answers *by summarizing the bundled `TT_COUNTRIES`/domain datasets*, not by freelancing. This is the important guardrail: for the legally/medically adjacent domains (Visa, Vaccination, Customs, Driving Rules, Safety & Advisories), the model must cite and restate the existing verified, dated, disclaimed data — never generate new claims — preserving the "best-effort, dated, disclaimed" bar the data-consolidation effort established.

---

## 7. Workstream E — Data completeness & quality

- **ATM Fees:** either close the remaining 37-country gap with clearly-labeled "approximate" figures, or formally document the caveat-only state as permanent-by-design (it's currently an implicit exception, not a documented one).
- **Recurring re-verification cadence:** several domains are inherently time-sensitive and will silently rot without a refresh cycle — GSA per-diem rates (published annually), airline baggage fees, Public Holidays' movable-feast tables (currently validated only through 2030), driving speed/BAC limits, visa rules. Recommend an annual "data refresh" pass per domain, tracked like the original per-domain PRs.
- **Deepen destination granularity below the country level** — `TT_DESTINATIONS` guarantees one anchor city per country (332 rows), but Cost of Living (41) and the curated Destination Picker (113) are much smaller than the full registry; consider growing those two specifically where user demand shows up (e.g., via search-miss telemetry, if any is ever added).

---

## 8. Backlog — new traveler segments (not committed to v2.0)

Explicitly deferred per your answer — listed here so they aren't lost, not because they're being built now:

- **Pet travel** — pet documents, in-cabin/cargo airline rules, international import requirements, pet-friendly lodging checklist.
- **Accessibility & mobility** — wheelchair-accessible destination notes, service-animal rules by country, traveling with medical equipment.
- **Digital nomad / remote work** — long-stay/digital-nomad visa guide, wifi-quality expectations, coworking/time-zone-overlap planning.

Any of these could become a v2.1+ phase once the committed workstreams above are further along.

---

## 9. Suggested phasing

Following the one-PR-per-phase cadence established by the data-consolidation effort:

| Phase | Workstream | Ships |
|---|---|---|
| v2.0-P0 | C | PWA install + service worker + local expiry/countdown notifications (no account required) |
| v2.0-P1 | A | Destination Dossier aggregation view |
| v2.0-P2 | B | Itinerary `.ics` export + budget rollup; Packing/Wardrobe state unification |
| v2.0-P3 | D | BYO-key settings panel + AI-augmented Smart Packing List |
| v2.0-P4 | E | ATM Fee gap closure + documented refresh cadence for time-sensitive domains |
| v2.0-P5 | A/B | EV road trip mode; international Customs Duty allowances |
| v2.0-P6 | C | Multi-trip history dashboard + countries-visited map + light gamification |
| v2.0-P7 | D | "Ask GoTools" scoped Q&A over bundled data |
| v2.0-Design | C2 | Dedicated cloud-sync design doc (architecture/auth/cost) — gates any work on optional accounts |

Cloud sync (C2) is deliberately last/gated: it's the one change that touches the "no accounts" principle, so it gets its own design doc — the same way the data registry got `TripTools_DataConsolidation_Plan.md` — before any implementation phase is scheduled against it.

---

## 10. Open questions

- **AI provider scope:** which providers does the BYO-key panel support at launch (OpenAI + Anthropic only, or broader)? Affects the settings UI and the "Ask GoTools" grounding prompt design.
- **Cloud sync auth provider:** magic-link email, GitHub OAuth (reusing the existing Trip Sync relationship), or something else — to be resolved in the C2 design doc, not here.
- **Telemetry:** several data-completeness recommendations above (e.g., "grow destination granularity where user demand shows up") assume some way to see what's missing. GoTools currently has zero analytics by design — worth an explicit decision on whether v2.0 introduces any, even anonymous/local-only.
