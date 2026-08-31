# Environment URL Reference

**Purpose:** a single scannable table of every NumberHive URL/port, per environment
(dev/staging/production), across all repos. Where `subdomain-map.md` explains *why* the
domain scheme is what it is, this document is the quick-reference "what do I actually type
into a browser or curl" answer — pulled directly from each repo's own deploy config
(Pulumi, `render.yaml`, `docker-compose*.yml`, `.env.example`, `CLAUDE.md`/`DEPLOY.md`), not
from memory or convention alone. Every cell below is cited to its source file so it can be
re-verified as configs drift.

**Status:** first pass, built 2026-07-27 by reading each repo's live deploy config directly.
Several rows below are internally inconsistent *within their own source repo* (stale docs
disagreeing with live config) — these are flagged rather than silently resolved, since
picking one would hide a real drift problem someone should go fix at the source.

---

## 1. Marketing — `www` (WordPress)

| Env | URL |
|---|---|
| Dev | Not documented — externally hosted WordPress, not a NumberHive git repo, no known dev/staging environment |
| Staging | Not documented |
| Production | `https://www.numberhive.app` — **Live** |
| Admin | `https://www.numberhive.app/login` — WordPress wp-admin login. Credentials held by the user, not recorded here (see note below). |

No repo to cite — this is the one property with no NumberHive-owned source config at all.

**Credentials are deliberately not stored in this document.** This repo is a plaintext,
git-tracked documentation store — anything committed here stays in git history indefinitely
and is readable by anyone with repo access, so it's the wrong place for a live admin password
even if it's convenient to note alongside the login URL. Noted verbally to the user in-session
instead; see chat history 2026-07-27 if the password needs retrieving, or better, move it into
a real password manager if it isn't in one already.

---

## 2. Education App — `play` (frontend) + backend — `number-hive-complete`

| Env | Frontend | Backend | Source |
|---|---|---|---|
| Dev — full stack incl. Temporal (`npm run dev`) | `http://localhost:3500` | `http://localhost:3400/graphql` | `CLAUDE.md` lines 72–77 |
| Dev — shared personal dev box (developer-provided infrastructure) (`npm run dev:local`) | `http://<personal-dev-box>:20000` | `http://<personal-dev-box>:20001/graphql` | `CLAUDE.md` lines 54–58 |
| Dev — backend/frontend run standalone (per `CLAUDE.md`'s own "Environments" table) | `http://localhost:8080` | `http://localhost:3000` | `CLAUDE.md` lines 105–108 |
| Staging — **staging-local (personally-hosted, developer-provided infrastructure — retired at handover)** | `https://staging-local.numberhive.org` | `https://staging-local.numberhive.org/graphql` (same origin, path-routed — `/graphql` and `/health` go to the backend container, everything else to frontend) | `STAGING-LOCAL.md` lines 39–43 |
| Staging — Render (**the company's staging from 1 Oct** — see resolution below) | `numberhive-frontend-staging.onrender.com` / `numberhive-staging-frontend.onrender.com` (two service-naming generations exist) | `numberhive-backend-staging.onrender.com` / `numberhive-staging-backend.onrender.com` | `render.yaml` (root); confirmed directly via the Render API 2026-08-31 — see resolution below |
| **Production** — corrected 2026-07-27, **confirmed by the user same day** | `https://play.numberhive.org` (migrating to `play.numberhive.app` — not yet live) | `https://backend.numberhive.app` | Frontend: `backend/.env.prod` line `client_base_url=...`, validated directly in `SHIPPING.md`'s production ship-log entries ("Visit `play.numberhive.org` as a new user"). Backend: `backend/.env.prod` line `server_base_url=...` (the real production secrets file — also holds live Stripe keys), **independently confirmed against the live Render dashboard** per `docs/conventions/cookie-consent-management.md`'s history (2026-07-26 entry). Also reachable via Render's own default `numberhive-backend-production.onrender.com` (used for health checks in `docs/operations.md` and `SHIPPING.md`) — same service, two hostnames, not a conflict. |

**Three separate dev configs exist side by side in this repo** (full-stack Docker, a
personally-hosted shared-box Docker setup, and standalone backend/frontend) — pick whichever matches how you're
actually running it; they are not aliases of each other, they use different ports.

**Production backend — corrected from this document's first pass, and from a claim in
`subdomain-map.md` that was wrong.** This repo had (at least) **three mutually-disagreeing
production backend configs on record**, only one of which is actually live:

1. `api.numberhive.org` (+ legacy alias `api.hive.mightybyte.us`) — from
   `backend/iac/kubernetes/Pulumi.production.yaml`. This document's first pass (and
   `subdomain-map.md` §4, written 2026-07-24) treated this Kubernetes/Pulumi config as **"the
   real live deployment."** **Confirmed abandoned by the user directly, 2026-07-27** — this
   was an earlier infra approach that was never cleaned up from disk, not a live path.
2. `api.numberhive.app` — a hardcoded fallback in `frontend/env.ts`'s `prodEnv` block. Already
   flagged elsewhere (`cookie-consent-management.md`'s own history) as a **code-level default,
   not a confirmed deployed value** — easy to mistake for real because it's committed source,
   not a stale doc.
3. **`backend.numberhive.app`** — `backend/.env.prod`'s actual `server_base_url`, the file that
   also holds the real live Stripe/JWT secrets. **Confirmed correct**, both by direct check
   against the live Render dashboard (recorded in `cookie-consent-management.md`'s 2026-07-26
   history) and by the user's 2026-07-27 confirmation that the Kubernetes alternative (#1) is
   dead. This is the value to treat as production going forward.

**Net effect: `subdomain-map.md` §4's recommendation and backend-domain table need revisiting**
— they're built on top of the now-confirmed-wrong Pulumi/Kubernetes claim. Not corrected in that
document as part of this pass (that section makes an architectural recommendation, not just a
URL statement, and deserves its own review) — flagged here so it isn't missed.

**Staging — resolved 2026-08-31, queried directly via the Render API.** `number-hive-complete`
has two generations of Render staging services on record — `numberhive-backend-staging`/
`numberhive-frontend-staging`/`numberhive-temporal-staging` (the older set, matching
`docs/operations.md`'s naming) and `numberhive-staging-backend`/`numberhive-staging-frontend`
(a newer set). **Both generations exist and both are currently `suspended`** (confirmed via
`GET /v1/services`, not inferred from docs) — neither is serving traffic today, and neither has
a custom domain attached in Render (only their default `.onrender.com` hostnames, if resumed).
`staging-local.numberhive.org` (personally-hosted Docker Compose, per `STAGING-LOCAL.md`) is
the environment that's actually been in day-to-day use — but it is **developer-provided
infrastructure and is retired at handover**, along with the rest of the personally-hosted dev
box. **Practical consequence: the company's staging environment for `number-hive-complete` from
1 October is whatever Render provides — and that means someone needs to un-suspend one of the
two Render staging service sets (recommend retiring the older `*-staging` naming generation and
keeping the newer `staging-*` one, or vice versa — a judgement call, not resolvable from source)
before staging is actually usable again.** This is now a concrete action item, not an open
question — see [`open-items.md`](../handover/open-items.md) #3.

**Known drift, not corrected here (flagging only):**
- `CLAUDE.md`'s own "Environments" table (line 109) lists Production as `TBD` / `TBD` — stale, and (per the correction above) was never actually reconciled even after `backend.numberhive.app` became knowable.
- `frontend/env.ts`'s `stagingEnv` block points at `api-staging.numberhive.app` / `staging.numberhive.app` — a **fourth** staging candidate, distinct from both `staging-local.numberhive.org` and the `*-staging.onrender.com` pair above. Whether this is live, dead, or the actual Render staging path under a different custom-domain name is not resolved by anything read for this document — one more reason the staging picture above is flagged unresolved rather than picked.
- `backend/render.yaml` (the standalone one, not root) separately sets `server_base_url: https://api-staging.numberhive.app` for its staging service, matching `env.ts`'s guess — these two at least agree with each other, just not with `staging-local.numberhive.org`.

---

## 3. Public Game — `game` (frontend) + `game-api` (backend) — `number-hive-newvis`

| Env | Frontend | Backend (`game-api`) | Source |
|---|---|---|---|
| Dev — local (`npm run dev`) | `http://localhost:3000` (Vite) | `http://localhost:3001` (backend `.env.example` default `PORT`) | `package.json` line 6; `vite.config.ts` lines 9–15 |
| Dev — personal preview box (developer-provided infrastructure — retired at handover) | `https://nhvis.puddicombe.com`, whatever dev-server preview is currently running; **502 when no preview is active** | same origin, `/api` and `/v1` proxied | `DEPLOY.md` line 14 |
| Staging — personally-hosted Docker Compose (developer-provided infrastructure — retired at handover) | *(personally-hosted staging; specific domain intentionally not documented here)* | bundled with frontend, same container/path | `DEPLOY.md` lines 15, 132 |
| Staging — Render (**the company's staging from 1 Oct** — confirmed live via the Render API 2026-08-31, `staging` branch, not suspended) | `https://staging-game.numberhive.app` | `https://staging-game-api.numberhive.app` | `render.yaml` lines ~16–17, services `numberhive-game-staging-frontend`/`-backend`; custom domains verified in Render |
| **Production** | `https://game.numberhive.app` — **Live** (confirmed + smoke-tested 2026-07-26) | `https://game-api.numberhive.app` — **Live** (confirmed + smoke-tested 2026-07-26) | `render.yaml` services `numberhive-game-production-frontend`/`-backend`; `DEPLOY.md` lines 8, 17–18, 51–62 |

**⚠ Correction needed in `subdomain-map.md`:** that document (written/confirmed 2026-07-24)
lists `game.numberhive.app` / `game-api.numberhive.app` as *"To be created — production DNS
not yet live."* That's now stale — `number-hive-newvis/DEPLOY.md` confirms both went live
and were fully smoke-tested on 2026-07-25/26, one day after `subdomain-map.md`'s "agreed"
pass. Flagging here; `subdomain-map.md` §1's table should be updated to **Live** the next
time that document is revised.

---

## 4. Admin — `number-hive-admin` (new repo, first slice — Google OAuth sign-in — implemented)

| Env | Frontend | Backend | Source |
|---|---|---|---|
| Dev — local (`npm run dev`) | `http://localhost:5173` (Vite dev server; `PUBLIC_ORIGIN`) | `http://localhost:5174` (Express `SERVER_PORT` — **internal only**, Vite proxies `/auth/*` and `/api/*` to it so the browser only ever sees one origin) | `.env.example` lines 19–31 |
| Dev — personal preview box (developer-provided infrastructure — retired at handover) | `https://<personal-dev-box>:20167` (dynamic reserved port) | `/api/*` proxied through the same origin (internal port `20166`) | `dev_server_status`, 2026-08-01 |
| Staging | Not yet built — no staging config exists in this repo yet | — | — |
| **Production** | `https://admin.numberhive.org` — **not yet reachable at this domain**; the `number-hive-admin` service is deployed and running on Render (branch `main`, auto-deploy on) but `admin.numberhive.org` DNS is not wired to it — it resolves to a parked GoDaddy IP (`199.192.24.221`) with an expired TLS certificate, not Render. Reachable today only at the Render-issued `https://number-hive-admin.onrender.com`. Confirmed directly via the Render API + a live DNS/TLS check, 2026-08-31. | Same origin (single Express app serves built client + API in prod — `npm run build && npm start`) | `.env.example` line 24; `README.md` "Production build & start" section; Render API `GET /v1/services` and `GET /v1/services/{id}/custom-domains` |

This is the newest, least-built-out surface. It is deployed to Render in the sense that the
service exists, builds, and auto-deploys on push to `main` — but it is not yet reachable at its
intended production domain, and staging is genuinely not built at all yet.

**Superseded (was here 2026-08-01, corrected 2026-08-02):** this section previously described
a self-hosted **ClickHouse** container on a personally-hosted dev box as `number-hive-admin`'s analytics event
store (MVP0.1, CHG-3975/CHG-3976). That's no longer accurate — CHG-4093 (2026-08-02) migrated
the events pipeline off ClickHouse onto the same Admin Postgres database, to unblock the
Render deployment (Render doesn't run a separate ClickHouse container). The `@clickhouse/client`
app dependency and all ClickHouse-specific admin tooling/code were removed as part of that
change; no historical dev data was carried over into Postgres (fresh empty table).

**Correction (2026-08-03):** the ClickHouse *container itself* was **not** torn down — it's
still running on a personally-hosted dev box (developer-provided infrastructure, retired at
handover), just stale and unused. `number-hive-admin`'s Lead confirmed its most
recent row dates from 2026-08-01, pre-migration, and independently checked that a 2026-08-03
smoke-test event landed only in Postgres, not ClickHouse. Flagging so nobody mistakes "app code
removed" for "infrastructure decommissioned" — since the container is on developer-provided
infrastructure being retired at handover, it goes away with the rest of that box rather than
needing separate decommissioning.

**Current state:** events land in an `fg_events` table on `number-hive-admin`'s existing
Postgres database — same `DATABASE_URL` as the rest of the admin app, so it doesn't get its
own row in the table above. Dedup is via a `UNIQUE(row_key)` constraint + `ON CONFLICT DO
NOTHING`, replacing ClickHouse's `ReplacingMergeTree`/`FINAL` approach. Filter/facet/search
semantics are unchanged from the original design; only the storage engine and query dialect
changed. This is a mirrored copy of `number-hive-newvis`'s event stream, pushed cross-repo per
[`cross-repo-data-push.md`](../docs/conventions/cross-repo-data-push.md). Shipped to
production 2026-08-02. Postgres has no native TTL the way ClickHouse did, so there's an open
follow-up idea (CHG-4097, not yet built) to add a 90-day retention purge job for `fg_events`.

---

## 5. Summary — what's solid vs. what's still open

| Surface | Dev | Staging | Production |
|---|---|---|---|
| `www` | — | — | ✅ Live |
| `play` + backend | ✅ (3 variants) | ⚠ Render staging exists but is **suspended** (confirmed via Render API); `staging-local.numberhive.org` was the day-to-day environment but is developer-provided infrastructure, retired at handover — someone needs to resume the Render staging services before 1 Oct | ✅ Live — `play.numberhive.org` / `backend.numberhive.app` (corrected + user-confirmed 2026-07-27; Kubernetes/Pulumi path confirmed abandoned; frontend migration to `.app` still pending) |
| `game` + `game-api` | ✅ | ✅ Render staging live (confirmed via Render API, not suspended) — this is the company's staging today | ✅ Live, friends-and-family testing, not yet publicly launched |
| `admin` | ✅ | ❌ not built | ⚠ Service deployed and running on Render, but `admin.numberhive.org` DNS not wired to it (parked GoDaddy IP, expired cert) — confirmed via Render API + live DNS/TLS check |

---

*Created 2026-07-27, compiled directly from each repo's own deploy config
(`number-hive-complete`, `number-hive-newvis`, `number-hive-admin`) rather than
from `subdomain-map.md`'s narrative, so it could serve as an independent cross-check.
See [`subdomain-map.md`](subdomain-map.md) for the *why* behind the domain scheme (the
`.app`/`.org` convention, cross-property tracking implications) — this document is the
*what's actually configured, right now* companion to it.*
