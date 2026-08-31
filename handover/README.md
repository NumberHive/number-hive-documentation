# Incoming Technical Lead Handover — NumberHive Ecosystem

**Purpose:** everything a new incoming technical lead (David) needs to pick up technical
ownership of NumberHive from James. Written 2026-08-28, cross-checked against the live repos rather than transcribed
from memory or old docs — see the provenance notes throughout.

**Status:** first full pass. The docs in this folder plus the existing `architecture/`
docs together are the handover package. [`open-items.md`](open-items.md) lists what still needs an answer or a
decision before (or shortly after) handover completes — read that one even if you skim
everything else.

---

## Start here

1. **This file** — orientation: what NumberHive is, the repo map, how building/deploying
   actually works day to day.
2. **[`tech-inventory.md`](tech-inventory.md)** — domains, hosting, databases, every
   third-party service, costs, and where credentials live. Reconciled against what's actually
   deployed, not just what an older internal doc said.
3. **[`open-items.md`](open-items.md)** — things that need a decision, a verification, or an
   action, ranked by how much it'd hurt if left undone. Some of these are pre-existing gaps
   (backups, security policy, incident response were never documented); some are things I
   found doing the reconciliation (a cloud cluster that may still be running and billing with
   nothing pointing at it; two zombie repos).
4. **[`access-model.md`](access-model.md)** — who can do what, and how it's actually checked in
   code, across all three systems. Includes two confirmed authorization gaps found while
   compiling it (`remove-user-from-hive`, and a more severe one in the Journey resolvers) — both
   also tracked in `open-items.md` (#22, #23).

Beyond those four, the rest of this folder is product/domain-specific deep dives, written to
answer specific questions raised in the run-up to handover rather than to be read start to
finish:

- **[`analytics-inventory.md`](analytics-inventory.md)** — what analytics/event tracking exists
  for Arcade and Education, where it lives in code, and how it's currently accessed (or isn't).
- **[`arcade-data-model.md`](arcade-data-model.md)** — Arcade's document lifecycles, TTL/index
  behaviour, and how the cross-repo ADR-003 storage contract applies on that side; a companion to
  `../architecture/database-schema-free-game.md`, not a replacement for it.
- **[`async-arcade-architecture.md`](async-arcade-architecture.md)** — how an asynchronous
  player-vs-player game in Arcade moves from invite to a resolved outcome: the concurrency cap,
  the queue-on-cap invite flow, and "Your Turn" surfacing.
- **[`launch-readiness-notes.md`](launch-readiness-notes.md)** — a factual, no-judgement map of
  where in the code each item from the CEO's Launch Brief (25 and 31 August 2026) currently
  stands.
- **[`promotion-runbook.md`](promotion-runbook.md)** — the deploy/promotion pipeline for each of
  the four repos (they're not uniform); flagged as pending a live walkthrough to confirm Render
  dashboard settings actually match what's committed.
- **[`security-remediation-status.md`](security-remediation-status.md)** — status of
  `number-hive-complete`'s security remediation plan, phase by phase, cross-checked against
  current code.

Then, for the actual system architecture (not infra/ops, but what the product *is* and how the
pieces fit together), the existing docs in this repo are the canonical source and I haven't
duplicated them here:

- **[`../architecture/platform-overview.md`](../architecture/platform-overview.md)** — start
  here for the product itself: the whole platform in one diagram.
- **[`../architecture/system-overview.md`](../architecture/system-overview.md)** — the three
  areas (school product, free game, admin), who owns what data, which ADR governs each
  boundary.
- **[`../architecture/environment-urls.md`](../architecture/environment-urls.md)** — the
  single most load-bearing doc for infra reality: every URL/port per environment, cited to
  each repo's actual deploy config, with known drift and contradictions flagged rather than
  papered over. This is what caught the dead GKE/Pulumi path (see below) — read it before
  trusting any older doc (including the original tech inventory) on infra questions.
- **[`../architecture/subdomain-map.md`](../architecture/subdomain-map.md)** — domain scheme
  and rationale (`.app` = customer-facing, `.org` = NH-internal/staff).

---

## What NumberHive is

**Number Hive Education**, the paid product for schools, plus **Number Hive Arcade**, a free
public game that funnels into it, plus emerging internal company-ops tooling. Not one
application — a small family of repos, each owning its own data, talking to each other over
defined APIs/event pushes rather than sharing databases. See the glossary below for how the
product names map onto the terms actually used in code and in the other docs.
[`system-overview.md`](../architecture/system-overview.md) is the authoritative map; the
one-paragraph version:

| Area | Repo | Status |
|---|---|---|
| Number Hive Education (the paid app) | `number-hive-complete` | **Live** — this is the main live system |
| Number Hive Arcade (the free game) | `number-hive-newvis` | Deployed, live at `game.numberhive.app` — currently in friends-and-family testing, not yet publicly launched |
| Company operations / admin | `number-hive-admin` | Deployed and running on Render (service `number-hive-admin`, branch `main`, auto-deploy on, not suspended — confirmed via the Render API) at its Render-issued URL; **`admin.numberhive.org` itself is not yet wired to it** — that DNS name still resolves to a parked GoDaddy IP with an expired TLS certificate, not Render. Functionally, only Google OAuth staff sign-in and the `fg_events` analytics mirror are live; billing/customer data hasn't migrated out of `number-hive-complete` yet |
| Marketing site (`www`) | Not a NumberHive repo | Live — WordPress, externally hosted |
| Ecosystem documentation (this repo) | `number-hive-documentation` | Cross-repo architecture, conventions, and (as of this handover) onboarding docs |

## Glossary — code terms vs. product terms

The product has its own external vocabulary that doesn't always match the internal code/ticket
language. This maps one to the other, with where each lives in code:

| Product term | Code term(s) | Where it lives |
|---|---|---|
| Altitude | `HIQ` / `HiveIQ` (NumberHive Rating) | The scoring engine actually lives in `number-hive-newvis`, not `number-hive-complete` — `src/scoring/Hiq.ts`, `src/scoring/HiqProfile.ts`, backend `backend/src/lib/hiqSeed.js`/`hiqPvp.js`/`hiqDeltaBound.js`, route `backend/src/routes/progression.js`. Design spec sits in `number-hive-complete/docs/game/hiq-and-challenge-score-design.md`. |
| The Rival ladder (Bumble = its first rung) | `Rival` / `RIVALS` | `number-hive-newvis/src/ui/RivalData.ts` — `RIVALS: Rival[]` array, first entry `id: 'bumble', name: 'Bumble'`, "Index 0 (Bumble) is always unlocked". Selector UI: `src/ui/RivalSelector.ts`, `RivalCard.ts`. |
| Challenge / Challenge Score | `Challenge` / `ChallengeScore` | `number-hive-newvis/src/challenge/ChallengeClient.ts` (`createChallenge`/`getChallenge`/`submitAttempt`, backend `backend/src/routes/challenges.js`); score formula in `src/scoring/ChallengeScore.ts` and `src/scoring/ChallengeScoreProfile.ts` — a distinct, per-match/per-seed score from Altitude/HIQ. |
| Live Play (an asynchronous human-vs-human match) | async PvP / `MatchClient` | `number-hive-newvis/src/pvp/MatchClient.ts` — explicit "(Live Play)" labels in the header comment and at `joinMatch`/`listMatches`. |
| Play-Me link (a standing invite link) | "standing link" / `StandingLinkClient` | `number-hive-newvis/src/links/StandingLinkClient.ts` — "the standing Play-Me link + QR flow (CHG-3763, REQ-008)"; backend `backend/src/routes/links.js`. |
| Roster | `Connections` / `ConnectionsClient` | `number-hive-newvis/src/links/ConnectionsClient.ts` — "the known-opponents roster"; backend `backend/src/routes/connections.js`, `backend/src/lib/connectionStats.js`. No equivalent concept in `number-hive-complete`. |
| Hive, and Hive.code (the join code) | `Hive` model, `code` field | `number-hive-complete/backend/src/database/models/hive.ts` — class `Hive`, unique-indexed `code` field, used for student join-by-code. GraphQL: `backend/src/graphql/hive/join-hive/join-hive.service.ts`. |
| Journey (curriculum progression) | `Journey` model | `number-hive-complete/backend/src/database/models/journey.ts` — class `Journey`, `currentStage`/`EStageId`. Related: `backend/src/constants/journey-type.ts`, `backend/src/graphql/journey/journey.resolver.ts`. |

## The infrastructure, in one diagram

[`platform-overview.md`](../architecture/platform-overview.md) already diagrams the *product* shape (which repo owns which data).
This one is the complementary *infra/ops* view — what's actually deployed, where, and the one
open question (the greyed-out GCP box) that [`open-items.md`](open-items.md) #1 asks you to resolve:

```mermaid
flowchart TB
    Dev["Developer"]

    subgraph Domains["Domains — GoDaddy DNS, manual A/CNAME records"]
        AppDomain["numberhive.app\n(customer-facing)"]
        OrgDomain["numberhive.org\n(NumberHive-internal/staff)"]
    end

    Dev -- "git push staging" --> StagingDeploy["Render: auto-deploy on push"]
    Dev -- "git push production\n(no approval gate — real footgun)" --> ProdDeploy["Render: auto-deploy on push"]

    subgraph RenderProd["Render.com — Production (the real, current hosting)"]
        CompleteFE["numberhive-frontend-production\nplay.numberhive.org"]
        CompleteBE["numberhive-backend-production\nbackend.numberhive.app"]
        Temporal["numberhive-temporal-production\n+ managed Postgres"]
        GameFE["numberhive-game-production-frontend\ngame.numberhive.app"]
        GameBE["numberhive-game-production-backend\ngame-api.numberhive.app"]
        AdminSvc["number-hive-admin\n(Render: deployed, branch main,\nauto-deploy on — confirmed via Render API)\nreachable only at its .onrender.com URL"]
        AdminDB[("number-hive-admin-db\nmanaged Postgres + fg_events")]
        MongoAtlas[("MongoDB Atlas\nprimary app DB — confirmed Atlas\n(mongodb+srv:// connection string)")]

        CompleteFE --> CompleteBE --> MongoAtlas
        CompleteBE --> Temporal
        GameFE --> GameBE
        AdminSvc --> AdminDB
    end

    subgraph RenderStaging["Render.com — Staging"]
        GameStaging["numberhive-game-staging-frontend/-backend\nbranch staging, NOT suspended — live today\n(confirmed via Render API)"]
        CompleteStaging["numberhive-*-staging (complete)\nbranch staging, SUSPENDED\n(confirmed via Render API — two generations,\nboth suspended)"]
        StagingLocalNote["staging-local.numberhive.org\n(personally-hosted, developer-provided\ninfrastructure) — retired at handover.\nFrom 1 Oct, complete's staging is Render's\n(currently suspended — needs resuming)"]
    end

    subgraph Legacy["GCP / GKE / Pulumi — abandoned 2026-07-27"]
        GKE["backend-cluster (GKE)\nnode pools, static IP, Cloud NAT\nSTILL RUNNING? — unconfirmed,\nsee open-items.md #1"]
        FirebaseHosting["Firebase Hosting projects\nstill billing? — open-items.md #2"]
    end

    subgraph Marketing["Marketing — not a NumberHive repo"]
        WP["WordPress\nwww.numberhive.app"]
    end

    ProdDeploy --> RenderProd
    StagingDeploy --> RenderStaging

    AppDomain --> CompleteFE
    AppDomain --> GameFE
    AppDomain --> WP
    OrgDomain -. "admin.numberhive.org DNS not wired to\nRender — resolves to a parked GoDaddy IP\n(199.192.24.221, expired TLS cert)" .-> AdminSvc

    GKE -. "no longer deployed to\n(config still on disk in\nnumber-hive-complete/backend/iac/)" .-> CompleteBE
    FirebaseHosting -. "superseded by Render\nfor hosting" .-> CompleteFE
```

**Solid lines are live today. Dotted lines are dead/superseded or not-yet-wired paths** — the
opposite emphasis from [`platform-overview.md`](../architecture/platform-overview.md)'s diagram (there, dotted means "designed but not yet
built"; here, dotted means "used to be true, isn't the deploy path anymore" — or, for
`OrgDomain -> AdminSvc`, "the service exists and deploys, but this specific DNS path to it
doesn't work yet"). The two grey boxes in `Legacy` are exactly [`open-items.md`](open-items.md) #1 and #2 — nobody has
confirmed they're actually turned off, only that nothing deploys to them anymore.

---

## Repos that are *not* part of the live system

Two repos exist in the NumberHive GitHub org that look like they might be live infrastructure
but are not — worth knowing before you go looking for what deploys them:

| Repo | Last commit | What happened to it |
|---|---|---|
| `number-hive-backend` | 2026-03-16 | Superseded — its `iac/` (Pulumi) and `cloudbuild.yaml` were absorbed byte-for-byte into `number-hive-complete/backend/` on 2026-03-14 ("chore: initial monorepo setup"). Confirmed identical content via diff. Nothing has been pushed to it since; it's a fossil, not a fallback. |
| `number-hive-app` | 2026-03-12 | Same story — this was the standalone React Native mobile app repo. `number-hive-complete/CLAUDE.md` now states plainly: *"iOS/Android builds are not maintained"* — the frontend in the monorepo is web-only. |

Neither is archived on GitHub (`archived: false` as of this check) — see [`open-items.md`](open-items.md)
for the recommendation on that.

---

## How building and deploying actually works

**Hosting: Render.com is the real, current answer for every live service.** `number-hive-complete`,
`number-hive-newvis`, and `number-hive-admin` all deploy via `render.yaml` Blueprints. There is
**no live Kubernetes/GKE deployment** despite what an older internal doc says — see the
correction below.

**Branch → environment convention** (from `number-hive-complete/CLAUDE.md`, and consistent
across the other repos):

- `main` (or `develop`, repo-dependent) — integration branch
- `staging` — deploys to a staging environment on push
- `production` — deploys to Render production **automatically on push**. Every repo's own
  CLAUDE.md is emphatic about this: never push to `production` without an explicit
  instruction to do so, every time, no exceptions. This is a real footgun, not boilerplate
  caution — a push to that branch ships to real users immediately.

**Local development** runs via Docker Compose in each repo (`docker-compose.yml` /
`docker-compose.dev.yml`) — see each repo's own `CLAUDE.md`/`README.md` for the exact commands,
they differ slightly per repo.

### The GKE/Pulumi correction — read this before trusting any older doc on infra

The backend architecture was originally built as GCP + GKE + Pulumi-provisioned Kubernetes,
with staging/production namespaces, node pools, Cloud Build triggers, the works. **That was confirmed abandoned by the user on
2026-07-27** (recorded in [`environment-urls.md`](../architecture/environment-urls.md)) — it was an earlier infra
approach that was never cleaned up from disk, not a live path. The actual, current production
backend is `backend.numberhive.app`, served from Render, confirmed both against the live
Render dashboard and against the app's own production secrets file.

**This matters for handover in one concrete way: nobody has confirmed whether the GCP/GKE
infrastructure itself was ever torn down.** "Not used" in the deploy pipeline is not the same
as "doesn't exist and isn't billing." See [`open-items.md`](open-items.md) #1 — this is the single highest-value
thing to verify early, since if the cluster is still running it's both a live cost and a live
(unmonitored) attack surface.

---

## Suggested first week for David

1. Read [`../architecture/platform-overview.md`](../architecture/platform-overview.md), then [`system-overview.md`](../architecture/system-overview.md), then
   [`environment-urls.md`](../architecture/environment-urls.md) in that order — product, then map, then ground-truth infra state.
2. Read [`open-items.md`](open-items.md) in full and pick which items you want to personally verify vs. delegate
   vs. accept as "known gap, revisit later."
3. Get Render dashboard access (all three live services are there) and GCP console access
   (to resolve the GKE question) before anything else — those two unlock verifying most of
   [`open-items.md`](open-items.md) yourself rather than taking this doc's word for it.
4. Get GoDaddy/DNS access sorted explicitly — see [`tech-inventory.md`](tech-inventory.md)'s access section; it's
   currently informally delegated and that should become an explicit, documented arrangement.
