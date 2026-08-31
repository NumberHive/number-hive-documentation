# NumberHive Documentation

This repository is the **central knowledge base** for the NumberHive product ecosystem — the single place where architecture, cross-repo processes, and shared conventions are documented so they stay in sync across every repo that makes up NumberHive.

**Start here:** [`architecture/platform-overview.md`](architecture/platform-overview.md) — the
whole platform in one diagram, why the game/education/admin areas are kept apart, and the
current state of (and gaps in) tracking a person as they move across surfaces.

## Why this repo exists

NumberHive is not a single application — it's a **collection of repositories** that together form one product, served across multiple subdomains. As the number of repos grows, so does the risk of architectural knowledge living only in one person's head, or in a Slack thread, or scattered across READMEs that drift out of date with each other.

This repo solves that by being the **one place** that documents:

- How the various NumberHive repos relate to and depend on each other
- Cross-repo events, contracts, and processes (e.g. things that happen in one repo that other repos need to know about or react to)
- Shared architectural decisions that apply across the ecosystem
- Which subdomain each part of the product is served from, and how they fit together

If it affects more than one repo, or needs to stay consistent across repos, it belongs here.

## The NumberHive ecosystem

NumberHive currently includes (and will continue to grow):

- **Marketing site** (`numberhive.app` / `www.numberhive.app`) — WordPress, live today
- **Number Hive Education** (`number-hive-complete`) — the paid, learning-focused school product
- **Number Hive Arcade** (`number-hive-newvis`) — the free, public-facing game experience
- **Administrative facilities** *(upcoming)* — internal/admin tooling for managing the platform; proposed as its own service, `number-hive-admin` (see ADR-005 in `number-hive-complete`, referenced from `architecture/subdomain-map.md` and `architecture/system-overview.md`)

Each of these is expected to be served from its own subdomain under the NumberHive domain, while sharing underlying data, architecture, and processes. Convention (agreed 2026-07-24, see `architecture/subdomain-map.md`): customer-facing properties live on `.app`, NH-internal/staff properties live on `.org`. Other supporting and prototype repositories exist alongside these in the [NumberHive GitHub org](https://github.com/NumberHive) — this document will be expanded to map out which are active production services, which are prototypes, and how each one fits into the whole as that picture is confirmed.

## What belongs in this repo

- **Architecture** — system diagrams, service boundaries, data ownership, integration points between repos
- **Cross-repo processes** — workflows, events, or triggers that span more than one repo (e.g. "when X happens in the game repo, the admin repo needs to know about it")
- **Subdomain map** — which repo serves which subdomain, and how routing/deployment ties them together
- **Shared conventions** — naming, data formats, auth patterns, or other standards multiple repos need to agree on
- **Decision records** — why architectural choices were made, so the reasoning isn't lost
- **Brand guidelines & assets** — the canonical brand guide PDF, logo/favicon assets, brand voice copy, and colour tokens, kept here as source copies (not just links) since there are only a handful of key assets — see `brand/README.md`

## What does *not* belong here

- Implementation detail specific to a single repo (that stays in that repo's own README/docs)
- Code — this is a documentation-only repository
- Decisions scoped to a specific *pair* of repos rather than the whole ecosystem — e.g. `number-hive-complete/docs/adr/001-004` govern the free-game ↔ school-product boundary specifically (database separation, migration safety, offline/CDN strategy) and stay there. ADR-001 was amended 2026-07-24 to record the boundary between "authoritative for this pair" (stays in `number-hive-complete`) and "authoritative for the whole ecosystem" (belongs here).

## Documents

### Conventions

| Document | What it covers |
|---|---|
| `docs/conventions/analytics-and-ops-logging.md` | Cross-repo convention for audience-separated tracking (product vs. ops/security), database access control, visualization, cadence, and the privacy floor. Instantiated per-repo — see the table inside for which repos have adopted it. |
| `docs/conventions/cookie-consent-management.md` | Draft — cookie/tracking consent requirements across `www` (CookieYes), `game`, and `play`; current focus is a consent-gated shared cookie on `game` to attribute teacher acquisition through to a `play` subscription. |
| `docs/conventions/cross-repo-data-push.md` | Cross-repo convention for how one repo's data legitimately reaches another (e.g. play/game usage → `number-hive-admin`'s dashboard, admin's entitlement → play) once direct DB access is off the table: push not pull, real-time event lane + batch/reconciliation lane, standard envelope, idempotency, scoped write credentials. Generalises the entitlement-projection pattern from ADR-005. |
| `docs/conventions/deployment-version-tracking.md` | Cross-repo convention requiring every tracked event to carry `deployedAt` (Unix ms) and `versionHash` (commit SHA) — build-time-injected, not read at runtime — so timing questions and code-state debugging are answerable from the event itself. Reference implementation: `number-hive-admin`; `number-hive-newvis` next in line. |

### Brand

| Document | What it covers |
|---|---|
| `brand/README.md` | Canonical home for NumberHive brand guidelines and key assets (guide PDF, logo, favicon, brand voice copy, colour tokens) and the cross-repo brand-consistency picture — what's adopted where, what still drifts. |

### Architecture

| Document | What it covers |
|---|---|
| `architecture/platform-overview.md` | **Start here.** The whole platform in one diagram (marketing site, free game, education app, admin), why the three areas are architecturally kept apart, and a dedicated section on cross-surface user tracking — what's built, what's missing, and the proposed shared-cookie fix. |
| `architecture/system-overview.md` | The whole-ecosystem map — the three areas (school product, free game, company-ops/admin), who owns what data, and which ADR governs each repo boundary. Migrated from `number-hive-complete/docs/SYSTEM-OVERVIEW.md` 2026-07-25; a stub is expected at the old location once that repo's team applies it. The ADRs it references (001–005) stay in `number-hive-complete` — linked cross-repo, not duplicated. |
| `architecture/platform-strategy.md` | Working document — Game App vs. Dashboard App split proposal covering the free game, paid game, and educator/admin experience. Migrated from `number-hive-newvis` 2026-07-24; a stub remains there. Domain names in this doc are superseded — see `subdomain-map.md`. |
| `architecture/page-inventory.md` | Point-in-time snapshot of the paid app's (`number-hive-complete`) full screen inventory, used as supporting evidence for `platform-strategy.md`. Migrated alongside it 2026-07-24; a stub remains in `number-hive-newvis`. Includes an appendix on `number-hive-newvis`'s own CHG-2330 screens. |
| `architecture/subdomain-map.md` | The authoritative subdomain table (WordPress marketing site, free game, education app, proposed admin split) and the current state of cross-property visitor tracking — what's centralised, what isn't, and what's designed-but-unconfirmed. Synthesised 2026-07-24 from specs across `number-hive-complete` and `number-hive-newvis` that hadn't previously been cross-referenced. |
| `architecture/environment-urls.md` | **Quick-reference table** — every NumberHive URL/port (www, play+backend, game+game-api, admin) for dev/staging/production, cited directly to each repo's own deploy config. The "what do I actually type in a browser" companion to `subdomain-map.md`'s "why". Compiled 2026-07-27; flags known drift (e.g. `subdomain-map.md`'s game domains now stale — they've since gone live). |
| `architecture/database-schema-education-app.md` | Entity-relationship reference for `number-hive-complete`'s `school_hive` database (Typegoose/Mongoose) — ER diagram plus field-level tables for identity/session, hives/journeys, gameplay, organisations/subscriptions/billing, and attribution/analytics events. Compiled 2026-08-28 directly from `backend/src/database/models/*.ts`. |
| `architecture/database-schema-free-game.md` | Entity-relationship reference for `number-hive-newvis`'s `free_game` database (raw MongoDB driver, no ODM) — ER diagram plus field-level tables for visitor/account identity, events, matches/challenges, the social graph, and ratings/push. Compiled 2026-08-28 from `backend/src/db/indexes.js` and route/lib call sites; supersedes `number-hive-newvis/docs/architecture.md` §4.2 for `fg_matches`. |

### Handover

| Document | What it covers |
|---|---|
| `handover/README.md` | **CTO handover entry point** (James → David, 2026-08-28) — orientation, repo map, how build/deploy actually works, and the GKE/Pulumi correction. Links out to `tech-inventory.md`/`open-items.md` below and to the architecture docs above rather than duplicating them. |
| `handover/tech-inventory.md` | Domains, hosting (Render — the real current answer, correcting the old GCP/GKE inventory), databases, third-party services, costs, and access, reconciled against what's actually deployed. Credentials are never inlined — `[see Google Doc: ...]` placeholders point at the (out-of-repo) source of truth. |
| `handover/open-items.md` | Prioritized list of what needs a decision, verification, or action before/after handover — e.g. confirming the old GKE cluster and Firebase Hosting projects are actually torn down, not just unused. |

## Status

This repo has just been established as the central documentation store. Content is being built out incrementally — expect this README and the structure around it to expand as architecture and cross-repo processes are captured.

## Contributing

When you make a change in any NumberHive repo that affects another repo, or that other developers need to understand to work across the ecosystem, document it here rather than (or in addition to) the originating repo.
