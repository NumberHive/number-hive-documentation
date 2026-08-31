# Tech Inventory — NumberHive Ecosystem

**Purpose:** the concrete list of domains, hosting, databases, third-party services, and costs
behind NumberHive — reconciled against what's actually deployed today, per the same discipline as
[`../architecture/environment-urls.md`](../architecture/environment-urls.md) (which this
document leans on heavily for the infra rows).

**On credentials:** this repo is plaintext, git-tracked, and kept indefinitely in history —
the wrong place for secrets even briefly. Every place a password/key would go is marked
`[see Google Doc: <item>]`. Those three(ish) items remain in James's Google Doc; this file is
the structural map of *what* exists and *where*, not the secrets themselves.

---

## 1. Domains

| Domain | Registrar | DNS | Next renews | Cost |
|---|---|---|---|---|
| numberhive.org | GoDaddy | GoDaddy | 6 April 2026 | £19.82/yr |
| numberhive.app | GoDaddy | GoDaddy | 6 April 2026 | £20.86/yr |

**DNS access:** [sso.godaddy.com/access](https://sso.godaddy.com/access). Currently informally
delegated (James → account holder "Chris", authority passed via "Fletch") — that's simply how
it stands today. Introducing David into that chain is part of the handover itself — see
[`open-items.md`](open-items.md).

**`.app` vs `.org` convention:** `.app` = customer-facing (`play.numberhive.app`,
`game.numberhive.app`); `.org` = NumberHive-internal/staff (`admin.numberhive.org`,
`staging-local.numberhive.org`). Full rationale in
[`subdomain-map.md`](../architecture/subdomain-map.md).

**DNS is manual, not IaC-managed** — a carryover fact from the dead GKE plan that's still true
today: new services need an A/CNAME record created by hand in GoDaddy, nothing automates this.

---

## 2. Hosting — Render.com (the real, current answer)

**GCP/GKE/Pulumi is dead infrastructure, not the live path.** The backend was originally built
on a GCP/GKE/Pulumi Kubernetes architecture (VPC, private cluster, 3 autoscaling node pools,
Cloud Build triggers). **That path was confirmed abandoned by James on 2026-07-27** — see
`environment-urls.md`'s "GKE/Pulumi correction" section for the full reconciliation. The
Pulumi/`iac/` config still physically exists in `number-hive-complete/backend/iac/` (untouched
since the monorepo merge) but nothing deploys through it. **Whether the GCP project itself
(cluster, node pools, static IP, Cloud NAT) was ever torn down is unconfirmed — see
[`open-items.md`](open-items.md) #1, the single highest-priority item in this whole handover.**

**What's actually live, all on Render, deployed via each repo's own `render.yaml`:**

| Service | Repo | Env | Notes |
|---|---|---|---|
| `numberhive-frontend-production` / `-backend-production` | `number-hive-complete` | Production | `play.numberhive.org` / `backend.numberhive.app` |
| `numberhive-temporal-production` + `numberhive-temporal-pg-production` | `number-hive-complete` | Production | Temporal workflow engine — **still live**, just moved from the old Pulumi/K8s deployment to a Render private service + managed Postgres. Used for game timeouts/timed events. |
| `numberhive-frontend-staging` / `-backend-staging` / `-temporal-staging` + pg | `number-hive-complete` | Staging | Staging status genuinely unresolved — see `environment-urls.md` §2, there's also a parallel `staging-local.numberhive.org` (personally-hosted Docker Compose) that some docs call canonical and others call decommissioned. **Needs a live check**, not resolvable from source alone. |
| `numberhive-game-production-frontend` / `-backend` | `number-hive-newvis` | Production | `game.numberhive.app` / `game-api.numberhive.app` — confirmed live + smoke-tested 2026-07-26 |
| `numberhive-staging-frontend` / `-backend` | `number-hive-newvis` | Staging | `staging-game.numberhive.app` / `staging-game-api.numberhive.app` |
| `number-hive-admin` (single service) | `number-hive-admin` | Production (planned) | `admin.numberhive.org` — target domain, **not yet confirmed live**; this repo is still early (only Google OAuth sign-in + an events mirror shipped so far) |
| `number-hive-admin-db` | `number-hive-admin` | — | Managed Postgres, also backs the events mirror (`fg_events` table) migrated off ClickHouse — see `environment-urls.md` §4 |
| `audit-log-purge` | `number-hive-admin` | — | Render cron job |

**Render dashboard access** is the single most useful thing to get David set up with early — it's
the ground truth for "is X actually live" that no repo's source code can answer alone.

---

## 3. Databases

| Database | Role | Where |
|---|---|---|
| MongoDB (Atlas or self-hosted — **not confirmed which**) | Primary app DB for `number-hive-complete` (Mongoose ODM, `mongoose ^6.3.1` in `backend/package.json`) | `mongodb_url` in `backend/.env.prod` (production secrets file) |
| MongoDB, read-only user | Analyser feature (CHG-1003) | `MONGODB_READONLY_URI`, dedicated read-only Mongo user — recommended to be created fresh, dedicated, not reused from any shared/demo credential |
| Postgres (`numberhive-temporal-pg-production`) | Temporal's backing store | Render managed Postgres |
| Postgres (`number-hive-admin-db`) | Admin app data + `fg_events` mirror table | Render managed Postgres |

**Flagged for rotation:** the MongoDB connection string on record uses a demo-style credential
(`nh-demo:[REDACTED]@cluster0...`) — this looks like a shared/demo cluster, not a dedicated
production one. Worth confirming whether the real production Mongo connection is a dedicated
cluster with its own credentials, or genuinely this one — see [`open-items.md`](open-items.md)
for the recommended next step.

### MongoDB usage per service

| Service | Database | ODM/driver | Notes |
|---|---|---|---|
| `number-hive-complete` backend | primary app DB (`mongodb_url`, `backend/.env.example`) | Mongoose `^6.3.1` | The Education product's whole data model — hives, students, teachers, progression, billing links. |
| `number-hive-complete` backend, Analyser feature (CHG-1003) | same cluster, dedicated **read-only** user (`MONGODB_READONLY_URI`) | Mongoose, read-only aggregation queries | `.env.example` includes the exact `db.createUser({..., roles: [{role: 'read', ...}]})` snippet to provision this — a documented, deliberate least-privilege pattern, not an afterthought. `analyser_query_timeout_ms` caps aggregate() wall-clock (CHG-2383, default 10s). |
| `number-hive-newvis` backend | separate database, `MONGODB_DB_NAME=numberhive_free` on its own connection string (`MONGODB_URI`, `backend/.env.example`) | raw MongoDB driver, no ODM (`backend/src/db.js`) | Entirely separate from `number-hive-complete`'s Mongo — different cluster/db name, no shared schema. Full collection-level detail in [`arcade-data-model.md`](arcade-data-model.md). |
| `number-hive-admin` | **no MongoDB at all** — Postgres only (`DATABASE_URL`, `.env.example`) | — | Confirmed via repo-wide search: zero MongoDB references anywhere in `number-hive-admin`. Its ANALYSER feature (CHG-4140+) is Postgres-based and was built independently — see [`open-items.md`](open-items.md) for the full note on why this isn't a port of `number-hive-complete`'s Mongo-based CHG-1003 Analyser. |

---

## 4. Third-party services (confirmed live via `package.json`/`.env.example` in `number-hive-complete`)

| Service | Role | Evidence |
|---|---|---|
| **Stripe** | Subscriptions/billing | `stripe ^13.4.0` dependency; `stripe_api_key`/`stripe_api_secret`/`stripe_webhook_secret`/`stripe_account_id` in `backend/.env.example`; live keys confirmed present in `backend/.env.prod` (per `environment-urls.md`'s citation) |
| **SendGrid** | Transactional email (production) | `sendgrid_api_key`; `src/utils/sendgrid.ts` |
| **Mailchimp** | Bulk email to subscribers | `mailchimp_api_key`/`mailchimp_dc`/`mailchimp_list_id`; required for prod, optional in dev |
| **Firebase** | `firebase ^10.5.0` in frontend; `firebase_project_id`/`firebase_client_email`/`firebase_base64_private_key` in backend (Admin SDK) | Historically used for hosting (superseded by Render for the education app — see below), Analytics, Cloud Messaging |
| **Mixpanel** | Server-side event analytics | `mixpanel ^0.18.0`; `mixpanel_api_key`; `src/services/analytics.ts` — full event-level detail in [`analytics-inventory.md`](analytics-inventory.md) |
| **Nodemailer** | Local/fallback SMTP | Dev-only fallback — not independently re-verified this pass |

### Stripe integration points and webhooks (`number-hive-complete` only — the other three repos have no Stripe code)

**Webhook endpoint:** `POST /api/webhooks/stripe`, mounted in `backend/src/router/api/webhooks/index.ts:6`
(`router.use('/stripe', stripeWebhook)`), handler in `backend/src/router/api/webhooks/stripe.ts`.

**Signature verification:** `backend/src/utils/stripe.ts:262`, via the SDK's
`stripe.webhooks.constructEventAsync(rawBody, signature, webhookSecret)` — `webhookSecret` reads
from `stripe_webhook_secret` (`backend/src/config/index.ts:203`). The raw body needed for
signature verification is captured globally in `server.ts`'s `express.json({ verify: ... })`
hook, not re-read per-route.

**Events handled** — a `switch` statement in `backend/src/utils/stripe.ts` covers 12 distinct
event types (line numbers as of this pass):

| Event | Line |
|---|---|
| `customer.subscription.deleted` | 265 |
| `invoice.payment_succeeded` | 273 |
| `invoice.payment_failed` | 278 |
| `checkout.session.completed` | 282 |
| `checkout.session.expired` | 296 |
| `customer.subscription.created` | 300 |
| `customer.subscription.updated` | 315 |
| `invoice.finalized` | 331 |
| `invoice.paid` | 335 |
| `invoice.payment_failed` (again) | 339 |
| `invoice.voided` | 343 |
| `invoice.marked_uncollectible` | 347 |
| `charge.refunded` | 352 |

**Bug found this pass:** `invoice.payment_failed` is a `case` twice (line 278 and line 339).
JavaScript `switch` statements match the first matching case top-to-bottom, so the block at
line 339 is dead code — unreachable for that event. Not fixed as part of this handover (out of
scope — documentation only); flagged here and in [`open-items.md`](open-items.md) as a
"confirm with James / worth a real fix" item, since it's not obvious from a glance which of the
two blocks was intended to be authoritative.

**Other Stripe surfaces:**
- Checkout sessions are created with dynamic `price_data` (no pre-created Stripe `Price` objects
  in the Stripe dashboard) — pricing lives in this repo's code, not in Stripe's product catalog.
- Billing portal access via `createPortalSession` (same file).
- `cancelSubscription()` exists in `backend/src/utils/stripe.ts` but no non-test call site was
  found this pass — possibly dead code, possibly cancellation is handled a different way (e.g.
  directly through the Stripe billing portal rather than this app's own code path). Flagged as
  "confirm with James" in [`open-items.md`](open-items.md) rather than assumed dead.
- The CHG-1577 admin-facing Stripe-config UI (subscription/price lookups for support/ops use) is
  separate, live code — see [`open-items.md`](open-items.md) for its status.

**Firebase Hosting — likely stale.** Firebase Hosting was historically the deploy target for
the mobile/web client (`number-hive-staging.web.app`, `play.numberhive.org` via Firebase). The
mobile app repo (`number-hive-app`) hasn't been touched since 2026-03-12 and
`number-hive-complete/CLAUDE.md` states iOS/Android builds are not maintained. **Firebase's role
today is likely limited to Analytics/FCM/Admin SDK (server-side), not hosting** — the actual
production frontend is Render (`numberhive-frontend-production`), not Firebase Hosting. Worth
confirming whether the Firebase Hosting projects (`number-hive-staging`, `number-hive`) still
exist and are billing, same class of question as the GKE cluster. Added to [`open-items.md`](open-items.md).

**Not independently re-verified this pass:**
- Apple App Store (App ID `1636921061`, dev account `james@puddicombe.com`) and Google Play
  Store (package `com.numberhive`) listings — no evidence found of active mobile CI/CD in either
  live repo, consistent with "mobile builds not maintained." These listings likely still exist
  and may still be live to end users even though nobody is building new releases for them —
  **that's a product/business decision (pull the apps vs. leave stale builds live), not
  something this technical review can resolve.**
- Termly (privacy policy/ToS hosting)
- HetrixTools (platform monitoring)

---

## 5. Cost map (as last recorded — needs a fresh pull, not re-verified this pass)

| Item | Provider | Frequency | Amount |
|---|---|---|---|
| Admin workspace | Google | Monthly | ≈ A$159.67 |
| Domain registration ×2 | GoDaddy | Annual | £19.82 + £20.86 |
| Front-end (**historically Firebase — see correction above, may now be Render**) | Google Firebase | Monthly | £135 (stale figure) |
| Back-end (**historically Google Cloud — see GKE correction, may be partly dead spend**) | Google Cloud | Monthly | £61 (stale figure — could include the unconfirmed-torn-down GKE cluster) |
| Database | MongoDB Atlas | Monthly | $161 |
| Render (all services above) | Render.com | Monthly | **Not previously tracked** — Render is the actual current hosting bill and needs adding |
| Marketing | LinkedIn | — | not recorded |
| www | WordPress | — | not recorded |
| Email general | SendGrid | — | not recorded |
| Email bulk | Mailchimp (Intuit) | — | not recorded |
| Stripe, Mixpanel | usage-based | — | not recorded |
| Platform monitoring | HetrixTools | ad hoc | ad hoc |

*Previously recorded total: Google Cloud costs Jan 2026, £134.26 — predates the Render migration, likely
no longer representative of where the actual spend is now.* **A fresh cost pull across GCP,
Render, and Google Workspace billing consoles is needed** — flagged in [`open-items.md`](open-items.md) as it
directly informs whether anything (the GKE cluster in particular) is being paid for with no
purpose.

---

## 6. Secrets management / access

| System | What's there | Access |
|---|---|---|
| Pulumi (stacks: `gke:general`, `kubernetes:staging`, `kubernetes:production`) | State for the now-abandoned GKE path; state bucket `gs://number-hive-pulumi` | `[see Google Doc: Pulumi]` — **relevance now depends on whether the GCP infra is being torn down or kept**; see [`open-items.md`](open-items.md) |
| WordPress admin (`www.numberhive.app/login`) | Marketing site CMS | User `jamesp@numberhive.app`; `[see Google Doc: WordPress]` |
| MongoDB (`nh-demo` user — flagged for rotation) | Looks like a demo/shared credential, not confirmed as the real production one | `[see Google Doc: MongoDB]` — flagged in [`open-items.md`](open-items.md) for David to confirm/rotate once he has Atlas access |
| Mailchimp API key (`NH-demo` key — same pattern) | Same "demo"-named pattern as the Mongo credential above | Held in `backend/.env.prod`, not reproduced here |
| SendGrid API key | Transactional email | Held in `backend/.env.prod` |
| Render dashboard | All live hosting/deploy for `complete`, `newvis`, `admin` | Needs explicit access grant to David — this is now the single most operationally important credential in the whole system |
| GCP console | GKE/Pulumi infra (status unconfirmed), Google Workspace-adjacent billing | Needed to resolve [`open-items.md`](open-items.md) #1 |
| GitHub org (NumberHive) | All four repos | Standard org access, not separately tracked here |

**Access management matrix (GCP / GitHub / Pulumi state bucket / MongoDB / Stripe / Firebase)**
has never existed — access has always been tracked informally, held by James. What handover
actually requires is each of these granted to David directly (see
[`open-items.md`](open-items.md) #6); a standing, documented matrix is an optional process
improvement David can build later if he wants one, not a precondition.

---

## 7. Scheduled jobs and sweeps

| Job | Repo | Mechanism | Schedule | Purpose |
|---|---|---|---|---|
| Digest scheduler | `number-hive-complete` | in-process `node-cron`, `backend/src/services/digest-scheduler.ts:13` (`startDigestScheduler`), started from `server.ts` | Hourly | Batches/sends teacher digest notifications. |
| Financial reconciliation | `number-hive-complete` | in-process `node-cron`, `backend/src/services/digest-scheduler.ts:67` (`startFinancialReconciliationScheduler`), started from `server.ts:120` (`// CHG-1917: Nightly financial reconciliation (02:00 UTC)`) | Nightly, 02:00 UTC | Pulls the last 30 days of Stripe invoices and reconciles them against local records — `backend/src/services/financial-ingestion.service.ts` does the actual ingestion work this job calls. |
| Journey-stage results sync | `number-hive-complete` | in-process `node-cron`, `backend/src/services/journey-stage-results-sync.ts:157` | Every 30 minutes (`*/30 * * * *`) | Feeds the Analyser feature (CHG-1003). A one-time fire-and-forget full backfill also runs on every server boot, separate from the recurring 30-min job. |
| `autoResignSweep` | `number-hive-newvis` | in-process `setInterval` | Hourly, 14-day inactivity threshold | Auto-resigns matches nobody has touched in 14 days. CAS-guarded. Full detail in [`async-arcade-architecture.md`](async-arcade-architecture.md) §7. |
| `offerExpirySweep` | `number-hive-newvis` | in-process `setInterval` | Hourly, 7-day offer threshold | Expires stale pending challenge offers. CAS-guarded. Full detail in [`async-arcade-architecture.md`](async-arcade-architecture.md) §7. |
| `audit-log-purge` | `number-hive-admin` | **Render-native cron service** (not in-process) | Per `render.yaml` | The only job in this ecosystem that runs as an actual Render Cron Job resource rather than an in-process timer — see §2 above. |

**Single-instance risk (unguarded, not currently active):** none of `number-hive-complete`'s
three in-process `node-cron` jobs use a distributed lock. This is not an active risk today
because the backend runs as a single Render instance, but it would silently double-fire (e.g.
double-pulling/double-reconciling Stripe invoices) if the service were ever scaled to multiple
instances without adding a lock first. `number-hive-newvis`'s two sweeps are explicitly
CAS-guarded already (safe to scale) — see [`async-arcade-architecture.md`](async-arcade-architecture.md)
§7 for why that repo's sweeps were built differently. Worth a line in
[`open-items.md`](open-items.md) as a "fragile if scaled" note.

**Temporal is not used as a general cron/scheduling layer**, despite being present in the stack
(§2 above). Its one confirmed production use is a 70-second per-user disconnect timeout
(`onUserDisconnect`). The workflow engine's `startScheduledEvent` capability
(`backend/src/services/timed-events/timed-event.ts`) exists in code but has zero production call
sites found this pass — only referenced from test mocks. Not claimed as dead code (Temporal
workflows can be started from places a static grep won't find), but flagged as
"confirm with James" in [`open-items.md`](open-items.md).

---

## 8. Environment variables per service (names and purposes only — values redacted or unset)

### `number-hive-complete` backend (`backend/.env.example`)

| Variable | Purpose |
|---|---|
| `env`, `port` | Runtime mode (local/development/production), listen port. |
| `server_base_url`, `client_base_url` | Own-service and frontend base URLs. |
| `minimum_mobile_js_version` | Mobile app compatibility gate (mobile is unmaintained — see §4). |
| `mongodb_url` | Primary MongoDB connection string — see §3, flagged for credential rotation. |
| `access_token_base64_private_key` / `access_token_base64_public_key` / `refresh_token_base64_private_key` / `refresh_token_base64_public_key` | JWT signing keypairs for access + refresh tokens. |
| `sendgrid_api_key`, `sender_email_address` | Transactional email. |
| `firebase_project_id`, `firebase_client_email`, `firebase_base64_private_key` | Firebase Admin SDK (server-side — Analytics/FCM, not hosting; see §4). |
| `stripe_api_key`, `stripe_api_secret`, `stripe_webhook_secret`, `stripe_account_id` | Stripe billing + webhook signature verification (see §4 above). |
| `is_timed_events_enabled`, `timed_events_server_address` | Temporal connection (§2, §7 above). |
| `mixpanel_api_key` | Server-side analytics — see [`analytics-inventory.md`](analytics-inventory.md). |
| `mailchimp_api_key`, `mailchimp_dc`, `mailchimp_list_id`, `demo_origin_url` | Bulk email (CHG-589); required for prod cutover, optional in dev. |
| `anthropic_api_key`, `follow_up_model` | Automated personalised follow-up (CHG-318). |
| `MONGODB_READONLY_URI`, `analyser_query_timeout_ms` | Analyser feature (CHG-1003) — see §3 above. |

### `number-hive-complete` frontend (no `.env.example` — build-time vars via `env.ts` / webpack `EnvironmentPlugin`)

| Variable | Purpose |
|---|---|
| `SERVER_URL`, `API_URL`, `SERVER_WEB_SOCKET_URL` | Backend API/websocket endpoints, baked in at build time (not runtime-configurable). |
| `WEBSITE_BASE_URL` | Frontend's own public base URL. |
| `BYPASS_PAYMENT` | Dev/test flag to skip Stripe checkout. |
| `APP_ENV`, `NODE_ENV` | Build target/environment flags. |

### `number-hive-newvis` backend (`backend/.env.example`)

| Variable | Purpose |
|---|---|
| `MONGODB_URI`, `MONGODB_DB_NAME` | Own, separate MongoDB connection — see §3 above. |
| `PORT`, `NODE_ENV` | Runtime. |
| `FRONTEND_ORIGIN`, `APP_ORIGIN`, `API_ORIGIN` | CORS origin, invite-link base URL, and this backend's own public origin (used to build the Google OAuth `redirect_uri`). |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | Web Push notifications — optional, silently disabled if unset. |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Google OAuth (CHG-2329) — **required in every environment**, server fails fast (`process.exit(1)`) if missing. No "disabled" mode, unlike VAPID. |
| `JWT_SECRET` | Signs the 30-day session JWT issued after login. |
| `OAUTH_STATE_SECRET` | Signs the OAuth CSRF `state` token — deliberately a separate secret from `JWT_SECRET` so one leak doesn't compromise both. |
| `EMAIL_ENCRYPTION_KEY` | AES-256-GCM key encrypting account emails at rest, also used as the HMAC-SHA256 pepper for the lookup-only `emailHash` field. |
| `SLOW_REQUEST_THRESHOLD_MS` | Optional. Flags/persists slow requests to `fg_system_events` (CHG-2882). |
| `PROGRESSION_RACE_SAMPLE_RATE` | Optional. Sampling rate for a diagnostic pre-read used to detect `progression_race` events (CHG-4017/CHG-2806). |
| `NEWVIS_EVENTS_PUSH_URL`, `NEWVIS_EVENTS_PUSH_SECRET`, `NEWVIS_EVENTS_PUSH_INTERVAL_MS`, `NEWVIS_EVENTS_PUSH_BATCH_SIZE` | Cross-repo analytics push to `number-hive-admin` (CHG-3977) — optional, silently no-ops if unset. The `_SECRET` is the same coordination value `number-hive-admin`'s `.env.example` calls `NEWVIS_EVENTS_PUSH_SECRET` — see below, the two must match. |

### `number-hive-admin` (`.env.example`)

| Variable | Purpose |
|---|---|
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Staff Google OAuth sign-in (dev client, separate from a future prod client). |
| `ALLOWED_HD_DOMAIN` | Server-side staff-domain gate against the verified ID token's `hd` claim — explicitly not trusted from the OAuth request alone. |
| `SESSION_SECRET` | Session cookie signing — rotating invalidates all sessions. |
| `DATABASE_URL` | This service's own Postgres — also backs the `fg_events` analytics table (formerly ClickHouse, CHG-4093). Render auto-wires this in production. |
| `ENTITLEMENT_PUSH_SECRET`, `ENTITLEMENT_PUSH_TARGET_URL` | Outbound entitlement push, `number-hive-admin` → `number-hive-complete` (ADR-005). HMAC-signed, scoped per-direction. **Real provisioning/rotation not yet designed** — placeholder value only, needed before production ship. |
| `NEWVIS_EVENTS_PUSH_SECRET` | Inbound event push credential `number-hive-newvis`'s push worker authenticates with (see above) — **same "not yet designed for production" caveat**. |
| `OPENROUTER_API_KEY`, `OPENROUTER_MODEL` | ANALYSER LLM integration (CHG-4142, routed via OpenRouter as of CHG-4172), used by `server/src/analyser/anthropicClient.ts`. Model is optional, defaults to a current-generation Claude model. |
| `PUBLIC_ORIGIN` | Externally-reachable base URL the browser uses to reach this app. |
| `SERVER_PORT`, `CLIENT_PORT` | Express API port (never exposed directly — Vite dev server proxies to it) and Vite dev server port. |
| `TLS_CERT_PATH`, `TLS_KEY_PATH` | Dev-only HTTPS for the Vite dev server (needed for OAuth redirect testing off a Tailscale hostname). |
| `VITE_DEV_ALLOWED_HOST` | Dev-only Vite Host-header allowlist — see `client/viteAllowedHosts.ts`, and [`open-items.md`](open-items.md) for its history. |

---

## 9. Deployment mechanics

Full detail already lives in each repo's own `CLAUDE.md`/`DEPLOY.md`/`README.md` and in
[`environment-urls.md`](../architecture/environment-urls.md). Summary:

- **Branch convention:** push to `staging` → Render staging deploy; push to `production` →
  Render production deploy, **automatically, on push, no approval gate**. Every repo's own docs
  warn about this explicitly — it's a real footgun.
- **CI:** GitHub Actions (`.github/workflows/create-release.yml`) handles the release workflow;
  Render's own build pipeline (triggered by the branch push, not a separate Cloud Build step —
  that was the dead GCP path) does the actual build+deploy.
- **Local dev:** Docker Compose per repo. `number-hive-complete` has three different local dev
  configurations documented side by side (full-stack, a personally-hosted shared dev box,
  standalone backend/frontend) — see `environment-urls.md` §2, they use different ports and are not
  interchangeable.
- **Mobile build pipeline** (Firebase CLI, `npm run deploy:staging`/`deploy:prod`):
  **stale.** This described `number-hive-app`, last touched 2026-03-12, explicitly
  unmaintained per `number-hive-complete/CLAUDE.md`. Not carried forward as a live procedure.

---

## 10. What's simply unknown (unresolved placeholders, still unanswered)

Several areas were never documented at all (`?`, "Unknown:") rather than being stale facts to
correct — there's nothing to reconcile them against, only gaps to fill. Listed here so
they're visible, and mirrored as decisions/actions in [`open-items.md`](open-items.md):

- Financial tracking tool: unknown
- Backup/disaster-recovery procedure for MongoDB and Temporal's Postgres — automation, retention,
  storage location, restore procedure: all unanswered
- Security policy — secret storage strategy (Pulumi secrets? GCP Secret Manager? Render's own
  env var store — most likely answer today, but not confirmed as a deliberate policy), key
  rotation cadence, network restrictions
- Testing strategy — unit/integration/E2E/CI test stage: not documented at the ecosystem level
  (individual repos may have their own test suites; this was never rolled up)
- Incident response — who gets paged, how incidents are tracked, postmortem process
- Data governance — user data storage regions, GDPR compliance posture, data deletion workflows
- CI for iOS/Android, TestFlight, Play Console release tracks — moot while mobile is unmaintained,
  but worth an explicit decision (see §4 above) rather than silent drift

---

*Compiled 2026-08-28, cross-checked against `number-hive-complete`, `number-hive-newvis`, and
`number-hive-admin`'s live `package.json`/`.env.example`/`render.yaml` files, and against
[`environment-urls.md`](../architecture/environment-urls.md) (compiled 2026-07-27 by the same
method). Original source: James's Google Doc infrastructure inventory, passwords removed before
being shared for this reconciliation.*
