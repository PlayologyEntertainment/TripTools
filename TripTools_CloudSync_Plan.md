# GoTools — Optional Cloud Sync (Google Sign-In + Drive) Design Plan

**Status:** Proposal for review — *no code lands until this is signed off*
**Author:** Generated for PlayologyEntertainment / GoTools
**Date:** 2026-07-02
**Supersedes:** the BYO-GitHub-token / Gist Trip Sync (`GoTools.html` ~L19683+)
**Related:** `ROADMAP.md` §2.1 (C1/C2 architecture decision), §4 (v2.0-Design gate), `TripTools_DataConsolidation_Plan.md` (precedent: design-doc-before-code)

**Decision inputs (agreed with owner, 2026-07-02):**
1. **Deliverable now** — *this written plan first.* Get sign-off, **then** implement. (Q1 → A)
2. **Sharing scope** — **preserve companion sharing** in this first step, using **visible, shareable Drive files** (not the hidden App Data folder). (Q2 → B)
3. **GitHub removal** — **keep GitHub/Gist as a fallback** until Google sign-in is wired, deployed to production, and verified; remove it in a later phase. (Q3 → A)
4. **Google Cloud setup** — **include exact step-by-step setup instructions** in this doc. (Q4 → B)

---

## 1. The goal in one paragraph

GoTools is a single-file, local-first web app (`GoTools.html`) served statically from
`https://www.playologyentertainment.com/GoTools/GoTools.html` (Hostinger, FTP deploy — see
`.github/workflows/deploy.yml`). Today its only "cloud" feature is **Trip Sync**, which asks each
user to paste a **GitHub personal-access-token** and stores their active trip in a **private Gist**;
companions exchange Gist ids ("sync codes") to share a trip. We are adding an **optional, opt-in
account layer** — **Google Sign-In + Google Drive** — that does the same two jobs (cross-device sync
of your own trips, and sharing a trip with a companion) with **less friction and no GitHub account**.
This is **purely additive**: **every tool remains 100% usable with zero account**, exactly as today.
The only thing sign-in unlocks is sync. Once Google is live and proven, the GitHub path is removed
so Google becomes the single sync mechanism.

### 1.1 Non-negotiable principle (restating `ROADMAP.md` §1.4)

> *No accounts, no tracking, nothing leaves the browser unless a tool explicitly needs it.*

Cloud sync is the one workstream that brushes against this, so the rules are:

- **Zero-account guarantee.** The app must fully function with no sign-in. Sign-in gates **only** the
  sync panel. A companion *receiving* a shared trip must be able to import it **without** signing in
  (read-only, via the public share link — see §4.3).
- **No GoTools-hosted backend.** Auth and storage run **entirely client-side** against Google's APIs.
  No server, no database, no proxy, **$0 running cost** (see §9).
- **No GoTools-held secrets.** The only credential we ship is a **public** OAuth Client ID (safe to
  embed). We never handle a client secret (the client-side token flow doesn't use one — §3.2).

---

## 2. Current-state inventory (what we're replacing)

The existing Trip Sync lives in `GoTools.html`:

| Piece | Location | Role |
|---|---|---|
| Sync UI panel | ~L2395–2440 (`#mt-sync`) | Token field, "sync code" (own Gist id), companion codes, Publish/Refresh |
| `syncCfgGet/Set` | ~L19705 | Config in `localStorage['tt_sync_cfg']` (`token`, `myGistId`, `partners[]`, `lastSyncedAt{}`) |
| `syncApi` | ~L19742 | Authenticated `fetch` to `api.github.com` with `Authorization: Bearer <PAT>` |
| `syncMakeGist` / `syncPublish` | ~L19761 / L19801 | Create/update a **secret** Gist holding the bundle |
| `syncFetchBundle` / `syncRefresh` | ~L19835 / L19858 | Pull companions' Gists by id, merge **last-published-wins** |
| `syncBuildBundle` / `syncAdopt` | ~L19786 / L19845 | Serialize/deserialize the synced payload |
| `SYNC_DATA_KEYS` | ~L19695 | The exact `localStorage` slices that travel with a trip |

**The synced payload (unchanged by this migration).** Only MyTrip data moves — the trip record plus
these per-trip `localStorage` slices, keyed by `trip.id`:

`tt_pack_` · `tt_bizpack_` · `tt_budget_` · `tt_expenses_` · `tt_souvenir_` ·
`tt_wardrobe_items_` · `tt_cdn_prec_` · `tt_itinerary_` (Itinerary Builder day-by-day).

The **bundle schema** (`{app, kind:'trip-sync', schema:1, device, publishedAt, trip, data}`) and the
**last-published-wins** merge are **retained verbatim** — we are swapping the *transport* (Gist →
Drive file), not the data model. This keeps the diff small and the migration low-risk.

### 2.1 Why move off GitHub PATs

- **Friction:** users must have a GitHub account, navigate to *Settings → Developer settings → PATs*,
  pick the right scope, and paste a scary-looking token. That's a wall for a travel-planning audience.
- **Trust optics:** a `gist`-scoped PAT technically grants access to *all* the user's gists, not just
  ours. Google's `drive.file` scope (see §3.1) is strictly narrower — the app can only ever see files
  **it** created — which is a materially better privacy story to put on a consent screen.
- **Reach:** far more of the target audience already has a Google account than a GitHub one.

---

## 3. Auth design — Google Identity Services (client-side)

### 3.1 Scope: `https://www.googleapis.com/auth/drive.file` (and nothing more)

We request **exactly one** Drive scope: **`drive.file`** — "per-file access to files created or
opened by the app."

- The app can **only** read/write files **it created** (or that the user explicitly hands it via the
  Google Picker). It **cannot** see the rest of the user's Drive. Best-in-class privacy posture.
- **`drive.file` is a "recommended"/non-sensitive scope**, so the OAuth consent screen does **not**
  require Google's sensitive-scope verification or a CASA security assessment before going to
  production. This is the difference between "publish the consent screen and ship" and "wait weeks for
  a security review." We deliberately **avoid** `drive`, `drive.readonly`, and `drive.appdata`.
- We chose visible `drive.file` files over the hidden **App Data folder** because App Data files are
  private to each user and **cannot be shared** — and preserving companion sharing is a hard
  requirement for step 1 (decision Q2 → B).

### 3.2 Flow: GIS **token model** (implicit, no backend, no secret)

Because GoTools is a static SPA with no backend to exchange an auth code, we use **Google Identity
Services (GIS)** `google.accounts.oauth2.initTokenClient` — the browser **token model**:

```
[Sign in] → GIS popup → user consents to drive.file
          → GIS returns a short-lived OAuth access token (~1h) to a JS callback
          → app calls the Drive REST API with `Authorization: Bearer <token>`
```

- **No client secret** is involved (token model). The **Client ID is public** and embedded in
  `GoTools.html` as a constant — this is expected and safe for browser OAuth.
- **No redirect URI** to configure (GIS uses `postMessage`, not a redirect). We only register
  **authorized JavaScript origins** (§7).
- **Token lifetime:** access tokens live ~1 hour. We keep the token **in memory only** (never
  `localStorage`/`sessionStorage`) to limit XSS token-theft exposure. When a token expires mid-session
  we silently re-request; GIS can return a new token without a popup if the user is still signed in and
  consent is unchanged. A "Sign in with Google" / "Signed in as name@… · Sign out" state sits in the
  sync panel.
- **What persists in `localStorage`** (like today): non-secret sync config — which trips have a Drive
  file id, partner sync codes, `lastSyncedAt`, and a `signedInHint` (email/name for UI only). **Never
  the token.**

### 3.3 Library load — and the offline guarantee

We add one script tag: `https://accounts.google.com/gsi/client` (GIS), loaded **`async defer`**. If it
fails to load (offline, blocked), the app **degrades gracefully**: the sync panel shows "Sign-in
unavailable offline" and **every other tool is unaffected** — the app is offline-first and the GIS
script is the *only* new network dependency, isolated to the sync panel. No other code path awaits it.

---

## 4. Storage & sharing design (Google Drive)

### 4.1 File layout

- On first publish, the app creates a folder **"GoTools Trips"** in the user's Drive (id cached in
  config) and writes **one JSON file per trip** inside it.
- **File name:** `GoTools — <Trip Name>.gotrip.json`, MIME `application/json`.
- **Metadata:** Drive `appProperties` carries `{ app:'GoTools', kind:'trip-sync', tripId, schema,
  publishedAt }` so files are self-identifying and we can locate "my trip N" without parsing contents.
- **File body:** the exact bundle from §2 (`syncBuildBundle`).

### 4.2 Cross-device sync (your own trips — the primary win)

Sign in with the **same Google account** on another device → the app lists its own `drive.file` files
(it can see files it created), matches by `appProperties.tripId`, and pulls the newest by
`publishedAt`, merging **last-published-wins** (identical logic to today's `syncRefresh`). Publish
writes/updates the trip's file. This is the headline quality-of-life feature: your trips follow you
across phone/laptop/tablet with a single sign-in and no codes to copy.

### 4.3 Companion sharing (decision Q2 → B) — mirrors today's model

Today, companions **exchange Gist ids**. We keep that exact mental model, swapping Gist id → Drive
file id as the **"sync code."** Each person owns their own trip file; they exchange codes; Refresh
pulls each partner's file. Symmetric, last-published-wins — same as today.

Making a trip shareable:

1. Owner toggles **"Share this trip with a companion."** The app sets a Drive permission on that trip's
   file: **`type: anyone, role: reader`** (link-visible, read-only). This yields a stable **file id**
   = the sync code (optionally a `drive.google.com` link).
2. Owner sends the code to the companion (as today).
3. **Companion imports — no sign-in required.** Because the file is "anyone-with-link, reader," the
   companion's app fetches it via the public link **without any Google account**, preserving the
   zero-account guarantee for the *receiving* side. Two fetch paths, in order:
   - Primary: the shared file's web content link (public read for anyone-with-link files).
   - Fallback: `files.get?alt=media` when the companion happens to be signed in.
4. **Two-way editing:** if the companion also signs in and shares *their* copy back (exchanging codes,
   exactly like today's mutual Gist exchange), both sides Refresh and merge last-published-wins.

**Revoke:** a **"Stop sharing / make private"** button removes the `anyone` permission. The link stops
working immediately — a capability today's Gist flow lacks and a genuine improvement.

### 4.4 Privacy posture of a shared file (state it plainly in-UI)

"Anyone with the link, reader" means the trip JSON is readable by **anyone who has the (unguessable)
file id** — the **same trust model as today's *secret* (unlisted-but-not-private) Gist**. The sync
panel will say so in plain language, and default trips to **private** (unshared) until the user opts to
share. No trip is ever shared implicitly.

---

## 5. UI / UX changes to the sync panel (`#mt-sync`)

- **Headline flips from GitHub to Google.** Primary control: **"Sign in with Google"** (Google-branded
  button per Google's branding guidelines). Signed-in state shows the account and a **Sign out**.
- **Cross-device sync becomes the hero copy:** "Sign in to keep your trips on all your devices — it's
  optional; GoTools works fully without an account."
- **Sharing:** per-trip **Share / Stop sharing** toggle producing a copyable sync code; a **companion
  codes** box (import others' trips), and **Publish / Refresh** — same verbs users already know.
- **Legacy GitHub sync** moves into a collapsed **"Advanced · legacy GitHub sync"** `<details>` during
  the transition (Phase B), with a one-click **"Import my GitHub-synced trips → switch to Google"**
  migration helper. Removed entirely in Phase C.
- **Every sync action is reversible and clearly optional**; nothing about sign-in changes the hub, the
  tools, or the local-first data path.

---

## 6. Migration & phasing (decision Q3 → A: GitHub stays until Google is proven)

| Phase | Ships | GitHub sync | Gate to advance |
|---|---|---|---|
| **A — Build alongside** | Google Sign-In + Drive publish/refresh/share, behind the same panel. Placeholder `GOOGLE_CLIENT_ID`. | Fully intact, primary | Owner creates the Cloud project & Client ID (§7); verified working on a dev/localhost origin |
| **B — Google primary, GitHub legacy** | Google is the default path; GitHub collapsed under "Advanced · legacy" with a migration helper. Consent screen published to production. | Deprecated, still functional | Google sign-in verified **live** on `https://www.playologyentertainment.com`; a real cross-device sync + a companion share tested end-to-end |
| **C — Remove GitHub** | Delete PAT UI + `syncApi`/Gist code paths; clean up `tt_sync_cfg` legacy keys with a one-time migrator. Google is the sole mechanism. | **Removed** | Phase B stable in production for an agreed soak period; no open migration complaints |

Each phase is its own PR (per the `TripTools_DataConsolidation_Plan.md` per-phase cadence). This doc is
the **Phase 0** deliverable.

---

## 7. Exact Google Cloud setup (decision Q4 → B)

*Owner performs these once in the Google account that will "own" the app. ~15 minutes.*

1. **Create a project.** [console.cloud.google.com](https://console.cloud.google.com) → project
   picker → **New Project** → name **"GoTools"** → Create → select it.
2. **Enable the Drive API.** *APIs & Services → Library* → search **"Google Drive API"** → **Enable**.
   *(No API key is needed — Drive REST calls authenticate with the OAuth access token. An API key is
   only required if we later add the Google **Picker**, which is optional — §10.)*
3. **Configure the OAuth consent screen.** *APIs & Services → OAuth consent screen*:
   - User type: **External** → Create.
   - App name **GoTools**, **user support email**, app **logo** (use `GoTools.svg`/`Favicon.jpg`),
     **app domain** `playologyentertainment.com`, links to home/privacy/terms on that domain,
     **developer contact email**.
   - **Scopes** → Add → select **`.../auth/drive.file`** only. *(Because this is a non-sensitive
     "recommended" scope, no Google security assessment is required to publish.)*
   - **Test users** → add your own Google address (needed only while status is "Testing").
   - Save. Leave in **Testing** for Phase A; **Publish app** (→ "In production") before Phase B so any
     Google user can sign in.
4. **Create the OAuth Client ID.** *APIs & Services → Credentials → Create credentials → OAuth client
   ID*:
   - Application type: **Web application**, name "GoTools Web".
   - **Authorized JavaScript origins** (exact, scheme-qualified, no trailing slash):
     - `https://www.playologyentertainment.com`
     - `https://playologyentertainment.com` *(only if the apex also serves the app — confirm; see §11)*
     - `http://localhost:5173` *(or whatever local dev port we use; add each dev origin)*
   - **Authorized redirect URIs:** *leave empty* — the GIS token model doesn't use redirects.
   - Create → **copy the Client ID** (looks like `xxxxx.apps.googleusercontent.com`).
5. **Wire it in.** Paste the Client ID into the `GOOGLE_CLIENT_ID` constant near the sync code in
   `GoTools.html`. It's public; committing it is fine.
6. **Publish** the consent screen to production before Phase B (step 3, last bullet).

> **Note on origins:** the Client ID is bound to the exact origins above. Sign-in will silently fail on
> any origin not listed (e.g. `file://` when opening the HTML locally). Local testing must be over
> `http://localhost:<port>` that's registered, not a bare file open.

---

## 8. Security & privacy summary

| Concern | Mitigation |
|---|---|
| Client ID exposure | Public by design; safe to embed. Origin allow-list is the real guard. |
| Access token theft (XSS) | Token kept **in memory only**, never persisted; short (~1h) lifetime; re-requested silently. |
| Over-broad Drive access | **`drive.file` only** — app can never see files it didn't create. |
| Shared-trip exposure | Same as today's secret Gist (anyone-with-link reader); trips default **private**; explicit share toggle; **revoke** button. |
| No secrets held by GoTools | No client secret, no backend, no proxy — nothing to leak. |
| Zero-account guarantee | App fully usable signed-out; companions import shared trips **without** an account. |
| Data minimization | Only MyTrip slices (§2) sync; nothing else leaves the browser. GoatCounter analytics stay cookie-less/aggregate as today. |

---

## 9. Cost

**$0.** No hosting, no backend, no database. Google Drive API is free within per-user quotas (ample for
per-trip JSON blobs — publishing/refreshing is a handful of requests). We should handle Drive
**429/rate-limit** responses with a friendly "try again in a moment" message, but no billing is
involved. Static hosting stays on the existing Hostinger plan; deploy is the unchanged FTP push of
`GoTools.html`.

---

## 10. Out of scope for this step (explicitly deferred)

- **Notifications** (passport/visa expiry, trip countdown). The user's broader vision pairs sync with
  notifications, but **push** needs either the local **Notification API** (works with zero account —
  a *separate* ROADMAP item, ships independently) or a real backend (ROADMAP C2). This doc is **Drive
  sync only**; notifications are not gated on it and are not built here.
- **A GoTools-hosted backend / magic-link accounts** (ROADMAP C2). Not needed for Google Drive sync.
- **Google Picker–based companion import.** The public-link fetch (§4.3) avoids it. Picker (which needs
  an API key + Picker API) is a possible later nicety for "browse Shared-with-me," not required now.
- **AI / BYO-key features** (ROADMAP §2.3) — unrelated workstream.

---

## 11. Open questions — RESOLVED (owner, 2026-07-02)

1. **Apex vs. www.** ✅ **Both.** The app is served from `https://playologyentertainment.com` *and*
   `https://www.playologyentertainment.com` → register **both** origins on the OAuth client.
2. **Consent-screen branding.** ✅ Use **GoTools** as the app name with **`GoTools.svg`** as the logo.
   Privacy Policy and Terms of Service pages have been created in the repo (`privacy.html`,
   `terms.html`) and deploy to `https://www.playologyentertainment.com/GoTools/privacy.html` and
   `/GoTools/terms.html` — use those as the consent-screen privacy/terms URLs.
3. **Local-dev origin.** ✅ **`http://localhost:5173`** is registered as an authorized JS origin.
4. **Soak period for Phase C.** ✅ **None.** Remove the GitHub/Gist code **as soon as** the Google
   path is verified working live — Phase B and Phase C collapse into effectively one step.

### 11.1 Impact on the phasing (§6)

With no soak period, Phase B ("Google primary, GitHub legacy") is momentary: once Google sign-in is
verified on the production origins, the same or immediately-following PR deletes the GitHub PAT/Gist
code (Phase C). A one-time in-app migrator still runs so existing Gist-sync users can pull their trips
into Google before the old path disappears.

---

## 12. Proposed next step

On sign-off of this plan, proceed to **Phase A**: implement Google Sign-In + Drive publish / refresh /
share inside `GoTools.html` behind a placeholder `GOOGLE_CLIENT_ID`, keeping the GitHub path intact,
delivered as one PR — mirroring the per-phase cadence used for the data-registry work. **Blocking
input for Phase A:** the **OAuth Client ID** from the owner's Google Cloud project (§7) — code can be
written against a placeholder, but cannot be verified live without it.
