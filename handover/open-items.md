# Open Items — NumberHive Handover

**Purpose:** everything surfaced while reconciling [`tech-inventory.md`](tech-inventory.md) and [`README.md`](README.md) against
the live repos that needs a decision, a verification, or an action — ranked by how much it'd
hurt if left undone, not by how it happened to come up. Some of these are pre-existing gaps in
the original inventory (backups, security policy, incident response were never documented);
some are things this reconciliation pass found (a cloud cluster that may still be running and
billing with nothing pointing at it; two zombie repos; a security exposure unrelated to the
deed work).

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

**Owner:** whoever gets GCP console access first — David or James, before handover completes.

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

**Next step:** check `backend/.env.prod`'s actual `mongodb_url` against what's in Atlas.
If it is still `nh-demo` or similarly shared/broad, rotate to a dedicated production
credential as part of handover, not as an afterthought — this is a credential a departing
CTO's laptop/history/Slack may have seen copy-pasted, exactly the kind of thing a handover
should force a rotation of regardless of the deed/security items already in flight.

**Owner:** James, before handover completes (this is a credential-hygiene action alongside
the PAT/`.env.tmp` items already underway — see the note at the bottom of
this doc).

### 5. Two zombie repos in the NumberHive GitHub org, not archived

**What's known:** `number-hive-backend` and `number-hive-app` were superseded by the
`number-hive-complete` monorepo merge on 2026-03-14/16 and haven't been pushed to since.
Neither is marked `archived` on GitHub.

**Why it matters:** a new CTO scanning the org's repo list has no signal that these are
dead — he could waste time investigating them, or worse, someone could accidentally deploy
from a fossil branch believing it's current.

**Next step:** archive both on GitHub (Settings → Archive repository). This is reversible,
low-risk, and purely a signal-to-future-readers fix — no code or history changes.

**Owner:** James or David, either can do this in under a minute each.

### 6. Access management matrix — genuinely never filled in

**What's known:** there has never been a completed access matrix for GCP / GitHub / Pulumi state bucket /
MongoDB / Stripe / Firebase access. There is currently no
record of who holds what beyond James.

**Next step:** build this fresh as part of handover rather than trying to reconstruct
history — for each system, list who currently has access, and what David needs granted.
This is foundational to the handover actually completing (David can't verify #1–#4 above
without console access to begin with).

**Owner:** James, as the only person who currently knows the answer.

---

## Medium — should happen but doesn't block day-to-day operation

### 7. DNS/GoDaddy access — make the informal delegation explicit

**What's known:** DNS is currently informally delegated (James → account holder "Chris",
authority passed via "Fletch"). No documented, explicit arrangement exists.

**Next step:** get David added to the GoDaddy account (or a documented, explicit delegation
arrangement agreed with Chris) rather than relying on an informal chain that predates this
handover and doesn't name David at all.

**Owner:** James, to broker the introduction/access grant.

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

## Already in motion — not new, listed for completeness only

These security items surfaced during a separate incorporation/security review (ahead of
the NumberHive/Indigo Tide assignment deed) and are **James's action items already, not part
of this handover's open items** — listed here purely so David has the full picture and
doesn't rediscover them independently:

- A GitHub PAT embedded in `number-hive-complete`'s local `.git/config` remote URL (not
  committed — James rotating).
- `server/.env.tmp` in `number-hive-admin`'s git history (removed at tip, recoverable from
  history — James rotating the credentials it contained).

---

*Compiled 2026-08-28 alongside [`README.md`](README.md) and [`tech-inventory.md`](tech-inventory.md), as part of the CTO
handover package. Re-prioritize freely — the ranking here is a starting judgement, not a
fixed sequence.*
