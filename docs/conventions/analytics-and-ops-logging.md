# Analytics & ops logging — cross-repo convention

## Why this lives here, not in one product repo

Multiple NumberHive repos persist their own usage/diagnostic data, but they share one Atlas
project (see `number-hive-complete/docs/adr/001-free-game-infrastructure.md`: "same Atlas
project... separate databases"). Access control, visualization tooling, and privacy rules for
that data are Atlas-project-level and ecosystem-level concerns, not single-repo ones. If each repo
invents its own answer independently, the patterns will drift, someone will eventually reach for a
shared blanket credential "because it was already lying around," and the ecosystem's child-safety
commitments will end up enforced inconsistently between products that serve the same children.

This document is the one canonical version of the convention. Repos should link to it and describe
their own instantiation, rather than re-deriving or re-explaining the reasoning locally.

## The convention

1. **Two audiences, always kept separate.**
   - *Product/growth* — screens, funnels, feature usage, growth/marketing events.
   - *Ops/security* — faults, rate-limiting, latency, auth-failure/abuse signals.

   Keep them in separate collections/tables per repo. Never blend them into one dashboard and
   never join them in an analysis — ops data (e.g. an IP address captured for abuse detection) sits
   closer to PII than product data and must not leak into product-facing reporting.

2. **No bespoke admin UI or query API for either audience.** Use the database platform's own
   tooling — for MongoDB/Atlas: Compass or `mongosh` for ad-hoc queries, **Atlas Charts** for
   anything recurring. A custom internal dashboard is surface area to build, maintain, and secure,
   for a need the platform already covers natively.

3. **Scoped, read-only database users, one per audience — never a shared blanket credential.**
   Because the Atlas project is shared infrastructure across repos, provision these consistently
   (same naming convention, same permission shape) rather than each repo choosing its own pattern.
   A single "whoever has `MONGODB_URI` has full read/write to everything" credential is the
   default failure mode to avoid.

4. **Cadence proportionate to signal urgency.**
   - Ops/security signals warrant a proactive check — a scheduled query or an Atlas Charts
     threshold alert — because nothing else pages anyone when they occur.
   - Product analytics is fine on a periodic (sprint/monthly) review cadence tied to product
     decisions, not incident response.

5. **Privacy: aggregate-first, everywhere, non-negotiable.** Per the ecosystem's child-safety
   floor (`number-hive-complete/docs/game/product-design-principles.md` — minimal data, no dark
   patterns, no humiliation leaderboards), no analysis output, even internal, should surface
   individual per-user behaviour by identifier. Any field that sits closer to PII than the rest of
   a collection (e.g. an IP address recorded specifically for abuse detection) is restricted to the
   ops/security audience only and never joined into product-side analysis — in every repo, not
   just wherever it happened to be added first.

6. **Every tracked event (either audience) carries deployment version info.** Per
   `docs/conventions/deployment-version-tracking.md`: a build-time-injected `deployedAt`
   (Unix ms) and `versionHash` (commit SHA), so a wrong-looking event is debuggable back to
   exactly which deploy produced it, without cross-referencing deploy logs by hand.

## Per-repo instantiation

| Repo | Product/growth collection | Ops/security collection | Repo-local reference |
|---|---|---|---|
| `number-hive-newvis` (public game) | `fg_events` (`free_game` db) | `fg_system_events` (`free_game` db) | `docs/analytics-data-dictionary.md`, `docs/ops-events-dictionary.md`, `docs/analysis-process.md` |
| `number-hive-complete` (school product) | `trackEvents` (`school_hive` db) | *not yet documented here* | `docs/superpowers/specs/2026-05-16-attribution-architecture-design.md`, `docs/superpowers/specs/2026-05-17-main-app-event-tracking-design.md` |
| WordPress marketing site (`www.numberhive.app`) | *unconfirmed — see `architecture/subdomain-map.md` §3c* | *unconfirmed* | — |
| Administrative facilities *(upcoming, `admin.numberhive.org`)* | — | — | see `architecture/subdomain-map.md` for the proposed `number-hive-admin` split (ADR-005) |

Add a row here when a repo adopts this pattern. If a repo's implementation needs to diverge from
the convention above, either resolve the divergence or update this document to record a
deliberate, agreed exception — don't let repos silently drift apart on data they're jointly
accountable for.

See `architecture/subdomain-map.md` for the fuller picture of how (and whether) visitor identity
is actually stitched together across the marketing site, free game, and education app today —
this table only tracks which repos have adopted the audience-separation pattern, not whether
their identity spaces are unified.

This convention governs *what* gets tracked and how it's partitioned within a single repo's own
database. It says nothing about how that data (or any other domain data) should reach a
*different* repo once there's a legitimate need — e.g. `number-hive-admin` wanting to show
`fg_events`-derived usage stats. That's a separate, complementary convention — see
[`cross-repo-data-push.md`](cross-repo-data-push.md).

## Open cross-repo action items

- **Scoped read-only Atlas users (§3) don't exist yet, in any repo.** This should be set up once,
  coordinated across whichever repos need access, rather than per-repo. Currently anyone holding
  the shared `MONGODB_URI` has full read/write across every database in the Atlas project.
- **No repo currently has proactive alerting configured on its ops/security collection** (§4,
  first bullet) — only after-the-fact persistence.

## History

- 2026-07-24 — Established, generalizing the pattern first written up for `number-hive-newvis`
  (CHG-2864 analytics dictionary, CHG-2882/CHG-2888 ops events dictionary) after recognizing the
  access-control and privacy pieces are Atlas-project-level, not free-game-specific.
