# Subdomain Map & Cross-Property Visitor Tracking

**Status:** Living reference. Domain structure is now substantially **agreed** (confirmed directly
by the user 2026-07-24, corroborated by `number-hive-complete`'s ADR-001, which is itself in
"agreed" status). What remains open is listed in §5.

**Why this exists:** the user asked (2026-07-24) to record the live domain picture — a WordPress
marketing site, a public game, and an education area — and to capture the current view on
cross-property visitor tracking. This document is the "future us remembers" answer, synthesised
from specs scattered across `number-hive-complete` and `number-hive-newvis` that had not
previously been pulled together in one place, then corrected in a follow-up pass once the user
supplied the actual live URLs and the `.app`/`.org` split decision.

---

## 1. The domain convention (agreed 2026-07-24)

**Customer-facing properties use `.app`. NumberHive-internal/staff properties use `.org`.**

| Subdomain | Product | Owning repo | Status |
|---|---|---|---|
| `www.numberhive.app` | **WordPress marketing site** | Not a NumberHive git repo — externally hosted WordPress | **Live** |
| `play.numberhive.org` | Education app — students, teachers, **and NH staff admin (bundled in today)** | `number-hive-complete` | **Live** (current production) — **migrating** to `play.numberhive.app`, with NH admin removed from it as part of the move |
| `staging-game.numberhive.app` | Staging environment for the free game | `number-hive-newvis` | **Live** (staging only) |
| `game.numberhive.app` | Public free game (Phaser/Vite PWA) | `number-hive-newvis` | **To be created** — production DNS not yet live |
| `game-api.numberhive.app` | Free game backend API (Fastify) | `number-hive-newvis` | **To be created** alongside `game.numberhive.app`. Naming already established and agreed in ADR-004/ADR-001 (`game-api`, not bare `api`, to avoid implying a shared/ecosystem-wide API — see §4). |
| `admin.numberhive.org` | `number-hive-admin` — company-ops admin (billing, customer records, staff tooling), split out of `number-hive-complete` | New repo, **not created** | **To be created.** Note: this corrects ADR-005 in `number-hive-complete`, which proposed `admin.numberhive.app` (2026-07-10, before the `.app`/`.org` convention below existed) — see §4. |
| `amber.numberhive.org` | Amber — NumberHive's AI staff member (persona chat, email/calendar/Drive integration, autonomous task agent) | `amber` | Target deploy domain, per `amber`'s own `docs/deploy.md` (`nginx/DNS (amber.numberhive.org → onyx)` and the Plane-webhook URL example both name it) — that same doc treats the actual nginx/DNS wiring as a separate, out-of-scope ops step, so live status isn't independently confirmed here. Fits the `.org` = staff-internal convention regardless. |
| `planner.numberhive.org` | Self-hosted Plane instance — issue tracking backing Amber's task-agent scheduler | Not a NumberHive git repo — self-hosted Plane, external infra (like WordPress above) | **Live** — `amber`'s docs reference it as the real, reachable API host in operational curl/troubleshooting commands, not just a design target. |

**Amber and the admin facility:** per the user (2026-07-26), linking Amber to `admin.numberhive.org` once it exists — so she can interact with the NumberHive team using data pulled from the platform — is an intended direction, not just a speculative line on a diagram. `architecture/platform-overview.md` already draws this as a dotted "scoped, read-only API, narrow, widens deliberately" edge from Amber to the admin API; nothing in `amber`'s own docs describes that contract yet (checked 2026-07-26 — no `entitlement`/`admin` references found in `amber/ARCHITECTURE.md`), so it remains designed-not-implemented on both sides. Whoever builds it should treat `number-hive-complete`'s ADR-005 (admin/Amber data-access rationale) as the starting point.

**The `.app` vs `.org` logic, spelled out:** everything a customer (student, teacher, parent, or an anonymous visitor) ever loads in a browser sits on `.app`. `admin.numberhive.org` is where NH staff work — it's deliberately a different registrable domain, not just a different subdomain of `.app`, which also means it does **not** automatically share cookies with the customer-facing properties (see §3) — a reasonable side-effect for staff-only tooling that shouldn't need customer identity anyway.

### Reconciling with what was already documented before this correction

- `number-hive-complete/docs/adr/001-free-game-infrastructure.md` — status **"agreed"** — already has a URL table matching this almost exactly: `www.numberhive.app` (marketing), `game.numberhive.app` (free game), `game-api.numberhive.app` (free game backend), and `play.numberhive.app | Educational app (migrating from play.numberhive.org)`. This document independently confirms the migration direction the user just gave verbally — good, no conflict there.
- `number-hive-complete/docs/adr/005-numberhive-admin-separation-and-amber-data-access.md` — status **"proposed, pending review"** — proposed `admin.numberhive.app` (tentative). The user's `.org` instruction **supersedes that specific detail**. ADR-005 itself (the decision to split admin into its own service at all) is unaffected — only the TLD guess in its URL table needs updating. **This needs to be corrected in `number-hive-complete` directly** (out of scope for this Assistant to edit — that repo has its own Lead) — flagging here so it isn't lost.
- This repo's own `platform-strategy.md` (June 2026) is now doubly superseded: first by ADR-001/005's `.app`-only guesses, now by the `.app`/`.org` split. Its open-decisions row already points here; no further edit needed there.

---

## 2. What "NH admin is removed" from `play.numberhive.org` means

Per the user: when `play.numberhive.org` migrates to `play.numberhive.app`, NH staff admin (today bundled into the same React Native Web app per `page-inventory.md` §5 — 12 admin screens under `/admin/*`) does **not** come along. It moves to `admin.numberhive.org` instead. This is the domain-level confirmation of the split ADR-005 already proposed at the code/service level (own database, own auth, own deploy pipeline, entitlement data pushed from admin to the public product rather than shared). The two decisions — service split (ADR-005) and domain split (`.org`) — are consistent and reinforce each other: staff tooling isn't just a separate deploy, it's a separate domain, so there's no accidental cookie/session bleed between the public product and company-ops tooling either.

---

## 3. Tracking/identity architecture — current state (not centralised, but the new domain scheme helps)

There is **no single, centralised tracking facility** across the properties today. There are two independently-built visitor-identity systems, plus a WordPress site that isn't confirmed to be wired into either. See `docs/conventions/analytics-and-ops-logging.md` (this repo) for the audience-separation rules any of this must respect once it is connected.

### 3a. `play.numberhive.org` → `play.numberhive.app` (`number-hive-complete`) — the more mature pipeline

Documented across two approved specs in `number-hive-complete/docs/superpowers/specs/`:

- **`2026-05-16-attribution-architecture-design.md`** (Approved, Phase 1 live) — defines `nh_vid`, a client-generated UUID in `localStorage`, a `Visitor` MongoDB collection (first-touch UTM/referrer/country, frozen after first write), `POST /api/visitor/identify` (create-or-touch), `POST /api/visitor/link` (stitches anonymous → authenticated `User` at signup). CORS is already wildcarded on these endpoints — the spec explicitly notes this "future-proofs for marketing site."
- **`2026-05-17-main-app-event-tracking-design.md`** (Approved) — extends this to a server-authoritative `Session` model and a full event taxonomy (`auth`, `onboarding`, `game`, `journey`, `gym`, `hive`, `subscription`, `nav`), plus an explicit **`marketing` area reserved for future use** (`page_view`, `cta_clicked` from marketing pages) and a designed **cross-domain handoff**: because `localStorage` is domain-scoped, a marketing site linking into the app would need to append `?nh_vid=<uuid>` to outbound links, which the app's `visitorIdentity.ts` reads on boot and adopts if no local `nh_vid` exists.

This is the closest thing to a "centralised" design — but it was designed with a marketing site as a **future intended caller**, not one that's confirmed to be calling it. Nothing in either repo confirms the WordPress site actually fires `POST /api/visitor/identify` / `POST /api/track`, or emits the `?nh_vid=` param on its outbound links.

**The new `.app`-for-customers convention makes this easier, not just harder.** `www.numberhive.app`, `game.numberhive.app`, and (once migrated) `play.numberhive.app` are all subdomains of the same registrable domain, `numberhive.app`. That means the **shared first-party cookie approach originally floated in `platform-strategy.md` §8** (`Domain=numberhive.app`, one cookie readable by all three) is now actually straightforward to implement — no cross-domain hacks needed, just a server-side `Set-Cookie` with the right `Domain` attribute (server-side matters because Safari ITP caps JS-set cookies at 7 days — already noted in `platform-strategy.md`). This would be a more robust mechanism than the `?nh_vid=` URL-param handoff currently designed, which only survives a single click-through and does nothing for a visitor who arrives at the app directly in a later session. Worth revisiting once the `.app` migration lands — the URL-param handoff and the shared-cookie approach aren't mutually exclusive (param handoff still helps for the very first click-through; cookie carries identity on every subsequent visit).

`admin.numberhive.org` is intentionally **outside** this cookie domain — correct, since it shouldn't need or want customer visitor identity.

### 3b. `game.numberhive.app` (`number-hive-newvis`) — a separate, isolated identity system

Its own `clientId` (UUID, `localStorage`-persisted) and `fg_visitors` collection, in the **separate `free_game` MongoDB database** (deliberately isolated from `school_hive` — see ADR-001 precedent). No shared identity with `nh_vid` today. `platform-strategy.md` (this repo) proposes eventually retiring this stub backend so the free game calls the paid backend directly, which would collapse this into one identity space — but that's still "under consideration," not decided.

Earlier design intent (`NumberHive_Free_Game_Product_Tech_Spec.md`, principle P5) explicitly wanted the free game to dovetail with the `play.numberhive.org` stack "to avoid repeating the cross-domain attribution break documented in the *Marketing Site Migration* note" — **that referenced note could not be located in either repo.** Either it was never written, or it exists somewhere untracked. Worth asking whoever wrote that spec, or treating it as lost and rewriting the reasoning fresh.

### 3c. `www.numberhive.app` (WordPress) — unknown/unintegrated

No documentation in either code repo describes what tracking (if any) runs on the WordPress site today — presumably whatever a WordPress analytics plugin provides by default (if one is installed), with **no confirmed connection** to the `nh_vid` pipeline described in 3a. This is the actual gap behind the user's original question: the identity-stitching mechanism has been *designed* (URL-param handoff; now also plausible via shared cookie, §3a), but not *implemented or confirmed* on the WordPress side.

### Net assessment

| Property | Identity model | Database | Linked to the others? |
|---|---|---|---|
| `www.numberhive.app` (WordPress) | Unknown/unconfirmed | Unknown (not a NumberHive Mongo db) | **No confirmed link** — handoff mechanism designed, not confirmed implemented |
| `game.numberhive.app` | `clientId` / `fg_visitors` | `free_game` (separate Atlas db) | **No** — isolated identity space |
| `play.numberhive.org` / `.app` | `nh_vid` / `Visitor` | `school_hive` (same Atlas project) | — (this is the pipeline the others would link into) |
| `admin.numberhive.org` *(future)* | N/A — staff auth only, no customer visitor tracking | Own database (ADR-005) | Deliberately **not** linked — different concern entirely |

So: **three potentially separate visitor-identity spaces today (soon a fourth, staff-only domain that's deliberately out of scope for visitor tracking), one shared Atlas project for the customer-facing two, one deliberate database isolation (free game), one designed-but-unconfirmed handoff (marketing → app), and now a same-registrable-domain opportunity (all `.app`) that makes a shared-cookie approach realistic once the `play` migration happens.**

---

## 4. Backend API domain naming — open question, with a recommendation

The user asked: *"other domains for back-end apis?"* Here's what's already decided vs. what needs a decision:

**Already agreed, for the free game:** `game-api.numberhive.app`, distinct from the `game.numberhive.app` frontend. ADR-004 (`number-hive-complete`, offline-first-and-CDN) is explicit about why: *"Named `game-api`, not the bare `api`, to keep the naming consistent with the frontend... and to avoid implying this is a shared/ecosystem-wide API — it is this product's backend only."* This is a real architectural point, not just a naming preference: a bare `api.numberhive.app` would look like a shared/central API, which doesn't exist and isn't planned — each product's backend is independently owned (ADR-001, ADR-005 both make this same "one writer per data domain" point in different words).

**Confirmed live — the education app's backend already has its own origin.** Checked against the actual production infrastructure config (`backend/iac/kubernetes/Pulumi.production.yaml` — Pulumi/Kubernetes, not the `render.yaml` files, is the real live deployment):

| | Value |
|---|---|
| Frontend (`client_base_url`) | `https://play.numberhive.org` |
| Backend (`server_base_url` / `domain`) | `https://api.numberhive.org` — bare `api.` prefix, `.org` TLD. Also reachable via a legacy alias `api.hive.mightybyte.us` (MightyByte was the original dev agency/domain name, kept alive alongside the newer one). |

Staging mirrors this: frontend `number-hive-staging.web.app` (Firebase Hosting), backend `api.staging.numberhive.org` (+ legacy `api.staging.hive.mightybyte.us`).

This means there are now **three disagreeing backend-domain conventions on record for this one product**, none of which match each other:
1. **`api.numberhive.org`** — what's actually live (Pulumi/Kubernetes)
2. `api-staging.numberhive.app` / `api.numberhive.app` — baked into `backend/render.yaml` and `frontend/env.ts`'s prod fallback respectively; these look like a separate, likely-unused/exploratory Render migration attempt, not what's live
3. `game-api.numberhive.app`-style naming (`play-api.numberhive.app`) — the pattern ADR-004 already established for the free game, not yet applied here

**Recommendation** (not yet agreed — flagging for a decision): when `play.numberhive.org` migrates to `play.numberhive.app` per the agreed convention, the backend needs an explicit decision too, not just an implicit carry-over. Two options: (a) rename the backend to **`play-api.numberhive.app`**, matching `game-api.numberhive.app` and staying fully on `.app`; or (b) leave the backend on `.org` as `api.numberhive.org` while the frontend moves to `.app` — which would work technically (nothing requires frontend/backend to share a TLD) but breaks the "customer-facing = `.app`" rule as a *rule*, and reintroduces exactly the `.render.yaml`-vs-`env.ts`-vs-live inconsistency already found here. (a) is recommended for consistency; either way this needs the stray `api-staging.numberhive.app` / `api.numberhive.app` references in `render.yaml` and `env.ts` reconciled against whatever is decided.

**For admin, once `number-hive-admin` exists:** following the same logic, **`admin-api.numberhive.org`** if the admin frontend and backend end up on separate origins (unclear yet — ADR-005 doesn't specify this level of detail, only "separate backend service... separate frontend/admin UI build").

**Where does the WordPress site's tracking calls go?** If/when `www.numberhive.app` starts firing `POST /api/visitor/identify` / `POST /api/track` (§3c), the natural target is whichever backend owns the `Visitor`/`TrackEvent` collections today — that's `number-hive-complete`'s backend, i.e. `play-api.numberhive.app` under the recommendation above (or `play.numberhive.org`'s existing origin today, pre-migration). No new "marketing-api" domain is needed — the attribution spec already designed these endpoints to be called cross-origin (CORS wildcarded) from exactly this kind of external caller.

---

## 5. Documentation that exists and should be treated as the source material to unify

| Doc | Repo | Covers |
|---|---|---|
| `architecture/platform-strategy.md` | this repo | Original two-frontend split proposal; domain names superseded by ADR-001/ADR-005 and now the `.app`/`.org` convention here; §8 (shared-cookie cross-property tracking) is newly relevant again — see §3a above |
| `docs/adr/001-free-game-infrastructure.md` | `number-hive-complete` | **Agreed** URL architecture table — matches this document almost exactly (missing only the admin domain, since that postdates it) |
| `docs/adr/004-offline-first-and-cdn.md` | `number-hive-complete` | Establishes the `game-api` (not bare `api`) naming rationale — the precedent this doc's §4 recommendation extends to `play-api` |
| `docs/adr/005-numberhive-admin-separation-and-amber-data-access.md` | `number-hive-complete` | Admin service split rationale; its `admin.numberhive.app` domain guess is corrected to `.org` by this document |
| `docs/superpowers/specs/2026-05-16-attribution-architecture-design.md` | `number-hive-complete` | `nh_vid`/`Visitor` model, UTM/geo capture, anonymous→authenticated stitching |
| `docs/superpowers/specs/2026-05-17-main-app-event-tracking-design.md` | `number-hive-complete` | Session model, full event taxonomy, marketing-site cross-domain handoff design |
| `docs/architecture.md`, `DEPLOY.md`, `docs/launch-prep.md` | `number-hive-newvis` | Live/DNS status of `game.numberhive.app` / `staging-game.numberhive.app`; confirms WordPress site's existing DNS/CloudFront setup is untouched by other subdomain work |
| `NumberHive_Free_Game_Product_Tech_Spec.md` (duplicated at repo root **and** under `docs/` — worth flagging to that repo for de-duplication) | `number-hive-newvis` | Free game's separate anonymous identity model; references the missing "Marketing Site Migration" note |
| `docs/conventions/analytics-and-ops-logging.md` | this repo | Cross-repo rules any unified tracking approach must respect (audience separation, no bespoke query UI, scoped credentials) |

None of these currently cross-reference each other. This document is the first place that does.

---

## 6. Open items (not decided — flagging, not deciding, here)

1. **Correct ADR-005 in `number-hive-complete`** — its domain guess (`admin.numberhive.app`) needs updating to `admin.numberhive.org` to match the convention agreed here. Out of scope for this Assistant (different repo/Lead) — needs relaying.
2. **What happens to the backend domain when `play` migrates to `.app`** — the live backend today is `api.numberhive.org` (confirmed via Pulumi/Kubernetes prod config), not `play-api.numberhive.app`. Renaming to match `game-api.numberhive.app` is recommended but not yet agreed. Either way, `number-hive-complete`'s `render.yaml` files and `frontend/env.ts`'s prod fallback (`api.numberhive.app` / `api-staging.numberhive.app`) disagree with what's actually live and need reconciling by whoever owns that repo's deploy config.
3. **Does the WordPress site actually track visitors, and does it implement (or could it implement) the shared-cookie or `nh_vid` handoff?** Unknown — someone with WordPress admin access needs to check what's actually installed today.
4. **The missing "Marketing Site Migration" note** — referenced by `number-hive-newvis`'s tech spec, not found in either repo.
5. **Whether/when `number-hive-newvis`'s identity space merges into `number-hive-complete`'s** (per `platform-strategy.md`, still "under consideration").
6. **Old `.org` references in free-game code/docs that predate even the current convention's confirmation** — `EducatorCTA.ts`'s `OUTBOUND_BASE = 'https://play.numberhive.org'` is actually *correct* right now (that's the live URL), but will go stale the moment `play.numberhive.app` goes live and needs updating as part of that migration, not left behind as debt.

---

*Created 2026-07-24, synthesised from the repos above at the user's request. Revised same day after
the user supplied the live URLs and the `.app` (customer) / `.org` (NH-internal) domain convention.*
