# NumberHive System Overview

**Migrated from `number-hive-complete/docs/SYSTEM-OVERVIEW.md` on 2026-07-25.** This is the
canonical documentation reference for the whole NumberHive system — not just the school
product `number-hive-complete` contains. This page is the map: what each area is, which repo
it lives in, and which ADR governs its relationship to the others.

For any decision that crosses one of these boundaries, write (or update) an ADR in
`number-hive-complete/docs/adr/` (ADRs for the free-game ↔ school-product pair and the
admin/amber split remain authoritative there — see that repo's ADR-001 amendment and this
repo's README for the reasoning on what stays put vs. what belongs here), regardless of which
repo the change actually lands in.

---

## The four areas

| Area | Repo | Status | What it is |
|---|---|---|---|
| **School product** | `number-hive-complete` | Live | The paid educational product — GraphQL/MongoDB/Temporal backend, React Native Web frontend. Also currently contains company-operations admin tooling (see below — slated for extraction). |
| **Free game** | `number-hive-newvis` (sibling repo) | In development | Public, free-to-play PWA — the "front door" funnel into the school product. Phaser 3 / TypeScript / Fastify / MongoDB. Deployed at `nhvis.puddicombe.com`; targeting `game.numberhive.app`. |
| **Company operations / admin** | `number-hive-admin` (sibling repo — created, in development) | In development | Billing/subscriptions (Stripe), customer/organisation records, user administration, projects/tasks — **the billing/subscription data itself hasn't migrated yet**; still lives *inside* `number-hive-complete` (`backend/src/graphql/admin/`, `frontend/.../AdminPanel/`) pending that work. What's real in the new repo as of 2026-08-01: Google OAuth sign-in, an RBAC/access-control model, an audit log, entitlement-push scaffolding (ADR-005), and (MVP0.1, in progress) a new ClickHouse-backed analytics store mirroring `number-hive-newvis`'s event stream — see `docs/conventions/cross-repo-data-push.md`. |
| **Amber** | `amber` (sibling repo) | Early / shell stage | NumberHive's AI staff-member product — persona-scoped PWA. Consumes company data via a scoped, auditable API once `number-hive-admin` exists; does not hold billing/customer data itself. |

---

## Who owns what data

| Data domain | Owner | Consumers |
|---|---|---|
| School/game accounts, hives, journeys, gameplay | `number-hive-complete` | — |
| Free-game player/session data | `number-hive-newvis` | — (correlated with school signups via a handoff key at conversion, not shared storage — see `number-hive-complete/docs/adr/001-free-game-infrastructure.md`) |
| Subscriptions, billing, customer/org records, projects/tasks | `number-hive-admin` (repo exists; data itself still: `number-hive-complete`, pending migration) | `number-hive-complete` (entitlement projection only, via event push), `amber` (scoped read API) |
| Free-game analytics events (mirrored copy, not the source of truth) | `number-hive-admin` — a ClickHouse mirror of `number-hive-newvis`'s `fg_events`, pushed cross-repo (MVP0.1, in progress 2026-08-01) | — internal admin dashboard only, so far |
| Amber's own working data (chat state, notes, approvals she generates) | `amber` | — |

**One writer per data domain.** No service reaches directly into another's database. See
`number-hive-complete/docs/adr/005-numberhive-admin-separation-and-amber-data-access.md` for
the reasoning and the entitlement event-push mechanism specifically. For the generalised,
cross-repo shape that mechanism (and its planned siblings — play/game usage data flowing the
other way into `number-hive-admin`) should follow, see
[`conventions/cross-repo-data-push.md`](../docs/conventions/cross-repo-data-push.md).

---

## Repo boundaries and why

| Boundary | Decision | Governing ADR |
|---|---|---|
| `number-hive-complete` vs. `number-hive-newvis` (free game) | Separate repos, separate databases, same Atlas project; correlate via handoff key, not shared storage | [ADR-001](../../number-hive-complete/docs/adr/001-free-game-infrastructure.md) *(in `number-hive-complete`)* |
| `number-hive-complete` vs. `number-hive-admin` (company ops) | Extract admin into a new, separate repo/service; `number-hive-complete` gets a local entitlement projection via event push, not a live API call or shared DB | [ADR-005](../../number-hive-complete/docs/adr/005-numberhive-admin-separation-and-amber-data-access.md) *(in `number-hive-complete`)* |
| Amber vs. `number-hive-admin` | Amber stays a peer API consumer with scoped credentials; not fused into a single "NumberHive OS" | [ADR-005](../../number-hive-complete/docs/adr/005-numberhive-admin-separation-and-amber-data-access.md) *(in `number-hive-complete`)* |

See also [`subdomain-map.md`](subdomain-map.md) in this repo for how these areas map to actual
subdomains, and the current state of cross-property visitor tracking; and
[`environment-urls.md`](environment-urls.md) for the actual dev/staging/production URL for each
area, cited directly to each repo's deploy config.

---

## Status note

`number-hive-admin` does not exist yet — ADR-005 is a proposal, not an implemented
architecture. Until it's built, company-operations tooling continues to live inside
`number-hive-complete` as described in ADR-005's "Context" section. This document
describes the target shape as well as the current one; check each area's "Status" column
above before assuming something has been built.

---

## Migration note

This document previously lived at `number-hive-complete/docs/SYSTEM-OVERVIEW.md`. It was
moved here because it describes the whole ecosystem (all four areas, cross-repo data
ownership, repo boundaries) rather than anything scoped to the school product or to the
free-game/school-product pair specifically — squarely this repo's charter per its README.
The ADRs it references (001–005) are **not** moving — those stay authoritative in
`number-hive-complete` per that repo's ADR-001 amendment (2026-07-24) and this repo's README,
and are linked cross-repo above rather than duplicated.

A stub pointing here still needs to be left at the old location in `number-hive-complete`,
and that repo's `CLAUDE.md` pointer updated — out of scope for this repo's Assistant to edit
directly (different repo, different Lead); flagged to the user to relay to that project's team.
