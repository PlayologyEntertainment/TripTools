# GoTools Roadmap

**Status:** Revised — real priorities set
**Author:** Generated for PlayologyEntertainment / GoTools
**Date:** 2026-08-04 (supersedes the 2026-07-01 draft)
**Covers:** Version 1.0 recap → what shipped since → Version 2.0 priority order

**Revision note:** This is a working update, not a rewrite from scratch. Section 1 (v1.0) is unchanged.
Section 2 documents what actually shipped in the month since the original draft — notably, cloud sync
landed already, via a different path than the original plan sketched. Sections 3+ re-prioritize
everything still open, based on decisions made with the owner on 2026-08-04 (recorded inline).

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

**Confirmed still 77 as of this revision** (verified directly against the `TOOLS` array in `GoTools.html`) — no new applets have shipped since the last draft; see §2 for what *has* shipped instead.

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
- **Trip Briefing** — an aggregating dashboard: countdown banner (with a "heading home" flip and a C/F toggle), itinerary box, customizable/scrollable boxes.
- **Favorites** — starred tools, surfaced as a gold category above the phase grid.
- **Trip Sync — now Google Sign-In + Drive.** Originally BYO GitHub personal-access-token + private Gist (as this section said in the prior draft). **That path no longer exists in the codebase** — see §2.1, it was fully replaced this cycle.

### 1.4 Explicit v1.0 principles (from the design doc — worth restating so v2.0 doesn't accidentally violate them)

- No accounts, no tracking, nothing leaves the browser unless a tool explicitly needs it.
- Not a booking engine — every flight/hotel/lounge tool is an *estimator or comparator*, never a live search or affiliate funnel.
- Offline-first: offline tools must never spinner/error; online tools must declare connectivity before use.
- Zero build step, zero framework, one HTML file per surface.

---

## 2. What shipped since the last draft (2026-07-01 → 2026-08-04) — now also locked

The prior draft treated cloud sync as the single most sensitive open decision in all of v2.0 and recommended gating it behind a dedicated design doc before any code landed. That happened correctly, but the resulting feature **shipped already** — faster and via a different path than the original phasing (§9 of the old draft) assumed. Restating it here so the rest of this document doesn't re-litigate it or plan around a scenario that no longer exists.

### 2.1 Cloud sync: Google Sign-In + Drive (done, not GitHub OAuth, not a custom backend)

- A dedicated design doc (`TripTools_CloudSync_Plan.md`, dated 2026-07-02) was written and signed off first, per the process the data-consolidation effort established.
- Shipped as **Google Identity Services (token model) + Drive `drive.file` scope** — narrowest possible Drive permission, no backend, no server-held secret, $0 running cost. `GOOGLE_CLIENT_ID` is live in `GoTools.html` (not a placeholder).
- Companion trip-sharing preserved via visible, shareable Drive files (a trip file set to "anyone with the link, reader") — a companion can import a shared trip **without signing in**, keeping the zero-account guarantee intact on the receiving side.
- **The legacy GitHub PAT / Gist sync was fully removed** (not kept as a fallback) — the owner's sign-off explicitly waived the soak period once Google sign-in was verified live, so Phase B and Phase C of that plan collapsed into one PR.
- **This closes out old-§2 decision 1 (architecture) and old-§5's C1/C2 fork entirely.** There is no more open cloud-sync architecture decision. Any future cloud-sync work is refinement (e.g., Google Picker for browsing shared-with-me files), not a new foundational choice.

### 2.2 Growth infrastructure (shipped, not yet used)

- SEO/social meta tags, Open Graph/Twitter Card tags, `og-image.jpg`, and `GOTOOLS_MARKETING_PLAN.md` (a zero-cost, $0-budget push targeting 1,000 tries in 4–8 weeks).
- **GoatCounter** wired in — free, cookie-less, aggregate-only pageview analytics, consistent with the no-tracking principle.
- Privacy Policy and Terms of Service pages (`privacy.html`, `terms.html`), required for the Google OAuth consent screen and now also linked in the site footer.
- **As of this revision, per the owner: the growth push has not launched yet** — outreach hasn't started, so there is no usage data yet to mine. Treat GoatCounter as instrumented-but-empty; §3 below adds a second, more granular layer of telemetry to have *before* outreach starts, not after.

### 2.3 What did *not* ship (confirmed directly against the code, not assumed)

Every other 2026-07-01 workstream is still fully open — verified by direct inspection of `GoTools.html`, not carried over from the old draft:

- No PWA manifest, no service worker, no install prompt handling.
- No `Notification` API usage anywhere (no expiry/countdown local notifications).
- No `.ics`/`VCALENDAR` export.
- No Destination Dossier, no EV/kWh fuel mode, no gamification/badge system, no visited-countries map.
- No AI/BYO-key panel, no "Ask GoTools."

So the real state, in one line: **cloud sync is done and out of scope going forward; everything else from the last draft (new applets, deepening tools, PWA/notifications/gamification, AI, data completeness) is untouched and needs fresh prioritization** — which is what the rest of this document now does.

---

## 3. Decisions locked in for this revision (owner, 2026-08-04)

1. **Top priority: Workstream A (new applets)** — specifically the four items below, starting with Destination Dossier.
2. **Growth push has not launched yet.** No usage data exists to weight priorities by; ordering below is judgment-based, not data-based, by design.
3. **Add lightweight anonymous per-tool telemetry**, extending the existing GoatCounter integration (custom events, not a new tracking system) — and ship it **first**, before Workstream A, specifically so there's a usage baseline in place before new applets launch and before the growth push starts. This directly serves the "not yet launched" answer above: instrument now, while there's no traffic to disturb, so the first real traffic is measured from day one.
4. **AI features (old Workstream D) are demoted to the backlog**, alongside the traveler-segment ideas (pet travel, accessibility, digital nomad) — not committed v2.0 scope. Provider-scope questions (OpenAI/Anthropic/Gemini) are deferred until it's actually scheduled.
5. **Priority order for everything else:** Telemetry → **A** (new applets) → **C** (PWA install + local notifications + gamification) → **B** (deepen existing tools) → **E** (remaining data completeness: ATM gap, refresh cadence, destination granularity).

---

## 4. v2.0-P0 — Per-tool telemetry (ships first, before any new applet)

Extend the existing GoatCounter hook — do not stand up a new analytics system or add anything that leaves the "aggregate, cookie-less, no accounts" posture.

- Fire a GoatCounter custom event per tool open (`tool:<id>`), so usage-by-tool becomes visible in the existing dashboard.
- Track destination/country lookups that miss (`resolveDestination()` returning no match) — this is the concrete signal Workstream E's "grow destination granularity where user demand shows up" recommendation was blocked on.
- No per-user identifiers, no new consent UI needed (same script, same privacy posture already disclosed) — but do add one line to `privacy.html` noting that tool-open and lookup-miss events (not raw input) are counted in aggregate.
- **Why first:** the growth push hasn't started yet, so this is the one moment usage instrumentation can land with zero backfill gap — every subsequent phase (especially A and E) benefits from having a "before" baseline.

---

## 5. v2.0-P1–P4 — Workstream A: new applets (top priority)

In ship order, per owner decision:

1. **Destination Dossier** — a single-screen brief for the active trip's country (or any looked-up country) pulling one line each from Safety level, Emergency numbers, Tipping norms, Driving side, Quiet hours, eSIM status, next public holiday, and phrasebook language. Each line deep-links to its full tool. Pure aggregation over data that already exists in `TT_COUNTRIES`/`resolveDestination()` — no new data domain, natural extension of Trip Briefing. **Ships first** — highest leverage, lowest new-data risk of the four.
2. **EV / hybrid road trip support** — Fuel Cost Calculator currently assumes gas-only; add an EV mode (cost-per-kWh, charging-stop planning) inside the existing Road Trip phase.
3. **International Customs Duty** — Customs Duty Calculator is US-only (the $800 exemption); extend to Canada/UK/EU/Australia duty-free allowances so non-US travelers get value from the Home phase.
4. **Multi-trip history / "countries visited" map** — a new lightweight view (not a full tool) that reads past MyTrip records and renders a visited-countries map using the existing `TT_COUNTRIES` alpha-2 keys. No new data domain required. (Natural pairing with the multi-trip dashboard in Workstream C, below — sequenced after Dossier/EV/Customs Duty since it depends on there being multiple archived trips to show.)

---

## 6. v2.0-P5–P6 — Workstream C: retention & platform (PWA, notifications, gamification)

Deliberately placed *after* new applets per the owner's ordering, but *before* deepening existing tools — the reasoning being that these are zero-account, ship independently of any other decision, and matter most right before the growth push actually starts driving new visitors who need a reason to come back.

- **Installable PWA** — manifest + service worker for true offline caching (today "offline" means "no network call," not "installable/cacheable app shell").
- **Local notifications** — passport/visa/frequent-flyer-point expiry reminders and trip-countdown milestones, via the Notification API — zero account required, ships independently of cloud sync.
- **Multi-trip dashboard** — past-trips archive, stats (countries visited, days traveled, total spent by rolling up Trip Cost Recap history) — feeds and is fed by the visited-countries map in Workstream A item 4.
- **Light gamification** — milestone badges (countries visited, tools used across all 10 phases) — additive, cosmetic, no new data domain.

---

## 7. v2.0-P7–P9 — Workstream B: deepen existing tools

- **Itinerary Builder** — `.ics` calendar export; per-day budget rollup tying into Travel Budget Planner instead of living separately.
- **Group Trip Organizer** — real-time voting via the (now Google Drive-based) sync channel instead of local-only state.
- **Smart Packing List / Weather-Based Wardrobe** — unify their state so a forecast pull feeds the rules-based packing generator directly.
- **Park Day Optimizer** — layer in a live wait-time source (e.g., a free queue-times-style API) as a new *online* enhancement, consistent with the existing connectivity-badge pattern — falls back to the current static strategy guide when offline.
- **Trip Journal + Review Drafter** — combined "trip recap" export (journal entries + photos + spend summary) as a shareable artifact.
- **Frequent Flyer Tracker / Passport Expiry Checker** — pairs directly with the notification work shipped in Workstream C (P5–P6) — these tools already compute expiry dates, they just couldn't alert you until notifications existed.

---

## 8. v2.0-P10 — Workstream E: remaining data completeness

Telemetry itself moved to P0 (§4). What's left here is the data-quality work that telemetry will make *evidence-based* instead of guesswork, plus items that don't depend on it:

- **ATM Fees:** either close the remaining 37-country gap with clearly-labeled "approximate" figures, or formally document the caveat-only state as permanent-by-design.
- **Recurring re-verification cadence:** GSA per-diem rates (published annually), airline baggage fees, Public Holidays' movable-feast tables (validated only through 2030), driving speed/BAC limits, visa rules all silently rot without a refresh cycle. Recommend an annual "data refresh" pass per domain.
- **Destination granularity below the country level** — now decidable with real data: the P0 lookup-miss telemetry (§4) tells you exactly where `TT_DESTINATIONS`'/Cost of Living's/Destination Picker's smaller lists are actually being missed, instead of guessing.

---

## 9. Backlog — demoted or deferred, not committed v2.0 scope

### 9.1 AI-assisted features (was Workstream D — demoted this revision)

Mirrors the trust model already established by Drive sync: user pastes their own API key into a settings panel, stored in `localStorage` only, calls go straight from browser to provider. Kept here for when it's revisited — **do not schedule against this until it's pulled back into a committed phase**, and re-open the provider-scope question (OpenAI/Anthropic only vs. also Gemini, given Google sign-in is already integrated) at that time.

1. Smart Packing List, AI-augmented (free-text trip description augments, doesn't replace, the rules-based generator).
2. Itinerary auto-draft seeded from MyTrip + Bucket List + destination data.
3. "Ask GoTools" — scoped Q&A that must cite/restate the existing verified `TT_COUNTRIES`/domain data rather than freelancing, especially for legally/medically adjacent domains (Visa, Vaccination, Customs, Driving Rules, Safety & Advisories).

### 9.2 New traveler segments

- **Pet travel** — pet documents, in-cabin/cargo airline rules, international import requirements, pet-friendly lodging checklist.
- **Accessibility & mobility** — wheelchair-accessible destination notes, service-animal rules by country, traveling with medical equipment.
- **Digital nomad / remote work** — long-stay/digital-nomad visa guide, wifi-quality expectations, coworking/time-zone-overlap planning.

### 9.3 Monetization groundwork (from `GOTOOLS_MARKETING_PLAN.md` §6, not part of this roadmap's committed scope)

Not evaluated in this revision — the growth push hasn't launched, so there's no usage data to base an ads-vs-tip-jar-vs-Pro-tier decision on yet. Revisit once P0 telemetry (§4) and the growth push both have real numbers.

---

## 10. Suggested phasing (supersedes the old §9 table)

| Phase | Ships | Depends on |
|---|---|---|
| v2.0-P0 | Per-tool + lookup-miss telemetry (GoatCounter custom events) | Nothing — ships first |
| v2.0-P1 | Destination Dossier | P0 (so its usage is measured from day one) |
| v2.0-P2 | EV / hybrid road trip mode | — |
| v2.0-P3 | International Customs Duty (Canada/UK/EU/Australia) | — |
| v2.0-P4 | Visited-countries map | Benefits from P6 (multi-trip dashboard) but can ship standalone |
| v2.0-P5 | PWA install + service worker | — |
| v2.0-P6 | Local notifications + multi-trip dashboard + light gamification | P5 (notifications want the installed-app context) |
| v2.0-P7 | Itinerary `.ics` export + budget rollup | — |
| v2.0-P8 | Group voting via Drive sync; Packing/Wardrobe unification | Cloud sync (done, §2.1) |
| v2.0-P9 | Park Day Optimizer live wait times; Trip Journal/Review recap export | — |
| v2.0-P10 | ATM Fee gap closure + documented refresh cadence + destination-granularity growth (data-driven by P0) | P0 |
| Backlog | AI features (§9.1), new traveler segments (§9.2), monetization (§9.3) | Not scheduled |

---

## 11. Open questions

- **Telemetry event taxonomy:** P0 needs a concrete list of GoatCounter custom event names (e.g., `tool-open:<id>`, `lookup-miss:<query>`) before implementation — worth a quick one-page spec, not a full design doc, given the data-consolidation precedent of writing things down before code for anything touching data collection.
- **AI provider scope:** deferred per §9.1 — not worth resolving until the backlog item is pulled back into a committed phase.
- **Growth push timing:** the marketing plan is ready to go but hasn't launched. Worth deciding whether launch waits for P0 telemetry to land first (so day-one traffic is measured) or proceeds in parallel.
- **Monetization:** explicitly deferred per §9.3 until real usage data exists from both GoatCounter and the new P0 telemetry.
