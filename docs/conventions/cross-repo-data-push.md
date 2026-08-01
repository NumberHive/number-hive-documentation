# Cross-repo data push — convention

## Why this lives here, not in one product repo

The platform's load-bearing rule is **one writer per data domain, no service reaches directly
into another's database** — established by ADR-001 (free game vs. school product) and ADR-005
(admin split), and repeated in `architecture/platform-overview.md` and
`architecture/system-overview.md`. That rule answers *whether* one repo may read another's data
directly (no). It doesn't yet answer *how* data should move between repos when one legitimately
needs to know about the other's activity — e.g. `number-hive-admin` wanting to show usage stats
that only `number-hive-complete` or `number-hive-newvis` actually own.

As of 2026-07-27, at least four such flows are designed or being designed, none built yet:

| Flow | Data | Status |
|---|---|---|
| `number-hive-admin` → `number-hive-complete` | Entitlement projection (`{orgId, plan, status, seats, validUntil}`) | Designed (ADR-005), scaffolding shipped (CHG-3668 on `number-hive-admin`'s board) — not yet receiving real subscription data |
| `number-hive-complete` → `number-hive-admin` | Play/school usage data, for admin's growing dashboard/analysis arm | Designed 2026-07-27 (this document), not built |
| `number-hive-newvis` → `number-hive-admin` | Free-game usage events, mirrored into a ClickHouse analytics store (MVP0.1) | **Build started 2026-08-01.** Receiving side (`number-hive-admin`) in progress — CHG-3975 (ingest+read API) and CHG-3976 (browsing UI). Sending side (`number-hive-newvis` push worker) requested of that repo's Lead same day, not yet built. See "Concrete decisions" below — this flow evolved from "aggregate rollups" (the original 2026-07-27 design) to a raw per-event mirror once the actual use case (browsable event detail, not just rollups) was confirmed. |
| `number-hive-newvis` → `number-hive-admin` | Free-game feedback submissions, for staff triage | Designed 2026-07-27 (this document), not built |

Left to each pair of repos independently, these four would likely grow four different
transports, retry behaviours, and payload shapes. This document defines one shared pattern so
whoever builds each flow doesn't have to re-derive it, and so the flows stay consistent with each
other. It complements — doesn't replace — `docs/conventions/analytics-and-ops-logging.md`, which
governs *what* gets tracked and how it's partitioned (product/growth vs. ops/security) within a
single repo; this document governs how that data (or any other domain data) moves *between*
repos once a legitimate cross-repo need exists.

## The convention

1. **Push, not pull. The data owner always initiates.** The repo that owns a data domain calls
   out to the consumer's API; the consumer never reaches into the owner's database, and never
   polls/queries it directly. This keeps the "who owns this domain" line unambiguous and
   preserves each repo's freedom to change its own schema without breaking a consumer. Mirrors
   the entitlement-projection precedent in ADR-005, generalised to any direction.

2. **Two lanes, matched to urgency — not one flow for everything.**
   - **Event push (real-time)** — fired immediately when something actionable happens. Right fit
     for low-volume, individually-important events: an entitlement change, a feedback
     submission, anything a human might need to act on promptly.
   - **Batch/rollup push (scheduled)** — a periodic aggregate sync (hourly/daily, per flow). Right
     fit for high-volume data where only the aggregate matters, e.g. usage/dashboard stats.
     Pushing every raw event in real time for a KPI dashboard is wasted work.

   This follows the same "cadence proportionate to signal urgency" logic already agreed in
   `analytics-and-ops-logging.md` §4, applied to transport instead of alerting.

3. **The batch lane doubles as reconciliation — this is deliberate, not incidental.** Real-time
   pushes will fail sometimes (consumer down, deploy in progress, network blip). If event push
   were the only mechanism, failures would silently vanish. Instead, every scheduled batch
   re-sends a full or delta snapshot for its window, not just "what's changed since last
   success" — so anything a failed event push dropped gets picked up and self-healed on the next
   batch cycle automatically. No manual backfill, no dead-letter queue to babysit. The two lanes
   are one system: batch is both "the aggregate view" and "the safety net for the event view."

4. **Standard envelope across all flows**, so any receiver can handle any sender consistently:

   ```
   {
     sourceRepo: string,      // e.g. "number-hive-newvis"
     eventType: string,       // e.g. "usage.rollup.daily", "feedback.submitted"
     occurredAt: ISO8601,
     idempotencyKey: string,  // stable per logical event — safe to retry/resend without duplicating
     payload: { ... }         // shape is flow-specific, not standardised
   }
   ```

   The envelope is standard; `payload` is not — each flow defines its own contract for what it
   carries.

5. **Idempotent receivers.** Because batch re-sends and retries are core to the design (§3), every
   receiving endpoint must dedupe on `idempotencyKey` and treat re-delivery as a no-op, not an
   error and not a duplicate write.

6. **Scoped write credentials, one per direction — never a shared blanket credential.** Same logic
   as the scoped *read* credentials already mandated for analytics access in
   `analytics-and-ops-logging.md` §3, applied to the write side: `number-hive-newvis`'s
   credential for pushing into `number-hive-admin` should be able to do exactly that and nothing
   else.

7. **Alert only where urgency warrants it.** A failed entitlement push is worth paging on; a
   delayed usage-stats batch generally isn't. Decide per flow, don't apply one blanket policy —
   consistent with `analytics-and-ops-logging.md` §4's existing urgency-based cadence rule.

## What this does not decide

This document fixes the *shape* of the pattern (push direction, two lanes, envelope,
idempotency, credential scoping) — it deliberately does not pick a specific wire protocol
(plain HTTPS webhook vs. a queue/broker), retry/backoff parameters, or exact endpoint naming.
Those are implementation details for whichever repo builds each flow, and may reasonably differ
by flow (e.g. entitlement push probably wants a simple webhook; a future high-volume usage feed
might outgrow that). If a flow needs to diverge from the shape agreed here, update this document
to record a deliberate, agreed exception rather than silently drifting.

## Open cross-repo action items

- Three of the four flows in the table above are still not built. `number-hive-admin` itself
  now exists and has real code (sign-in, RBAC, audit log, entitlement-push scaffolding) — this
  document's original framing ("`number-hive-admin` doesn't exist yet") is stale as of
  2026-08-01; only the billing/subscription data migration itself hasn't started, not the repo.
- ~~Whoever builds `number-hive-admin`'s ingest side should also decide concrete endpoint
  naming and auth mechanics per flow, and record them in `number-hive-admin`'s own docs,
  cross-linked back here.~~ **Done for the usage-events flow (2026-08-01)** — see "Concrete
  decisions" below. Still open for the other two `→ number-hive-admin` flows (play/school usage,
  feedback submissions) whenever those get built.
- The free-game feedback flow's UI precedent inside admin is the existing **Demo Leads** screen
  (`architecture/page-inventory.md` §5.11) — an actionable table sourced from a different public
  surface. Worth reusing that shape rather than designing a new one.

### Concrete decisions — free-game usage events flow (2026-08-01)

The first flow to actually get built. Recorded here per the action item above, cross-linked
from `number-hive-admin`'s own `server/src/analytics/CONTEXT.md` (added alongside the ingest
endpoint, CHG-3975):

- **Endpoint:** `POST /api/ingest/events` on `number-hive-admin` (dev:
  `https://ripper.prawn-mamba.ts.net:20167/api/ingest/events`).
- **Auth:** `Authorization: Bearer <NEWVIS_EVENTS_PUSH_SECRET>` — a scoped, single-direction
  credential per §6 above, distinct from the entitlement-push secret.
- **Transport tier chosen:** batch/rollup lane (§2), not event-push — but note this flow ended
  up carrying *raw per-event* rows, not aggregates, because the actual driving use case (an
  admin screen where staff browse individual events) needs event-level detail, not just a
  rollup number. The "batch, not real-time" half of the §2 distinction still applies (a
  polling push every ~minute is fine, no LISTEN/NOTIFY needed) — it's the "only the aggregate
  matters" half that turned out not to hold for this particular flow. Worth remembering when
  applying §2's two-lane framework to a new flow: cadence and granularity are separate
  decisions, don't assume batch implies aggregated.
- **Storage on the receiving side:** ClickHouse (`fg_events` mirror table), not Postgres — a new
  infra category for `number-hive-admin`, chosen because the source data is a high-volume,
  insert-only event stream, the same shape ClickHouse is built for. See
  `architecture/environment-urls.md` §4 for the dev connection details.
- **Idempotency:** deliberately deferred for MVP0.1 — the receiver does not yet dedupe on
  `idempotencyKey` despite §5's general requirement. At-least-once delivery with occasional
  duplicate rows was accepted as a reasonable MVP trade-off for analytics data (unlike
  billing data, a duplicate event row is low-stakes); flagged as a known gap to close
  (e.g. via `ReplacingMergeTree`) if it proves to matter in practice, not silently ignored.

## History

- 2026-07-27 — Established, generalising the entitlement-projection pattern from ADR-005
  (`number-hive-complete`) after the user asked how free-game usage/feedback and school-product
  usage data should reach `number-hive-admin`'s growing dashboard/analysis arm, and specifically
  requested a standardised, generalised answer covering both real-time updates and batch
  catch-up/reconciliation.
- 2026-08-01 — First flow (free-game usage events) moved from designed to build-started: the
  user asked for an MVP0.1 proof point ("events browsable in admin coming into ClickHouse from
  the dev public game"), which reshaped the flow from an aggregate rollup into a raw per-event
  mirror (see "Concrete decisions" above) and introduced ClickHouse as new platform
  infrastructure. `number-hive-admin`'s ingest/read API and browsing UI were put on that repo's
  change board (CHG-3975, CHG-3976); the `number-hive-newvis` sending side was requested of that
  repo's Lead in parallel, not yet built as of this entry.
