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
delegated (James → account holder "Chris", authority passed via "Fletch"). **This should become
an explicit, documented arrangement as part of handover** — see [`open-items.md`](open-items.md).

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
production one. **Confirm the real production Mongo connection is a dedicated cluster with its own
credentials, not this demo one, before handover completes.** See [`open-items.md`](open-items.md).

---

## 4. Third-party services (confirmed live via `package.json`/`.env.example` in `number-hive-complete`)

| Service | Role | Evidence |
|---|---|---|
| **Stripe** | Subscriptions/billing | `stripe ^13.4.0` dependency; `stripe_api_key`/`stripe_api_secret`/`stripe_webhook_secret`/`stripe_account_id` in `backend/.env.example`; live keys confirmed present in `backend/.env.prod` (per `environment-urls.md`'s citation) |
| **SendGrid** | Transactional email (production) | `sendgrid_api_key`; `src/utils/sendgrid.ts` |
| **Mailchimp** | Bulk email to subscribers | `mailchimp_api_key`/`mailchimp_dc`/`mailchimp_list_id`; required for prod, optional in dev |
| **Firebase** | `firebase ^10.5.0` in frontend; `firebase_project_id`/`firebase_client_email`/`firebase_base64_private_key` in backend (Admin SDK) | Historically used for hosting (superseded by Render for the education app — see below), Analytics, Cloud Messaging |
| **Mixpanel** | Server-side event analytics | `mixpanel ^0.18.0`; `mixpanel_api_key`; `src/services/analytics.ts` |
| **Nodemailer** | Local/fallback SMTP | Dev-only fallback — not independently re-verified this pass |

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
| MongoDB (`nh-demo` user — flagged for rotation) | Looks like a demo/shared credential, not confirmed as the real production one | `[see Google Doc: MongoDB]` — recommend rotating to a dedicated production credential regardless, as part of handover, not just as a security-hygiene item |
| Mailchimp API key (`NH-demo` key — same pattern) | Same "demo"-named pattern as the Mongo credential above | Held in `backend/.env.prod`, not reproduced here |
| SendGrid API key | Transactional email | Held in `backend/.env.prod` |
| Render dashboard | All live hosting/deploy for `complete`, `newvis`, `admin` | Needs explicit access grant to David — this is now the single most operationally important credential in the whole system |
| GCP console | GKE/Pulumi infra (status unconfirmed), Google Workspace-adjacent billing | Needed to resolve [`open-items.md`](open-items.md) #1 |
| GitHub org (NumberHive) | All four repos | Standard org access, not separately tracked here |

**Access management matrix (GCP / GitHub / Pulumi state bucket / MongoDB / Stripe / Firebase)**
has never been filled in. Genuinely unknown who
currently holds what beyond James. **Recommend this gets built fresh as part of handover**
rather than inherited, since there was nothing in it to reconcile against.

---

## 7. Deployment mechanics

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

## 8. What's simply unknown (unresolved placeholders, still unanswered)

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
