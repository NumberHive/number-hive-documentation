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
| Dev — shared box `ripper` (`npm run dev:local`) | `http://ripper:20000` | `http://ripper:20001/graphql` | `CLAUDE.md` lines 54–58 |
| Dev — backend/frontend run standalone (per `CLAUDE.md`'s own "Environments" table) | `http://localhost:8080` | `http://localhost:3000` | `CLAUDE.md` lines 105–108 |
| Staging — **staging-local (onyx, canonical)** | `https://staging-local.numberhive.org` | `https://staging-local.numberhive.org/graphql` (same origin, path-routed — `/graphql` and `/health` go to the backend container, everything else to frontend) | `STAGING-LOCAL.md` lines 39–43 |
| Staging — Render (per `STAGING-LOCAL.md`: ~~legacy, decommissioned~~; per `docs/operations.md`: still the live "everyday changes" path) | `numberhive-frontend-staging.onrender.com` | `numberhive-backend-staging.onrender.com` | `render.yaml` (root); `docs/operations.md` lines 70–86 (health-check curl, dated 2026-04-22); **contradicted** by `STAGING-LOCAL.md`'s explicit *"Render.com staging has been fully decommissioned"* claim — see caveat below, genuinely unresolved |
| **Production** — corrected 2026-07-27, **confirmed by the user same day** | `https://play.numberhive.org` (migrating to `play.numberhive.app` — not yet live) | `https://backend.numberhive.app` | Frontend: `backend/.env.prod` line `client_base_url=...`, validated directly in `SHIPPING.md`'s production ship-log entries ("Visit `play.numberhive.org` as a new user"). Backend: `backend/.env.prod` line `server_base_url=...` (the real production secrets file — also holds live Stripe keys), **independently confirmed against the live Render dashboard** per `docs/conventions/cookie-consent-management.md`'s history (2026-07-26 entry). Also reachable via Render's own default `numberhive-backend-production.onrender.com` (used for health checks in `docs/operations.md` and `SHIPPING.md`) — same service, two hostnames, not a conflict. |

**Three separate dev configs exist side by side in this repo** (full-stack Docker, ripper
shared-box Docker, and standalone backend/frontend) — pick whichever matches how you're
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

**Staging is similarly unresolved, not just production.** `docs/operations.md` (last touched
2026-04-22, i.e. months before `STAGING-LOCAL.md`/`subdomain-map.md`'s 2026-07-24 pass)
describes Render staging as the live, everyday path — but the more recent documents say it was
decommissioned in favour of `staging-local.numberhive.org` on Onyx. Given the same repo just
turned out to have a stale-but-undeleted production config (Pulumi, above), it's plausible
`docs/operations.md`'s Render-staging instructions are similarly stale and just haven't been
removed — but that's an inference, not confirmed the way the production correction above is.
**Someone with access to the actual Render dashboard should confirm whether
`numberhive-backend-staging`/`numberhive-frontend-staging` services still exist and receive
traffic** — this table cannot resolve that from source code alone.

**Known drift, not corrected here (flagging only):**
- `CLAUDE.md`'s own "Environments" table (line 109) lists Production as `TBD` / `TBD` — stale, and (per the correction above) was never actually reconciled even after `backend.numberhive.app` became knowable.
- `frontend/env.ts`'s `stagingEnv` block points at `api-staging.numberhive.app` / `staging.numberhive.app` — a **fourth** staging candidate, distinct from both `staging-local.numberhive.org` and the `*-staging.onrender.com` pair above. Whether this is live, dead, or the actual Render staging path under a different custom-domain name is not resolved by anything read for this document — one more reason the staging picture above is flagged unresolved rather than picked.
- `backend/render.yaml` (the standalone one, not root) separately sets `server_base_url: https://api-staging.numberhive.app` for its staging service, matching `env.ts`'s guess — these two at least agree with each other, just not with `staging-local.numberhive.org`.

---

## 3. Public Game — `game` (frontend) + `game-api` (backend) — `number-hive-newvis`

| Env | Frontend | Backend (`game-api`) | Source |
|---|---|---|---|
| Dev — local (`npm run dev`) | `http://localhost:3000` (Vite) | `http://localhost:3001` (backend `.env.example` default `PORT`) | `package.json` line 6; `vite.config.ts` lines 9–15 |
| Dev — coder-team preview box | `https://nhvis.puddicombe.com` → Tailscale IP `100.64.21.9:20056`, whatever dev-server preview is currently running; **502 when no preview is active** | same origin, `/api` and `/v1` proxied | `DEPLOY.md` line 14 |
| Staging — **Onyx/Docker Compose (long-standing canonical)** | `https://play.puddicombe.com` → `onyx.prawn-mamba.ts.net:4173` | bundled with frontend, same container/path | `DEPLOY.md` lines 15, 132 |
| Staging — Render (parallel rehearsal, ADR-004 target shape) | `https://staging-game.numberhive.app` | `https://staging-game-api.numberhive.app` | `render.yaml` lines ~16–17, service `numberhive-staging-frontend`/`-backend` |
| **Production** | `https://game.numberhive.app` — **Live** (confirmed + smoke-tested 2026-07-26) | `https://game-api.numberhive.app` — **Live** (confirmed + smoke-tested 2026-07-26) | `render.yaml` services `numberhive-game-production-frontend`/`-backend`; `DEPLOY.md` lines 8, 17–18, 51–62 |

**Naming collision worth knowing about:** this repo's Onyx *staging* frontend is
`play.puddicombe.com` — nothing to do with the education app's *production* domain
`play.numberhive.org` (different repo, different product). `puddicombe.com` is a
personal/dev domain used for internal previews across repos; `numberhive.org`/`.app` are
the real product domains. Easy to misread as the same "play" — they are not.

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
| Dev — coder-team preview box (ripper) | `https://ripper.prawn-mamba.ts.net:20167` (dynamic reserved port, coder-team Dev Front) | `/api/*` proxied through the same origin (internal port `20166`) | `dev_server_status` (coder-team), 2026-08-01 |
| Staging | Not yet built — no staging config exists in this repo yet | — | — |
| **Production** | `https://admin.numberhive.org` — **planned, not yet live** (`.env.example` comment: *"In prod (once live)"*) | Same origin (single Express app serves built client + API in prod — `npm run build && npm start`) | `.env.example` line 24; `README.md` "Production build & start" section |

This is the newest, least-built-out surface — only local dev is real today; staging and
production are still target state, not deployed config.

**Superseded (was here 2026-08-01, corrected 2026-08-02):** this section previously described
a self-hosted **ClickHouse** container on `ripper` as `number-hive-admin`'s analytics event
store (MVP0.1, CHG-3975/CHG-3976). That's no longer accurate — CHG-4093 (2026-08-02) migrated
the events pipeline off ClickHouse onto the same Admin Postgres database, to unblock the
Render deployment (Render doesn't run a separate ClickHouse container). The ClickHouse dev
container, `@clickhouse/client` dependency, and all ClickHouse-specific admin tooling were
removed as part of that change; no historical dev data was carried over (fresh empty table).

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

## 5. Amber — AI staff member — `amber`

*(Not one of the four surfaces in the original ask, added per follow-up request — Amber is
a peer product in the ecosystem, not a customer-facing surface, so it's a separate table
rather than folded into the others above.)*

| Env | App | Source |
|---|---|---|
| Dev — local (`npm run dev`) | Web (Vite) `http://localhost:5173`; API `http://localhost:4000` | `README.md` lines 69, 80–81 |
| Staging | Not documented — no staging environment exists; Amber has only dev and a single deploy target | — |
| Production / deploy target | `https://amber.numberhive.org` (target domain; `docs/deploy.md` treats the actual nginx/DNS wiring as a separate, out-of-scope ops step — **live status not independently confirmed**) — deploys via Docker Compose to host **onyx**, API port `4000` | `docs/deploy.md` lines 3, 7, 23, 33 |

**Supporting infra (not a NumberHive repo, no dev/staging split):**
`planner.numberhive.org` — self-hosted Plane issue tracker backing Amber's task-agent
scheduler. Confirmed live/reachable — referenced as the real API host in Amber's own
operational curl/troubleshooting commands (`docs/deploy.md` lines 445–446, 552–553), not
just a design target.

**Port collision worth flagging:** Amber's dev web app and `number-hive-admin`'s dev Vite
server both default to `5173`. Harmless if the two are never run side-by-side on the same
machine, but both are documented as run on the shared box `ripper` — worth a second look if
someone hits "port already in use" running both at once.

---

## 6. Summary — what's solid vs. what's still open

| Surface | Dev | Staging | Production |
|---|---|---|---|
| `www` | — | — | ✅ Live |
| `play` + backend | ✅ (3 variants) | ⚠ 4 disagreeing candidates on record, none confirmed live the way production now is | ✅ Live — `play.numberhive.org` / `backend.numberhive.app` (corrected + user-confirmed 2026-07-27; Kubernetes/Pulumi path confirmed abandoned; frontend migration to `.app` still pending) |
| `game` + `game-api` | ✅ | ✅ two parallel staging paths, both live | ✅ Live |
| `admin` | ✅ | ❌ not built | ❌ planned only |
| `amber` | ✅ | ❌ none | ⚠ deploy target defined, live status unconfirmed |

---

*Created 2026-07-27, compiled directly from each repo's own deploy config
(`number-hive-complete`, `number-hive-newvis`, `number-hive-admin`, `amber`) rather than
from `subdomain-map.md`'s narrative, so it could serve as an independent cross-check.
See [`subdomain-map.md`](subdomain-map.md) for the *why* behind the domain scheme (the
`.app`/`.org` convention, cross-property tracking implications) — this document is the
*what's actually configured, right now* companion to it.*
