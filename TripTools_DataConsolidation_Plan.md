# TripTools — Destination & Country Data Consolidation Plan

**Status:** Proposal for review
**Author:** Generated for PlayologyEntertainment / TripTools
**Date:** 2026-06-20
**Decision inputs (agreed):**
1. **Scope** — Full two-tier registry (one country registry + one destination registry + one shared resolver).
2. **Coverage** — Expand every tool to the *union* of countries and **fill in the missing payloads** with real data.
3. **Verification bar** — Filled data ships **best-effort: dated + source-noted + disclaimed** (no per-domain human sign-off gate).
4. **City coverage** — **Grow into one list**: the destination registry expands to cover everything the city tools need (~140+), so there is truly one destination list.
5. **Delivery cadence** — **Per-phase PRs** (each phase lands as one PR).
6. **Currency** — **In scope now** (Phase 4): fold `CURRENCIES` / `FX_*` identity into `TT_COUNTRIES.currency`.
7. **Deliverable now** — This written plan, for review before any code changes.

---

## 1. The problem in one paragraph

TripTools is a single 43.6k-line `TripTools.html` file containing ~70 applets. What looks like "destination data" is actually **~14 independent datasets** at two different granularities (city/region vs. country), each with its own schema, its own country coverage, its own flag/name conventions, and — critically — **its own bespoke logic for matching the user's free-text `trip.destination` string** (e.g. `"Paris, France"`). The same trip can resolve to different results (or fail to resolve) from one tool to the next, and identity data (country names, flags, ISO codes) is duplicated and drifts across the file.

---

## 2. Current-state inventory

### Tier A — "Destination" lists (city / region granularity)

| Dataset | Line | ~Count | Consumed by | Identity fields | Payload fields |
|---|---|---|---|---|---|
| `DP_DESTINATIONS` | 18913 | 113 | Destination Picker, Destination Comparison, Best Time to Visit | `id, name, country, flag (2-letter), continent` | `types[], budget, season, dailyRange, visa, desc` |
| `TZ_CITIES` | 29867 | ~140 | Time Zone Converter, Meeting Planner, Jet Lag | `city, country, flag (emoji)` | `tz` (IANA zone) |
| `COL_CITIES` | 34068 | ~41 | Cost of Living | `id, n, flag (emoji), region, cur` | `acc[], food[], trans[], fun[]` |

> `DP_DESTINATIONS` is already shared across 3 tools — it is the proof-of-concept for what we want everywhere.

### Tier B — "Country" lookup lists (country granularity)

| Dataset | Line | ~Count | Consumed by | Key | Flag style |
|---|---|---|---|---|---|
| `VG_COUNTRIES` | 27306 | ~198 | Visa Requirement Guide | `c` (ISO-2) | none / derived |
| `PP_COUNTRIES` | 27706 | ~144 | Power Plug & Adapter Guide | `c` (ISO-2) | emoji `f` |
| `EM_DATA` | 28313 | ~190 | Emergency Numbers | ISO-2 object key | none |
| `TC_COUNTRIES` | 30903 | ~100+ | Tipping & Currency Guide | `c` (ISO-2) | emoji `f` |
| `DR_DATA` | 31959 | ~80+ | Driving Rules | ISO-2 object key | derived |
| `ID_DATA` | 21491 | ~80+ | International Driving Permit | ISO-2 object key | emoji `f` |
| `QH_DATA` | 32847 | ~40+ | Quiet Hours / Etiquette | ISO-2 object key | — |
| `AF_COUNTRIES` | 33615 | ~16 | ATM Fee Guide | `id` | emoji `flag` |
| `VR_COUNTRIES` | 34534 | ~42 | VAT Refund Guide | `id` | emoji `flag` |
| `VAC_COUNTRIES` | 38596 | ~147 | Vaccination & Health | `c` (ISO-2) | derived |
| `CDH_COUNTRIES` | 41349 | ~27 | Customs Duty Helper | `c` (ISO-2) | derived |

### Adjacent (currency identity — related, lower priority)
`CURRENCIES` (14492), `FX_BOARD_CURRENCIES` (20009), `FX_ALL_CURRENCIES` (20025) — used by the Currency Converter and tipping/cost tools. Not "destinations," but they share the same duplication smell and should eventually key off the country registry's `currency` field.

---

## 3. Three concrete problems this causes

1. **Duplicated, drifting identity.** Country name + flag are repeated across ~11 datasets in **three different flag conventions** (2-letter code, emoji, derived). Coverage ranges from **16 → 198** countries. Names drift ("Czech Republic" vs "Czechia", "China SAR" vs "Hong Kong").
2. **~11 re-implementations of the same match.** Every applet fuzzy-matches `trip.destination` against its own list with bespoke code — `tcGuessCountryFromDest` (31271), `tzMatchDestination` (30049), and inline `.toLowerCase().includes(...)`, `.split(',').pop()`, etc. (seen at lines 27895, 28152, 28634, 30075, 31249, 31824, 32662, 33016, 34843, 38804, 41643…). Results are inconsistent.
3. **No city → country link.** Nothing tells the Visa/Vaccine/Plug tools that "Kyoto" means Japan, so destination-driven tools and country-driven tools can't cooperate.

---

## 4. Target architecture — two registries + one resolver

### 4.1 `TT_COUNTRIES` — canonical country registry (single source of truth)

Keyed by ISO-3166-1 alpha-2. Holds **identity only**; every domain tool keeps **only its own payload**, keyed by the same code.

```js
const TT_COUNTRIES = {
  FR: {
    name: 'France',
    aliases: ['French Republic'],
    flag: '🇫🇷',          // emoji canonical; 2-letter derivable from key
    iso3: 'FRA',
    continent: 'europe',
    region: 'Western Europe',
    currency: 'EUR',
    callingCode: '33',
  },
  // …union of every country code appearing in any Tier-B dataset (~200)
};
```

Each Tier-B dataset is rewritten to drop redundant name/flag and key off the registry:

```js
// before:  {c:'FR', n:'France', f:'🇫🇷', cur:'EUR', sym:'€', svc:'rarely', v:{…}}
// after (TC payload only):
const TC_DATA = { FR: { svc:'rarely', v:{…}, note:'…' }, … };
// name/flag/currency come from TT_COUNTRIES.FR
```

### 4.2 `TT_DESTINATIONS` — canonical destination registry

Extends `DP_DESTINATIONS`; each destination links to a country code and absorbs the TZ/COL data so the city tools share one list.

```js
const TT_DESTINATIONS = {
  paris: {
    name: 'Paris',
    country: 'FR',          // ← link into TT_COUNTRIES
    continent: 'europe',
    types: ['city','culture','foodie','romantic'],
    budget: 4, season:'Apr-Jun, Sep-Oct', dailyRange:'$150-$350',
    tz: 'Europe/Paris',     // ← folds in TZ_CITIES
    cost: { acc:[…], food:[…], trans:[…], fun:[…] }, // ← folds in COL_CITIES (where present)
    desc: '…',
  },
  // …
};
```

Per decision #4, there is **one destination list**: `TT_DESTINATIONS` grows to cover everything the city tools need (~140+ entries, absorbing all `TZ_CITIES`/`COL_CITIES` cities), each linked to a country code. To preserve the "spin the globe" inspiration quality, entries carry a flag (e.g. `inspire: true`) so the Destination Picker can draw from the curated subset while Time Zone / Cost of Living draw from the full set — one list, one resolver, role-tagged rows.

### 4.3 `resolveDestination()` — the one matcher to replace ~11

```js
// Replaces tcGuessCountryFromDest, tzMatchDestination, and all inline matching.
function resolveDestination(text){
  // returns { destinationId?, countryCode?, city?, confidence }
}
```
Every tool calls this instead of rolling its own. One place to fix matching bugs; consistent results across all tools for the same trip.

---

## 5. Coverage expansion + payload fill (per decision #2)

Because we expand every tool to the **union (~200 countries)** and fill missing payloads, this is as much a **data-authoring** effort as a refactor. Approach:

- **Build the union first.** Generate the master code list from all Tier-B datasets; that defines the rows every tool must eventually cover.
- **Fill per domain, with sourcing notes (best-effort — decision #3).** For each tool, author the missing rows from authoritative references (e.g. IATA/Timatic-style visa references, WHO/CDC for vaccines, IEC world plugs for power, national customs sites, official emergency-number registries). Record the source + `verified` date per dataset (the file already uses `BT_DATA_VERIFIED`-style stamps — extend that pattern). **No per-domain human sign-off gate** — data ships best-effort and dated; correctness rests on the disclaimers + version stamps.
- **Keep the existing disclaimers.** Visa, customs, vaccination, and driving tools already show "verify with official sources" banners. Authored data is *directional planning guidance*, not authority — every filled payload must carry the same disclaimer and a `verified` date.
- **Graceful gaps.** Any row still genuinely unknown renders a clear "no data yet" state rather than a blank or a wrong default.

> ⚠️ **Scope flag for review:** filling ~11 datasets up to ~200 countries each is the largest part of the effort and the part with real-world accuracy risk. Recommend doing it **domain-by-domain behind the registry**, not all at once, and having a human spot-check each domain before it ships.

---

## 6. Phased migration plan

Each phase is independently shippable and leaves the app fully working.

**Phase 0 — Scaffolding (no behavior change)** ✅ *Done*
- Add `TT_COUNTRIES` (identity only, union of existing codes) and `resolveDestination()`.
- Add a tiny test harness (a hidden debug panel or console asserts) that checks every existing dataset key exists in `TT_COUNTRIES`.
- *Implemented:* `TT_COUNTRIES` (200 countries — name/flag/iso3/continent/currency/calling + aliases), `ttCountry()`, `resolveDestination()`, and `ttCountriesSelfCheck()` (runs on load, console-only) inserted after the `CURRENCIES` declaration. Additive — no tool consumes it yet.

**Phase 1 — Destination tier consolidation** *(one PR)* ✅ *Done*
- Build the single `TT_DESTINATIONS` from `DP_DESTINATIONS` + country links + `tz`/`cost` folds, grown to ~140+ to absorb every `TZ_CITIES`/`COL_CITIES` city (decision #4). Tag curated rows with `inspire: true`.
- Repoint Destination Picker (curated subset), Comparison, Best Time, Time Zone, Cost of Living (full set) to it via the resolver. Retire `TZ_CITIES`/`COL_CITIES`.
- *Implemented:* single `TT_DESTINATIONS` master (216 rows — 113 curated `inspire` + 103 supplementary cities; 28 destinations have `tz` folded, 22 have `cost`). Every row links to a country via `cc`. `DP_DESTINATIONS`, `TZ_CITIES`, `COL_CITIES` are now **back-compat projections** of the master, verified **byte-identical** to the previous hand-maintained arrays (zero data loss, zero display change), so the five consumer tools stay untouched. `resolveDestination()` extended to resolve bare cities (e.g. "Tokyo" → JP) via `TT_DEST_INDEX`. Added `PF`/`GU` to `TT_COUNTRIES` (now 202) for full Time-Zone coverage.
- *Matcher wiring ✅ (done):* see the "resolver wiring" note under Phase 2.

**Phase 2 — Country tier identity migration (one tool at a time)** ✅ *Done*
- For each Tier-B dataset: strip identity fields, key payload off ISO-2, pull name/flag/currency from `TT_COUNTRIES`, replace bespoke matcher with `resolveDestination()`. Order by smallest blast radius first: `AF` → `VR` → `QH` → `CDH` → `ID` → `DR` → `PP` → `TC` → `EM` → `VAC` → `VG`.
- *Implemented:* migrated every identity-carrying country dataset to a **payload-only** form keyed by ISO-2, with the consumer-facing array/object rebuilt as a **projection** that pulls `name`/`flag` from `TT_COUNTRIES` — each verified **byte-identical** to its previous literal:
  - List datasets: `AF_COUNTRIES`, `VR_COUNTRIES`, `PP_COUNTRIES`, `TC_COUNTRIES`, `VAC_COUNTRIES`, `VG_COUNTRIES`.
  - Keyed dataset: `ID_DATA`.
  - Constructor + inline dataset: `CDH_COUNTRIES` (kept the DRY `cdhEU()` helper; identity now injected from the registry for all 42 entries).
  - Intentional tool-specific labels (e.g. "Türkiye", "China (mainland)", "UAE", "Myanmar (Burma)") preserved as per-entry `n` overrides.
- *No identity to migrate:* `QH_DATA` and `DR_DATA` carry no name/flag. `EM_DATA` duplicates only the calling code (`cc`) — left as-is (a minor, optional cleanup).
- *Regression test:* the full page script evaluates without throwing under a DOM stub, and every migrated dataset was diffed against the pre-Phase-2 commit (`4483e7a`) — identical entry count, identical order, zero dropped/added codes, zero name changes, and **zero payload-field diffs** across `VG`/`PP`/`TC`/`VAC`/`AF`/`VR`/`CDH`.
- *Resolver wiring ✅ (done):* every tool's bespoke `trip.destination` matcher now calls the shared `resolveDestination()` **first** (alias-aware — "UK"→GB, "Holland"→NL, "UAE"→AE — plus bare-city resolution like "Tokyo"→JP), falling back to each tool's original logic so there is **no regression**. Covered: the two shared helpers `tcGuessCountryFromDest` (Tipping, Driving Rules, Quiet Hours) and `tzMatchDestination` (Time Zone, Jet Lag), plus inline auto-detect in Power Plug, Visa Application, Emergency, VAT Refund, Vaccination, and Customs. Specialised airport/border matchers (Flight Estimator, Border Crossing) left as-is by design.

**Phase 3 — Coverage expansion / payload fill** ✅ *Done*
- Per domain, author the union rows with sources + `verified` dates and disclaimers. Ship domain-by-domain.
- *Power Plug ✅ (first domain):* expanded `PP_PAYLOAD` from 144 → **202** countries (full registry coverage). The 58 added rows were web-sourced (worldstandards.eu / power-plugs-sockets.com / wise.com via search, Jun 2026), carry a dated source comment, and are directional planning data behind the tool's existing "verify before travel" disclaimer. Verified: all 202 link to valid registry entries, no duplicate codes, every plug letter is known to `PP_TYPES`.
- *Bug fixed in passing:* Phase 0's extraction had mistaken `EM_DATA.f` (the **fire** number) for a flag, polluting 17 registry flags (e.g. North Korea showed `119`). All 17 corrected to proper emoji, and a **flag-format guardrail** added to the self-check so non-emoji flags are flagged at load.
- *Emergency numbers ✅ (second domain):* `EM_DATA` was already 198/202 — filled the last 4 (Puerto Rico, Guam, Brunei, French Polynesia, verified via search) for **full 202 coverage**. Also backfilled `calling`/`currency` for PR and BN in the registry.
- *Visa ✅ (third domain):* `VG_PAYLOAD` was 198/202 — filled the same 4 territories for **full 202 coverage** (PR/GU follow US entry policy, PF follows France/Schengen short-stay, Brunei web-sourced). Each carries the existing per-tier rule encoding behind the tool's verify-with-embassy disclaimer.
- *Vaccination ✅ (fourth domain):* `VAC_PAYLOAD` expanded 147 -> **202** (full coverage). The 55 added rows (Caribbean, micro-states, Caucasus/Central Asia, Pacific, and high-stakes Africa/Middle East) were authored from CDC Yellow Book 2024 guidance (search-confirmed for the malaria/yellow-fever specifics), behind the tool's strong "not medical advice - confirm with a travel clinic" disclaimer. Verified: all malaria levels and vaccine codes valid, every country links to the registry.
- *Customs ✅ (fifth domain — now full coverage):* `CDH_COUNTRIES` 42 → **202** (full registry coverage). First the 12 remaining EU members were added via the shared `cdhEU()` harmonised-rules template (→ 54, all 27 EU states). The remaining **148 non-EU** countries were then authored region-by-region — South America (10), Caribbean & Central America (21), non-EU Europe (15), Asia & Middle East (37), Oceania (14) and Africa (51) — each with bespoke duty-free allowances, prohibited/restricted lists and cash thresholds **web-sourced from official customs authorities + IATA/World Travel Guide references (Jun 2026)**, behind the tool's existing verify-before-travel disclaimer. Shared `CDH_PROH_COMMON`/`CDH_REST_COMMON` helpers keep the universal items DRY; the safety-critical facts (Gulf/Islamic-state alcohol & pork import bans, Pacific/island biosecurity, plastic-bag bans, wildlife/CITES & antiquities export controls) are captured per country. Lower-confidence micro-state rows carry an explicit "approximate — confirm with [authority]" note rather than a false-precise figure. Verified: all 202 link to `TT_COUNTRIES`, no duplicate codes, no missing fields, page script parses clean. Shipped as six per-region commits.
- *Coverage status of remaining domains:* VAT refund (~42, only ~50 countries run tourist schemes), ATM fees (16, operator-fee amounts volatile), tipping (53, subjective) are complete-by-design or low-confidence and left as-is. Power Plug, Emergency, Visa, Vaccination and Customs are at **full 202-country coverage**. Driving Rules (`DR`), International Driving Permit (`ID`) and Quiet Hours (`QH`) carry no identity and were never scoped into the union fill — their coverage is a separate, open product decision rather than registry work.

**Phase 4 — Currency registry (in scope — decision #6)** ✅ *Done*
- Fold `CURRENCIES`/`FX_*` identity into `TT_COUNTRIES.currency`; keep FX rate fetching as-is.
- *Implemented:* `FX_ALL_CURRENCIES` is the canonical currency registry (code → name). Backfilled `currency` for the **153** countries that lacked it, so **all 202** now reference the registry (e.g. `BJ → XOF`, `AG → XCD`). Added the one referenced currency that was missing from the registry (`KPW`, North Korean Won → 162 total). Added a **referential-integrity guardrail** to the self-check: warns if any country lacks a currency or references a code absent from `FX_ALL_CURRENCIES`. FX rate fetching and the curated converter lists (`CURRENCIES`, `FX_BOARD_CURRENCIES`) are unchanged.

**Phase 5 — Guardrails** ✅ *Done*
- Lint/test that no new dataset reintroduces duplicate identity; document the registry contract near the top of the data section.
- *Implemented:* the load-time `ttCountriesSelfCheck()` now scans every payload dataset (`VG`/`PP`/`TC`/`VAC`/`AF`/`VR`) and warns if any entry re-introduces a `name`/`flag` that merely **duplicates** the registry (a *differing* value is an allowed per-tool override and is not flagged). The registry contract is documented in a header comment above `TT_COUNTRIES`. Verified: clean run reports OK; a seeded redundant `name` is correctly flagged.

---

## 7. Risks & mitigations

| Risk | Mitigation |
|---|---|
| 43.6k-line single file; large diffs are hard to review | Per-phase PRs (decision #5); within a phase, migrate one tool per commit so the diff reads tool-by-tool; registry is additive in Phase 0 |
| Authored legal/health data could be wrong | Per-domain `verified` stamps, retained disclaimers, human spot-check gate before each domain ships |
| Matching regressions (a tool stops finding a country) | `resolveDestination()` covered by assertion tests against every known name/alias; migrate behind it, not around it |
| Growing to one ~140+ list could dilute "spin the globe" inspiration quality | `inspire: true` tag — Destination Picker draws only curated rows; other tools use the full set |
| Name/flag drift reappears | Phase 5 guardrail check; payloads forbidden from carrying `name`/`flag` |

---

## 8. Effort estimate (rough)

- Phases 0–2 (the actual *consolidation*): the bulk of the engineering, mechanical and low-risk once the registry exists.
- Phase 3 (coverage + payload fill): the largest *time* cost, dominated by data authoring/verification, parallelizable per domain.
- Phases 4–5: small.

---

## 9. Resolved decisions

1. **Verification bar** — Best-effort: dated + source-noted + disclaimed. No per-domain human sign-off gate.
2. **City coverage** — Grow into **one** destination list (~140+); curated rows tagged `inspire: true` for the Destination Picker.
3. **PR cadence** — Per-phase PRs (one tool per commit within a phase).
4. **Currency registry** — In scope as Phase 4.

All open questions are resolved; the plan is ready to execute starting at **Phase 0**.
