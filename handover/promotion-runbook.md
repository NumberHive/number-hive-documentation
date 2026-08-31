> **Pending manual verification. To be walked live in the handover session.**
> Everything below is derived from committed repo config (`render.yaml`, workflows, compose files,
> deploy scripts, and each repo's own operational docs) as of 2026-08-31. **Render dashboard settings
> (branch wiring, auto-deploy on/off, plan, whether a Blueprint has even been activated, and whether
> `sync: false` secrets are actually filled in) can differ from committed config and cannot be
> confirmed by reading a repo alone.** §6 below lists, per repo, exactly which steps most need a live
> check in the Render dashboard/GitHub settings during the session — start there.

# Promotion Runbook

Four repos, four different actual pipelines — they are **not** uniform, and this document does not
pretend otherwise. Read the per-repo section for the repo you're touching; do not assume one repo's
pattern applies to another.

---

## 1. `number-hive-complete` (Number Hive Education — paid platform)

**Branches: `main`, `staging`, `production`.**

| Branch | What it's for |
|---|---|
| `main` | Integration branch. PRs merge here. "Safe to push to" (`CLAUDE.md:114`). |
| `staging` | Pre-release verification. Updated from `main` by direct merge, **not** cherry-pick. |
| `production` | Live. Updated **only** via isolated `hotfix/*` branches cherry-picked from `main` — never by merging `staging` wholesale (`docs/operations.md:96`: *"Production is updated via isolated hotfix branches cherry-picked from `main` — not by merging `staging` wholesale. This lets us ship individual, verified changes incrementally rather than in a big batch."*). `CLAUDE.md:113`: **"NEVER push to `production` or merge anything into `production` without an explicit instruction to do so."** |

**Promote `main` → `staging`:**
```bash
git checkout staging
git merge origin/main
git push origin staging
ssh james@onyx '/home/james/numberhive-staging-local/number-hive-complete/deploy-staging-local.sh'
```
This is a **self-hosted Docker Compose deploy**, not Render — see "What triggers the deploy" below.

**Ship a fix to `production` (the only supported path):**
```bash
git checkout -b hotfix/<short-name> main
git cherry-pick <commit-sha>          # the specific verified fix commit(s)
git push origin hotfix/<short-name>:production
```
Full 8-step checklist (including a **pre-written rollback command per ship**) lives in
`number-hive-complete/docs/SHIPPING.md` — this runbook does not repeat that checklist, only the
promotion mechanics.

**What triggers each deploy:**
- **`production`** — Render, `render.yaml:203-211` (`numberhive-backend-production`, Docker, plan
  `standard`) and `render.yaml:163-169` (`numberhive-frontend-production`, static site), both tracking
  branch `production`. No `autoDeploy` key is set in the file for any service, so Render's default
  (deploy on push) applies unless changed by hand in the dashboard — **unverified, check live.** The
  backend's own `Dockerfile` **runs `npm run test:unit` as a build step** (`backend/Dockerfile:23`,
  comment: *"if tests fail, the Docker build fails and the deploy is rejected"*) — this is the only
  test gate in the pipeline; there is no separate GitHub Actions CI step.
- **`staging`** — **NOT Render.** `render.yaml`'s staging service blocks (`numberhive-frontend-staging`,
  `numberhive-backend-staging`, `numberhive-temporal-staging`, lines 8-121) describe Render services
  that `STAGING-LOCAL.md:344-356` records were **manually decommissioned on 2026-05-06** ("pushes to
  `staging` no longer trigger any Render build hook") and replaced with the self-hosted `onyx` box
  above. **The committed `render.yaml` is one day stale relative to that decommission and was never
  edited to remove the staging blocks — treat those blocks as dead, pending live confirmation that the
  services are actually gone from the dashboard.**
- Only one GitHub Actions workflow exists in the repo, `backend/.github/workflows/create-release.yml`
  — triggers on branches named `build/x.y.z`, opens a PR into `staging`. No `build/x.y.z` branch has
  ever existed in this repo's history; **this workflow is vestigial and does not fire in practice.**
  It does not call Render.

**How to verify:**
- Production: `docs/SHIPPING.md`'s ship log records a specific smoke-test per ship (health endpoint,
  key GraphQL query) — follow the pattern of the most recent entry.
- Staging (onyx): `deploy-staging-local.sh` itself runs a GraphQL smoke check after `docker compose up`
  (script, final lines).

**Rollback:** `docs/SHIPPING.md:132-146` — standard rollback is
`git revert <hotfix-sha> --no-edit && git push origin HEAD:production`, target window **under 5
minutes**. Explicit caveat: *"If the hotfix commit introduced a DB migration that has already run, the
rollback reverts the code but NOT the data... decide separately whether to roll back data"* — there is
no automated down-migration. No staging rollback procedure is documented beyond re-running the deploy
script against an earlier commit by hand.

**Gotchas:**
- `backend/DEPLOYING.md`, `frontend/DEPLOYING.md`, `frontend/DEPLOYMENT.md`, `backend/cloudbuild.yaml`,
  `backend/.gitlab-ci.yml`, `backend/iac/**` all describe a **defunct Google Cloud Build / GKE /
  Firebase Hosting pipeline** from this repo's initial commit, superseded the next day by the switch to
  Render. **Do not follow these docs** — they describe infrastructure that (per the current, actively
  maintained docs) is not in use.
- `backend/render.yaml` is a separate, stale duplicate with only one commit ever (the initial monorepo
  setup) and old custom-domain URLs. Render Blueprints read `render.yaml` from the repo root by
  default; this file is very likely unused, but that can only be confirmed live in the dashboard.
- Migrations (`backend/src/database/migrations/`) run **automatically, concurrently, on every boot**
  (`backend/src/database/connect.ts:8-15`, `migrations/index.ts`), with no version ledger — each
  migration function is individually written to be idempotent/self-guarding rather than tracked. This
  matters for rollback: an older code version's boot will re-run its own (older) migration set, which
  does not undo anything a newer version already wrote.
- At last check, `staging` and `production` branch heads trailed `main` by 9 days — worth asking live
  whether promotion has continued on its normal cadence recently.

---

## 2. `number-hive-newvis` (Number Hive Arcade — free game, pre-launch)

**Branches: `develop`, `main`, `staging`.**

The user-facing branch order above lists `main` before `staging`, but **the actual promotion sequence,
evidenced both by the repo's own doc and by git history itself, is `develop` → `staging` → `main`**,
with `main` last (most release-ready, production):

> `DEPLOY.md:24`: *"This repo's release pipeline promotes commits `develop` → `staging` → `main` (see
> the 'Release promotion: X → staging/main' commits in `git log`)."*

`git log --oneline --all --grep="Release promotion"` shows real, dated merge commits confirming this —
e.g. `31b7470 Release promotion: 06138ca → staging`, `7f9e0b8 Release promotion: 50e775a → main`.

| Branch | What it's for |
|---|---|
| `develop` | Working/integration branch. |
| `staging` | Pre-release verification. Backs **two** environments: the personally-hosted Docker Compose staging deployment, and Render's `staging-game.numberhive.app`. |
| `main` | Most release-ready branch. Backs Render production, `game.numberhive.app`. |

**Promote `develop` → `staging` → `main`:**
```bash
git checkout staging
git merge origin/develop
git push origin staging       # -> Render staging auto-deploys; also update the personally-hosted box:
./deploy.sh staging -y        # run from/against the personally-hosted host

git checkout main
git merge origin/staging
git push origin main          # -> Render production auto-deploys
```
`DEPLOY.md:29` warns explicitly: *"Deploying `main` to a staging environment skips the verification
step the `staging` branch is meant to provide, since `main` sits *later* in the pipeline than
`staging`."* — don't shortcut past `staging`.

**What triggers each deploy:**
- **Render** (`render.yaml`, 156 lines) — four services: `numberhive-staging-frontend` /
  `numberhive-staging-backend` track branch `staging` (lines 73, 92); `numberhive-game-production-frontend`
  / `numberhive-game-production-backend` track branch `main` (lines 119, 137). No `autoDeploy` key set
  for any service — Render default applies, **unverified live.** Backend is Docker
  (`Dockerfile.backend`, `node:20-alpine`, `CMD ["node", "src/index.js"]`); frontend is a Render static
  site (`buildCommand: npm ci && npm run build`), not Docker. **No test-in-build gate** — unlike
  `number-hive-complete`'s backend Dockerfile, this one does not run tests during the Docker build.
- **No GitHub Actions exist in this repo at all** — confirmed, no `.github/workflows`. All promotion
  and deployment is manual git + Render's own branch-tracking, or the manual `deploy.sh` script for the
  personally-hosted path.
- **Personally-hosted staging** — `deploy.sh <branch> [-y]` (root): `git fetch && git checkout $BRANCH
  && git pull`, then `docker compose -f docker-compose.prod.yml -p numberhive build && ... up -d`.
  Defaults to `main` but warns/prompts if the target branch differs from `$EXPECTED_BRANCH` (default
  `staging`) — a deliberate guardrail against deploying the wrong branch to this host.
- Secrets for both Render environments live in two Render Environment Groups
  (`numberhive-staging-render`, `numberhive-production-render`), referenced by `fromGroup`, not defined
  in the file.

**How to verify:** `DEPLOY.md:49-64` records a launch-verification smoke-test table (last run
2026-07-26) covering `/health`, `/v1/visitors`, `/v1/auth/google`, `/v1/events`, `/v1/challenges` all
confirmed live in production, and confirms `ensureIndexes()` bootstrapped the database correctly on
that real deploy. Re-run the same checks after any new production promotion.

**Rollback:** **No rollback script or documented procedure exists** for either environment — this is a
gap relative to `number-hive-complete`. Baseline: Render dashboard's own "redeploy previous" for the
Render environments; for the personally-hosted box, re-run `./deploy.sh <branch>` against an earlier
commit/tag manually. `backend/src/db/indexes.js`'s `ensureIndexes()` runs unconditionally on every boot
and is additive-only/idempotent (`indexes.js:4-11`: *"createIndex is a no-op when the index already
exists with the same key + options, so this is safe to call on every startup"*) — an older version
rolling back does not drop indexes a newer version added, but nothing in the code guards against a
newer version's `unique` constraint having already let in data an older version's schema wouldn't
police. Treat as an open risk, not a solved one.

**Gotchas:**
- The production Render service names in the dashboard were changed by hand after creation
  (`numberhive-production-frontend/backend` → `numberhive-game-production-frontend/backend`);
  `render.yaml` was edited 2026-07-26 to match — a self-documented instance of dashboard-vs-file drift.
  Verify the dashboard names still match the file.
- `docs/architecture.md` §1.1 is **stale** — it records production DNS as unresolved (`NXDOMAIN`) as of
  a 2026-07-12 probe, which is contradicted by `DEPLOY.md`'s 2026-07-26 confirmation that production is
  live and smoke-tested. The file self-flags this as a point-in-time snapshot, but don't read §1.1
  alone as current evidence of production's status.
- `backend/.env` must exist on the personally-hosted host before `docker compose up` / `deploy.sh` will
  succeed — the backend fails fast on missing required env vars.
- The "frontend Node 18 Docker image" security item named in the CEO's follow-up email is **not** in
  this repo — both `Dockerfile.backend` and `Dockerfile.frontend` here already use `node:20-alpine`.
  That finding belongs to `number-hive-complete/frontend/Dockerfile` (see §5, Priority 5 extension of
  `open-items.md`, and `handover/security-remediation-status.md`). Do not treat it as a newvis gotcha.

---

## 3. `number-hive-admin` (internal ops/billing tool, in development)

**Branches: `develop`, `main`.**

| Branch | What it's for |
|---|---|
| `develop` | Working branch. |
| `main` | GitHub's configured default branch. Promoted to via the same "Release promotion" merge-commit convention seen in the other repos (`git log develop..main` shows repeated two-parent merges titled `Release promotion: <sha> → main`, e.g. `c1d849d`, `Merge: 8ebbb40 84a96ea`). This convention is **not documented in a written runbook inside the repo** — it's a platform-level practice, observable only in git history. |

**Promote `develop` → `main`:**
```bash
git checkout main
git merge origin/develop
git push origin main
```

**What triggers the deploy — this is the item most needing live verification in this repo.**
`render.yaml`'s own header comment is explicit and should be read in full before assuming anything is
live:

> `render.yaml:12-33`: **"ACTIVATION IS MANUAL — TODO(infra owner): committing this file does NOT
> itself create or update anything on Render."** A human must go to the Render dashboard, "New +" →
> "Blueprint", point at this repo, then fill in six `sync: false` secrets (`GOOGLE_CLIENT_ID`,
> `GOOGLE_CLIENT_SECRET`, `PUBLIC_ORIGIN`, `ENTITLEMENT_PUSH_SECRET`, `NEWVIS_EVENTS_PUSH_SECRET`,
> `OPENROUTER_API_KEY`), and add the OAuth redirect URI in Google Cloud Console. **"Until all of the
> above is done, none of what this file describes is actually live."**

No `branch:` key and no `autoDeploy:` key appear anywhere in `render.yaml` — which branch Render would
even deploy from, once/if activated, is not committed config at all. GitHub's default branch is `main`,
which is the likely candidate, but this is inference, not a stated fact — **confirm live whether the
Blueprint has been connected at all, and if so, which branch it tracks.**

Defined services, if/when activated: a `web` service (`render.yaml:65-124`, `npm install && npm run
build`, `preDeployCommand: npm run migrate --workspace server`, `npm start`) and a daily cron job
`audit-log-purge` (`render.yaml:49-64`, schedule `0 3 * * *` UTC). Managed Postgres only — **no
ClickHouse Render service is defined**, despite ClickHouse being part of this repo's stack per the
tech inventory; confirm live whether ClickHouse is hosted elsewhere or not yet provisioned.

No GitHub Actions workflows exist in this repo. No Docker/compose files exist — local dev is plain
`npm run dev`.

**How to verify:** `README.md:199-211` — a post-deploy smoke-test checklist: sign-in, Home page, Audit
Log page, Events page.

**Rollback:** None documented beyond the migration-blocks-cutover safety net (below). No down-migration
scripts exist.

**Migrations:** Plain numbered `.sql` files under `server/src/db/migrations/`, no framework
(no Knex/Prisma/node-pg-migrate). Run via `preDeployCommand: npm run migrate --workspace server`
(`render.yaml:75`) — **automatically before every deploy, blocking cutover on failure**: *"Migrations
run automatically before every deploy via `preDeployCommand`... If a migration fails, Render blocks the
cutover — the previous instance keeps serving traffic, so a failed migration is a safe no-op deploy,
not a partial-upgrade state"* (`README.md:193-197`). They are deliberately **not** run on server boot —
`server/src/db/assertSchemaCurrent.ts` instead fails the server fast at startup if any migration file on
disk hasn't been applied, naming the specific unapplied files.

**Gotchas:**
- The generic "staging → production, auto-deploy on push" branch convention described in this
  documentation repo's own `handover/tech-inventory.md` and `handover/README.md` (sourced from
  `number-hive-complete/CLAUDE.md`) **does not map onto this repo** — admin has no `staging` or
  `production` branches at all, only `develop`/`main`. Don't assume that generic description applies
  here without checking.
- Whether the Blueprint has ever actually been connected/activated is the single biggest open question
  for this repo's deploy story — everything else in this section is moot until that's confirmed.

---

## 4. `number-hive-documentation` (this repo)

**Branches: `develop`, `main`. No deploy target of any kind.**

Confirmed directly: no `render.yaml`, no `.github/workflows`, no Dockerfile/compose file, no
`scripts/` directory anywhere in this repo. `package.json`'s own `test` script literally echoes
*"number-hive-documentation is a docs-only repo with no automated test suite — nothing to run"* and
exits 0.

The same "Release promotion: `<sha>` → main" merge-commit convention appears in this repo's history too
(one such commit, `da7c2e7`) — used purely as a documentation-versioning/integration practice, not
because anything deploys.

```bash
git checkout main
git merge origin/develop
git push origin main
```

No triggers, no verification steps, no rollback procedure, no gotchas beyond ordinary git hygiene —
there is nothing here to deploy or roll back.

---

## 5. Cross-repo note: the "generic branch convention" in this repo's own docs is not universal

`handover/README.md` and `handover/tech-inventory.md` (pre-existing, extended under Priority 5)
describe a `staging`/`production`, auto-deploy-on-push convention drawn from
`number-hive-complete/CLAUDE.md`. That convention accurately describes `number-hive-complete`'s
*production* environment, roughly describes `number-hive-newvis` (both have `staging` and a
production-tracking branch, though the branch named `main` is newvis's production branch, not a branch
named `production`), and **does not describe `number-hive-admin` or `number-hive-documentation` at
all** (`develop`/`main` only, and in admin's case, deploy activation is manual and unconfirmed even to
exist yet). Do not extrapolate one repo's pattern onto another.

---

## 6. What most needs live verification in the session — priority order

1. **`number-hive-admin`: has the Render Blueprint ever actually been connected/activated?** Nothing
   in this repo's section of this runbook can be trusted as "live behaviour" until this is answered —
   currently it is genuinely unknown from the repo alone whether pushing to `main` deploys anything at
   all.
2. **`number-hive-complete`: do the Render staging services in `render.yaml` (lines 8-121) still exist
   in the dashboard?** `STAGING-LOCAL.md` records their decommission on 2026-05-06, one day after the
   file's last edit; if they're gone, that section of `render.yaml` should be deleted, not just ignored.
3. **`number-hive-complete` and `number-hive-newvis`: confirm actual `autoDeploy` setting and branch
   wiring for every Render service** — neither `render.yaml` sets the key explicitly in either repo, so
   both are 100% dependent on dashboard configuration/history that isn't visible from the repo.
4. **`number-hive-complete`: staging/production branch heads were 9 days behind `main`** at last check
   — confirm whether promotion has continued on its normal cadence or has stalled.
5. **`number-hive-newvis`: confirm the production Render service names in the dashboard** still match
   the hand-edited names in `render.yaml` (`numberhive-game-production-*`).
6. **`number-hive-admin`: confirm whether ClickHouse is hosted anywhere** — no Render service for it
   exists in `render.yaml`, and the tech inventory does not resolve this either.
7. **GitHub branch protection rules** (required reviews, required status checks) on `main` in any of
   the four repos — none of this is visible from repo contents; check live via the GitHub UI or
   `gh api repos/<org>/<repo>/branches/<branch>/protection`.

---

*Sources cited: `number-hive-complete/render.yaml`, `backend/render.yaml`, `backend/Dockerfile`,
`docker-compose*.yml`, `deploy-staging-local.sh`, `CLAUDE.md`, `docs/operations.md`,
`docs/SHIPPING.md`, `STAGING-LOCAL.md`, `backend/.github/workflows/create-release.yml`,
`backend/src/database/migrations/*`; `number-hive-newvis/render.yaml`, `Dockerfile.backend`,
`Dockerfile.frontend`, `docker-compose.prod.yml`, `deploy.sh`, `DEPLOY.md`, `docs/architecture.md`,
`backend/src/db/indexes.js`, `backend/src/index.js`; `number-hive-admin/render.yaml`, `README.md`,
`server/src/db/migrate.ts`, `server/src/db/assertSchemaCurrent.ts`,
`server/src/db/migrations/*.sql`; `number-hive-documentation/package.json`; git history (`git log
--all --grep="Release promotion"`) in all four repos. Compiled 2026-08-31.*
