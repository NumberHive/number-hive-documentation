# Open Items — NumberHive Handover

**Purpose:** everything surfaced while reconciling [`tech-inventory.md`](tech-inventory.md) and [`README.md`](README.md) against
the live repos that needs a decision, a verification, or an action — ranked by how much it'd
hurt if left undone, not by how it happened to come up. Some of these are pre-existing gaps in
the original inventory (backups, security policy, incident response were never documented);
some are things this reconciliation pass found (a cloud cluster that may still be running and
billing with nothing pointing at it; two zombie repos).

Each item states what's known, what's not, and a concrete next step — not just "investigate".

---

## Critical — live cost or live exposure, unconfirmed either way

### 1. Is the GCP/GKE/Pulumi infrastructure actually torn down?

**What's known:** the Kubernetes/Pulumi backend architecture (VPC, private GKE cluster, 3
autoscaling node pools, Cloud NAT, static IP, Cloud Build triggers) was confirmed abandoned by
James on 2026-07-27 in favour of Render. The `iac/` Pulumi config still exists untouched in
`number-hive-complete/backend/iac/`.

**What's not known:** whether the actual GCP resources — the cluster, node pools, static IP,
Cloud NAT — were ever deleted, or just stopped being deployed *to*. "Not in the deploy
pipeline" and "doesn't exist" are different facts, and nobody has checked which one is true.

**Why it's critical:** if the cluster is still up, it is simultaneously (a) a live, silently
accruing cost — the old doc's £61/mo GCP figure may understate a full private-cluster bill —
and (b) an unmonitored attack surface nobody is patching or watching.

**Next step:** log into the GCP console, check the `backend-cluster` GKE cluster, its node
pools, and the `gke-ip` static IP directly. If it's running: either intentionally decommission
it, or make an explicit decision to keep it (unlikely, but possible if something
undocumented depends on it) and document why. This is the single highest-value thing to verify
in this whole handover.

**Owner:** David, once he has GCP console access — good first-week task given the potential
live cost and exposure.

---

### 2. Is Firebase Hosting still deployed, and is it still billing?

**What's known:** Firebase Hosting was historically the production frontend
deploy target (`number-hive-staging.web.app`, `play.numberhive.org`). The actual production
frontend today is Render (`numberhive-frontend-production`). The mobile app repo
(`number-hive-app`) that used the Firebase deploy scripts hasn't been touched since
2026-03-12.

**What's not known:** whether the Firebase Hosting *projects* (`number-hive-staging`,
`number-hive`) still exist and are billing, independent of whether anything currently deploys
to them.

**Next step:** check the Firebase console for both projects. If they exist with no active
deploys pointing at them, decommission. Same class of question as #1, smaller likely cost.

**Owner:** same as #1 — bundle into one GCP/Firebase console pass.

---

## High — affects production data integrity or handover completeness

### 3. Staging environment ambiguity — Render staging vs. a personally-hosted `staging-local`

**What's known:** [`environment-urls.md`](../architecture/environment-urls.md) §2 documents two candidate staging environments for
`number-hive-complete` — Render's `numberhive-*-staging` services, and a self-hosted Docker
Compose setup (personally hosted, developer-provided infrastructure) at `staging-local.numberhive.org`. Different docs
call different ones canonical; neither has been confirmed live via a direct check in this
pass.

**Next step:** hit both URLs, confirm which (if either, or both) is actually serving traffic
today, and update [`environment-urls.md`](../architecture/environment-urls.md) with a definitive answer instead of "needs a live
check."

**Owner:** David, early — this affects where he tests things before his first deploy.

### 4. MongoDB production credential — confirm it's not the old demo credential

**What's known:** the production MongoDB connection string on record uses an `nh-demo` user
(`mongodb+srv://nh-demo:...@cluster0...`) — this reads as a shared/demo cluster credential,
not a dedicated production one. Mailchimp's API key follows the identical
"demo"-named pattern.

**What's not known:** whether production actually runs on a differently-named, dedicated
Atlas cluster/credential, or whether `nh-demo` genuinely is what's in `backend/.env.prod`
today.

**Next step:** check `backend/.env.prod`'s actual `mongodb_url` against what's in Atlas. If it
is still `nh-demo` or similarly shared/broad, rotate to a dedicated production credential —
worth doing rather than inheriting a demo-named credential indefinitely.

**Owner:** David, once he has Atlas access — credential hygiene, worth an early look but not
urgent enough to block anything.

### 5. Two zombie repos in the NumberHive GitHub org, not archived

**What's known:** `number-hive-backend` and `number-hive-app` were superseded by the
`number-hive-complete` monorepo merge on 2026-03-14/16 and haven't been pushed to since.
Neither is marked `archived` on GitHub.

**Why it matters:** a new CTO scanning the org's repo list has no signal that these are
dead — he could waste time investigating them, or worse, someone could accidentally deploy
from a fossil branch believing it's current.

**Next step:** archive both on GitHub (Settings → Archive repository). This is reversible,
low-risk, and purely a signal-to-future-readers fix — no code or history changes.

**Owner:** David, once he has GitHub org access — trivial, low-risk, whenever it's convenient.

### 6. Access management matrix — has never existed

**What's known:** there has never been a completed access matrix for GCP / GitHub / Pulumi
state bucket / MongoDB / Stripe / Firebase access. Access has always been tracked informally
and held by James — that's simply the current state, not a gap James introduced for this
handover.

**What handover actually requires:** each of these — GCP console, GitHub org, MongoDB Atlas,
Stripe, Firebase — granted to David directly. That's the mechanical part of the handover
itself (and a precondition for David verifying #1–#4 above), not a separate deliverable.

**Next step (optional, David's call):** turning that into a standing, documented access matrix
is a process improvement David may want once he's settled in — not something that needs to
exist before he can start.

**Owner:** David, if/when he decides it's worth building.

---

## Medium — should happen but doesn't block day-to-day operation

### 7. DNS/GoDaddy access — informal delegation, doesn't yet name David

**What's known:** DNS is currently informally delegated (James → account holder "Chris",
authority passed via "Fletch"). No documented, explicit arrangement exists — that's simply how
it's always worked, not something that changed for this handover.

**Next step:** introduce David into that existing chain — added to the GoDaddy account, or a
documented arrangement agreed with Chris — since the current one predates him and doesn't name
him at all.

**Owner:** James — this is the introduction itself, part of hand-off, not separate cleanup
work.

### 8. Cost map needs a fresh pull

**What's known:** the previously recorded cost figures (GCP £61/mo, Firebase £135/mo, MongoDB Atlas
$161/mo) predate the Render migration and may include dead spend from #1/#2. Render itself —
now the actual hosting bill for all three live services — was never previously tracked.

**Next step:** pull current billing from GCP, Render, Google Workspace, and MongoDB Atlas
consoles once #1/#2 are resolved, so the new total reflects reality rather than a mix of
stale and missing figures.

**Owner:** David, once console access (#6) is granted — good first-week task, doubles as a
forcing function to actually get into each console.

### 9. Security policy — never documented

**What's known:** security policy has never been documented — open questions include where
secrets are stored (Pulumi secrets? GCP Secret Manager? JWT key storage?), network
restrictions, and key rotation cadence. Best current guess: Render's own environment variable
store is the de facto secret storage today, but this has never been stated as a deliberate
policy.

**Next step:** write down the actual current practice (even if it's just "Render env vars,
no formal rotation schedule") as a baseline, then decide if that's sufficient or needs
tightening.

**Owner:** David, as part of establishing his own operational baseline.

### 10. Backups & disaster recovery — undocumented for both databases

**What's known:** nothing — whether MongoDB backups are automated, whether Temporal's Postgres
is backed up, retention policy, and restore procedure have never been documented.

**Next step:** confirm Atlas's backup configuration (it may have sane defaults enabled
already — Atlas typically does — but this needs confirming, not assuming) and Render managed
Postgres's backup/retention settings for the Temporal and admin databases. Document a restore
procedure, or at minimum confirm one exists and where.

**Owner:** David — this is a "find out before you need it" item, not urgent today but
dangerous to discover during an actual incident.

### 11. Testing strategy — never rolled up at the ecosystem level

**What's known:** individual repos may have their own test suites; there's no ecosystem-wide
statement of unit/integration/E2E coverage or a CI test gate.

**Next step:** not blocking, but worth a paragraph per repo in each repo's own README once
David has looked at what's actually there.

**Owner:** David, low priority, ongoing.

### 12. Incident response — undefined

**What's known:** nothing — who gets paged, how incidents are tracked, postmortem process
are all unanswered and not addressed by this reconciliation either.

**Next step:** worth a short doc once David is settled — doesn't need to be elaborate for a
team this size, but "what do I do when something breaks at 2am" shouldn't be a blank page.

**Owner:** David, first month.

---

## New this pass — surfaced while extending `tech-inventory.md` for the Dave handover session

Six items were specifically commissioned to check against code. Three confirm as originally
framed; three turned out to need correcting once checked against the actual repos — reported
here as the code actually shows, not as originally assumed, per this handover's citation rule.

### 15. `EventTypePickerModal` sort feature — confirmed develop-only, not on `main`

**What's known:** `number-hive-admin/client/src/components/EventTypePickerModal.tsx` exists on
both `develop` and `main` (added `main`-side via CHG-4139, commit `c7b2381`, wired into
`EventsPage` via `6a97ab8`). Two later commits — `50b8eef` ("Sort event-type picker options
alphabetically") and `bca35c7` ("Fix: sort a copy of options rather than mutating the prop") —
add alphabetical sorting on top of that. `git branch --contains` confirms both sort commits are
on `develop` only, not on `main`.

**Why it matters:** low-stakes on its own (a UI sort order), but it's a live example of
`develop` and `main` having actually diverged in `number-hive-admin` — worth knowing before
assuming the two branches are in lockstep.

**Next step:** none required — this is a documentation note, not a defect. If/when
`develop` next promotes to `main` (see [`promotion-runbook.md`](promotion-runbook.md) §3), the
sort behaviour ships as part of that normal promotion.

**Owner:** David, informational only.

### 16. `viteAllowedHosts` in `number-hive-admin` — confirmed wired in and working, **not** unwired

**What's known:** `client/viteAllowedHosts.ts` is actively imported and used in
`client/vite.config.ts` (lines 7 and 40), has its own test file
(`client/src/viteAllowedHosts.test.ts` equivalent), and is documented in both the repo's own
README and `.env.example` (`VITE_DEV_ALLOWED_HOST`, see
[`tech-inventory.md`](tech-inventory.md) §8). It was, once, silently dropped by an
auto-integration — commit `660ddb4`, "Restore Vite dev-server Host allowlist for ripper (lost
in CHG-3616 integration)" — and deliberately restored afterwards.

**Why it's listed here at all:** an earlier version of this handover's working notes described
this helper as "committed but unwired." That was checked directly against the code this pass
and is **not correct** — it's wired in, tested, and documented. The genuinely notable fact is
the *history*: it was dropped once by an automated integration and had to be manually restored,
which says something about that integration process rather than about this file's current
state.

**Next step:** none required for the file itself. Worth a general note (not a specific action)
that auto-integrations in this repo have, at least once, silently dropped working code —
something to watch for, not something to fix here.

**Owner:** David, informational only.

### 17. CHG-1577 Stripe admin-config links — confirmed shipped and live, **not** archived/part-built

**What's known:** `feat(CHG-1577): Add "Open in Stripe" links in admin panel` (`94c9565`) and
`fix(CHG-1577): fix config import in stripe-config resolver and test` (`7ad3578`), both dated
2026-06-15, are merged into `number-hive-complete`'s `main` and live today as
`backend/src/graphql/admin/stripe-config/`. The only remnant of the original working branch is
a stale git worktree, `worktree-chg-1577-stripe-links`, which has not diverged from `main` and
contains no unique work.

**Why it's listed here at all:** an earlier version of this handover's working notes described
this as "archived, part-built Stripe configuration work." Checked directly against the code
this pass — it shipped in full and is live; nothing about it is archived or incomplete.

**Next step:** delete the stale `worktree-chg-1577-stripe-links` worktree — it's safe (no
unique commits) and its continued presence could otherwise mislead a reader into thinking
there's unfinished work here.

**Owner:** David, trivial cleanup whenever convenient.

### 18. Constraint-drop script (CHG-2285) — confirm it has actually completed against production

**What's known:** `backend/src/graphql/admin/subscription-sync/subscription-index-migration.ts`
(design doc: `docs/superpowers/specs/2026-07-04-subscription-index-migration-design.md`) is a
self-healing script that drops and recreates the `stripeSubscriptionId_1` MongoDB index as
`unique+sparse`, guarding against duplicate values before doing so. It runs automatically as a
side effect of a real (non-dry-run) "Run Sync" click in the admin panel's Stripe-sync tooling.
There is no rollback path once it has run.

**What's not known:** whether this has actually completed successfully against the production
MongoDB cluster since it shipped (2026-07-04). No execution/verification log was found in this
repo confirming a production run.

**Why it matters:** if it hasn't run yet, the index may still be in whatever state predates
this migration, which matters for anyone relying on `stripeSubscriptionId` uniqueness at the DB
level. If it has run and something went wrong partway, there's no built-in undo.

**Next step:** unverified — confirm with James whether this has been run against production,
and if so, when and with what result.

**Owner:** David, in the handover session — this is exactly the kind of thing the session
exists to close out.

### 19. Decimal128/Stripe migration (CHG-2400–2467) — fused into production data; confirm the live backfill actually ran

**What's known:** this migration converted Stripe monetary fields (across `subscriptions`,
`subscriptioninvoices`, `subscriptioncharges`, `stripecoupons`) to Decimal128 `_USD`-suffixed
fields, **deleting the old field names from the schemas entirely** — this is not a purely
additive change. The runbook
(`docs/superpowers/plans/2026-07-15-stripe-usd-rollout-runbook.md`) is explicit that this is a
one-way code change: §6 ("Rollback") states in so many words that a bare `git revert` of the
migration code **does not revert the data** — the old fields are gone from the schema either
way — and gives the exact per-collection `mongorestore` commands as the actual recovery path,
keyed to a timestamped backup taken before the live run.

**Why restoring older application code against current data is unsafe:** if a rollback is ever
attempted by simply reverting to pre-migration application code without also restoring the
data via `mongorestore`, that older code will be reading for field names
(e.g. `amount`) that no longer exist in the documents — the data was migrated to the new
`_USD` Decimal128 field names in place, not duplicated alongside the old ones. **Any future
rollback of this migration must restore data, not just code.**

**What's not known:** whether the actual production `--live` backfill has run. A
suggestively-timestamped backup folder exists at
`backend/backups/stripe-usd-20260715T100905Z/` but **it is empty** — no archive files inside
it — which is not proof either way (an empty folder is consistent with "the backup step never
actually wrote anything" or "this is a leftover directory from a dry run"), but it is not
evidence of a completed live run either.

**Next step:** unverified — confirm with James whether the production `--live` backfill for
this migration has actually run, and if so, whether the corresponding backup archives exist
somewhere other than this empty folder.

**Owner:** David, in the handover session — highest-stakes item in this new section, since it
touches live billing data with a schema-destructive, one-way change.

### 20. "ANALYSER-in-admin" — confirmed independently built, **not** a port of `number-hive-complete`'s Analyser

**What's known:** `number-hive-complete` had an earlier, Mongo-based Analyser feature (CHG-1003,
started 2026-05-25 — confirming the "May–June" timeframe some working notes referenced).
`number-hive-admin`'s ANALYSER (CHG-4140 onward, started 2026-08-03) is a **separate, Postgres-based
build** — there are zero MongoDB references anywhere in `number-hive-admin` (confirmed by
repo-wide search, and consistent with §3's "MongoDB usage per service" table in
[`tech-inventory.md`](tech-inventory.md), which lists admin as Postgres-only). Admin's own
planning docs describe this as new foundational work, citing the older Mongo-based system's
*scope* only as a sizing reference, not as code being ported.

**Why it's listed here at all:** an earlier version of this handover's working notes described
admin's ANALYSER as "ported" from `number-hive-complete`'s May–June system. Checked directly
against the code this pass — it's an independent reimplementation, not a port; no shared code
exists between the two.

**Next step:** none required — this is a documentation correction, not an action item. Worth
knowing so nobody goes looking for shared code between the two ANALYSER implementations that
isn't there.

**Owner:** David, informational only.

### 21. Fragile areas flagged this pass, not elsewhere covered

- **Stripe webhook double-`case` bug** (`invoice.payment_failed` handled twice in
  `backend/src/utils/stripe.ts`, lines 278 and 339 — the second block is dead code). See
  [`tech-inventory.md`](tech-inventory.md) §4 for the full event table. Not fixed as part of
  this handover (documentation only) — worth a real look at which block was meant to be
  authoritative.
- **`cancelSubscription()` in `backend/src/utils/stripe.ts` has no confirmed non-test caller.**
  Possibly dead code, possibly cancellation happens entirely through the Stripe billing portal
  rather than this app's own code path — unverified, confirm with James.
- **No distributed lock on `number-hive-complete`'s three in-process `node-cron` jobs**
  (digest, financial reconciliation, journey-stage sync — see
  [`tech-inventory.md`](tech-inventory.md) §7). Not an active risk on a single Render instance,
  but would silently double-fire (e.g. double-pulling Stripe invoices) if ever scaled without
  adding a lock first.
- **Temporal's `startScheduledEvent` capability has zero production call sites** found this
  pass — only referenced from test mocks, despite Temporal being live infrastructure for the
  70-second disconnect-timeout use case. Not claimed as dead code (a static grep can miss
  dynamically-started workflows), but worth confirming with James whether anything relies on it.

---

## Low — real gaps, but low urgency or genuinely a business (not technical) decision

### 13. Mobile app store listings — pull or leave stale?

**What's known:** Apple App Store (App ID `1636921061`) and Google Play (`com.numberhive`)
listings likely still exist and may still be live to end users, even though
`number-hive-app` hasn't been built or deployed since 2026-03-12 and is explicitly
unmaintained.

**Next step:** this is a product/business call, not a technical one — pull the listings, or
knowingly leave old builds live. Flagging so it's a decision, not silent drift.

**Owner:** David + business stakeholders, not urgent.

### 14. Data governance / GDPR posture — undocumented

**What's known:** nothing beyond the fact that the education app handles student data under
school consent (noted in `platform-overview.md`). Storage regions, GDPR compliance posture,
and data-deletion workflows remain undocumented.

**Next step:** given the product handles student data, this is worth a proper compliance
review at some point post-handover — not something this technical reconciliation can answer.

**Owner:** David + legal/compliance, medium-term.

---

*Compiled 2026-08-28 alongside [`README.md`](README.md) and [`tech-inventory.md`](tech-inventory.md), as part of the CTO
handover package. Re-prioritize freely — the ranking here is a starting judgement, not a
fixed sequence. Items 15–21 added 2026-08-31 while extending `tech-inventory.md` ahead of the
Dave handover session — each checked directly against the live repos rather than transcribed
from working notes; three (16, 17, 20) corrected an earlier assumption once the code was
actually read.*
