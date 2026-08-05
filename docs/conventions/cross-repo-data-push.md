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
| `number-hive-newvis` → `number-hive-admin` | Free-game usage events, mirrored into an `fg_events` table on admin's Postgres DB | **Confirmed live end-to-end on dev, 2026-08-03 — production still pending admin's non-dev environments.** Receiving side shipped to production 2026-08-02T19:08:19 — CHG-3975 (ingest+read API) and CHG-3976 (browsing UI); storage was originally a dev-only ClickHouse store (MVP0.1), migrated to Postgres by CHG-4093 the same day. Sending side — CHG-3977, `eventsPushWorker.js` — deployed to production 2026-08-01 07:43, correctly dev-scoped. A smoke test on 2026-08-03 proved the full chain (`fg_events` insert → worker poll → POST → 2xx → cursor advance) works: a marked test event was delivered and the push-worker's cursor doc advanced accordingly, ~65s later, matching the 60s poll interval — independently corroborated from admin's side too (their `fg_events` row's `ingested_at` matches newvis's cursor-advance timestamp to within 12ms). Not yet running against staging/production — both sides remain dev-only until admin has those environments. See "Concrete decisions" below — this flow also diverges from the "aggregate rollups" shape the rest of this table describes: it's a raw per-event cursor push, an acknowledged exception recorded below. |
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

   Separately, per `docs/conventions/deployment-version-tracking.md`, every tracked event
   (including ones later pushed cross-repo via this envelope) should also carry `deployedAt`
   (Unix ms) and `versionHash` (commit SHA) — added once at the source repo's tracking layer,
   travelling with the event through the push. That convention's `deployedAt` is deliberately
   Unix ms rather than this envelope's ISO8601 `occurredAt`: `occurredAt` times the event,
   `deployedAt` times the deploy that produced it — different questions, both worth keeping.

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

- Two of the four flows in the table above are still not built (play/school usage data, and
  free-game feedback submissions). The free-game usage-events flow is now built on both ends
  (2026-08-02 receiving side, 2026-08-01 sending side) but end-to-end live traffic is
  unconfirmed — see the flow table and "Concrete decisions" above; needs a mutual check with
  `number-hive-newvis` before calling it live. `number-hive-admin` itself now exists and has
  real code (sign-in, RBAC, audit log, entitlement-push scaffolding) — this document's original
  framing ("`number-hive-admin` doesn't exist yet") is stale as of 2026-08-01; only the
  billing/subscription data migration itself hasn't started, not the repo.
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
- **Storage on the receiving side:** initially ClickHouse (`fg_events` mirror table), chosen
  because the source data is a high-volume, insert-only event stream, the same shape
  ClickHouse is built for. **Superseded 2026-08-02 (CHG-4093):** migrated onto
  `number-hive-admin`'s existing Postgres database (same `fg_events` table name, same
  `DATABASE_URL` as the rest of the app) — Render, the production hosting target, doesn't run
  a separate ClickHouse container, so the dev-only ClickHouse infra category never made it to
  production. The ClickHouse dev container, `@clickhouse/client` dependency, and
  ClickHouse-specific admin tooling were removed entirely; no historical dev data carried
  over. See `architecture/environment-urls.md` §4 for current connection details.
- **Idempotency:** originally deferred for MVP0.1 with ClickHouse. Resolved as part of the
  Postgres migration (CHG-4093) — deduping is now enforced via a `UNIQUE(row_key)` constraint
  + `ON CONFLICT DO NOTHING` on write, rather than ClickHouse's `ReplacingMergeTree`/`FINAL`
  read-time dedup. §5's general idempotency requirement is now met for this flow.
- **Retention:** Postgres has no native TTL the way ClickHouse did. An open follow-up idea
  (CHG-4097, not yet built) proposes a 90-day retention purge job for `fg_events`.
- **Receiving side independently validated (`number-hive-admin`):** confirmed 2026-08-03 by
  `number-hive-admin`'s Lead — the CHG-3975 ingest endpoint has been live and exercised since
  2026-08-01, initially via manual/synthetic `POST`s per CHG-3976's own acceptance criteria (UI
  validation wasn't blocked on the push worker existing). So the receiving side is verified
  working on its own merits, independent of whether real newvis traffic has reached it yet —
  it's "ready and waiting," not just "deployed and unverified."
- **Sending side (`number-hive-newvis`) — firsthand confirmation, 2026-08-03.** Directly
  verified by `number-hive-newvis`'s own Lead against their repo, change board, and dev box
  (superseding the earlier secondhand relay via `number-hive-admin`'s Lead, noted above as
  unverified — now resolved):
  - **Deployed, not just merged.** CHG-3977 integrated to `develop` 2026-08-01 02:05, promoted
    to staging 07:35, promoted to production 07:43. `eventsPushWorker.js` lives in
    `backend/src/lib/`, wired into `server.js`/`index.js` as designed.
  - **Design as documented:** cursor-based on `fg_events` `_id` (not a time window), POSTs to
    `NEWVIS_EVENTS_PUSH_URL` with `Authorization: Bearer ${NEWVIS_EVENTS_PUSH_SECRET}` using the
    standard envelope from §4, 60s poll interval, 200 rows/batch, at-least-once delivery (cursor
    only advances on a 2xx). No idempotency-key dedup on the sending side — relies on the
    receiving side's `UNIQUE(row_key)` dedup instead, confirmed as intentional
    belt-and-braces, not a gap.
  - **Env vars are set, but only on the dev box** (`nhvis.puddicombe.com`, the coder-team box's
    port 20056 — per that repo's `DEPLOY.md`, "dev" there means whatever's running on that box,
    not a persistent Render/Docker environment). `NEWVIS_EVENTS_PUSH_URL` matches admin's
    CHG-3975 dev endpoint (`https://ripper.prawn-mamba.ts.net:20167/api/ingest/events`)
    exactly; `NEWVIS_EVENTS_PUSH_SECRET` is set (non-empty). Correctly **not** set in
    staging/production, matching CHG-3977's scoped-dev-only design — no risk of an accidental
    prod push.
  - **Initial check (earlier 2026-08-03) found no evidence of flow yet:** `fg_events` had 0
    documents, no `fg_events_push_state` cursor document at all, dev server stopped. Verdict at
    that point: "wired, unverified," not "confirmed live." Newvis's Lead offered to run a smoke
    test; superseded by the confirmed result below.
  - **Confirmed live end-to-end, same day.** Newvis's Lead started the dev server and found the
    cursor doc *already* showed a successful push from earlier that day
    (`lastPushedAt: 2026-08-03T15:04:49Z` — i.e. it had been working intermittently already,
    this wasn't the first success). They then POSTed a clearly-marked smoke-test event
    (`eventName: smoke.test.cross_repo_push`, `clientId: smoke-test-client-cross-repo-push`) to
    the local ingest endpoint; it landed in `fg_events` at `_id: 6a70df7203547a6d1b02ad70`.
    ~65s later (one poll cycle) the cursor doc advanced to that exact `_id`, timestamped
    `2026-08-03T18:36:08.887Z` — the cursor only advances on a 2xx from the receiving endpoint
    (verified in the worker's code), so this is a genuine confirmed delivery, not just an
    attempted send. **The full chain — `fg_events` insert → worker poll → POST to admin's dev
    endpoint → 2xx → cursor advance — is confirmed working, on the dev box, as of 2026-08-03.**
  - **Independently corroborated from the receiving side, same day.** `number-hive-admin`'s
    Lead queried their Postgres `fg_events` table directly and found the row:
    `client_id: smoke-test-client-cross-repo-push`, `event_name: smoke.test.cross_repo_push`,
    `source_repo: number-hive-newvis`, `ingested_at: 2026-08-03T18:36:08.875Z` — matching
    newvis's cursor-advance timestamp (`18:36:08.887Z`) to within 12ms, and an
    `idempotency_key` embedding the same Mongo `_id` twice, exactly as designed. This is a
    fully two-sided confirmation (sending-side cursor advance + receiving-side row, matched to
    the millisecond) — as solid as this kind of check gets without production traffic. Row is
    safe to delete/ignore; both Leads treated it as confirmed and closed.
  - **Scope note — this confirms dev, not staging/production.** Env vars remain deliberately
    unset outside the dev box (per CHG-3977's scope, and because `number-hive-admin` doesn't
    have staging/production environments of its own yet — see `environment-urls.md` §4). Don't
    read "confirmed live" as "live in production" — it's the first proof the *mechanism* works
    end-to-end; the production version of this flow still needs admin's non-dev environments to
    exist first.
- **`EventsPage.tsx` has moved on since CHG-3976's original spec.** Two later
  `number-hive-admin` changes touched the same file: CHG-4069 (event search over `fg_events`)
  and CHG-4093 (the ClickHouse→Postgres storage migration above) — the change-tracking system
  logged partial rollback/overwrite warnings at merge time (165 and 19 lines respectively)
  because both modified a file CHG-3976 had already shipped. Per `number-hive-admin`'s Lead,
  this is expected iterative evolution, not data loss. Practical implication for anyone
  documenting "current" `EventsPage.tsx` behaviour: treat CHG-3976 (eventName/date-range
  filters, pagination) as the *original* shape, not the current one — CHG-4069 and CHG-4071
  (URL-synced filter state, still landing as of 2026-08-03) extend it further and are the more
  current reference for what the page actually does today.

### Agreed exception — raw-row push, not aggregate rollup

Per "What this does not decide" above, a flow that diverges from this document's shape must
record the exception here rather than silently drift. This flow is that exception:

Both the §2 lane framing and this document's original 2026-07-27 table entry described
`number-hive-newvis` → `number-hive-admin` usage data as a **batch/rollup** push — periodic
aggregates, not raw events. What actually got built (and is recorded under "Concrete
decisions" above) is a **raw per-event cursor push**: every individual `fg_events` row is
sent, deduped at the row level on both ends, not rolled up at all. This was a deliberate
decision on both sides once the real driving use case (an admin screen where staff browse
individual events, not a KPI number) was confirmed — not a silent deviation by either repo.
Cadence is still batch-shaped (60s polling, not a live push per event), which is why it still
broadly fits the *transport* half of §2's framing; it's specifically the "aggregate" half of
"batch/rollup" that doesn't apply here. Any future flow reusing this document's two-lane
framework should treat cadence and granularity as independently choosable, not assume batch
implies aggregated — the same lesson already flagged under "Transport tier chosen" above.

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
- 2026-08-02 — CHG-3975 and CHG-3976 shipped to production. Same day, CHG-4093 migrated the
  entire events pipeline off ClickHouse onto `number-hive-admin`'s existing Postgres database
  (new `fg_events` table, dedup via `UNIQUE(row_key)` + `ON CONFLICT DO NOTHING`) — done to
  unblock the Render deployment, which doesn't run a separate ClickHouse container. The
  ClickHouse dev container, `@clickhouse/client` dependency, and ClickHouse-specific admin
  tooling were removed. Filter/facet/search semantics are unchanged; only the storage engine
  changed. No historical dev data carried over (fresh empty table). Open follow-up idea
  CHG-4097 proposes a 90-day retention purge job for `fg_events`, since Postgres lacks
  ClickHouse's native TTL. See "Concrete decisions" above for the updated detail.
- 2026-08-03 — Sending side confirmed built: `number-hive-newvis` shipped CHG-3977
  (`eventsPushWorker.js`) on 2026-08-01, relayed via `number-hive-admin`'s Lead. Recorded the
  raw-row-cursor-vs-aggregate-rollup divergence as a formal agreed exception (see dedicated
  section above), per this document's own "record a deliberate, agreed exception rather than
  silently drifting" rule. Both sides of the flow now exist in code, but end-to-end live
  traffic is **not yet confirmed** — the sending worker is deliberately dev-only until admin
  has non-dev environments, and no one has checked ingest logs for real batches yet. Do not
  describe this flow as "live" elsewhere in the docs until that check happens.
- 2026-08-03 (later same day) — Follow-up from `number-hive-admin`'s Lead, prompted by the
  user prioritising `number-hive-newvis` activity visibility ahead of its friends-and-family
  launch: (1) confirmed exact production-ship timestamps for CHG-3975/CHG-3976
  (2026-08-02T19:08:19); (2) confirmed the CHG-3975 ingest endpoint has been independently
  validated live since 2026-08-01 via manual/synthetic `POST`s, so the receiving side is
  verified "ready and waiting" regardless of newvis's push-worker status (see "Concrete
  decisions" above); (3) flagged that `EventsPage.tsx` has been extended past CHG-3976's
  original spec by CHG-4069/CHG-4093 (partial-overwrite merge warnings — expected, not data
  loss) and is still being extended by CHG-4069/CHG-4071 (URL-synced filter state); (4)
  **importantly, clarified that the earlier "CHG-3977 shipped 2026-08-01" claim was always a
  relay, not a firsthand confirmation** — `number-hive-admin` has no visibility into
  `number-hive-newvis`'s board or deploys and recommended asking that repo's Lead directly.
  Documentation Lead has messaged `number-hive-newvis`'s Lead directly for independent
  confirmation of deploy status and observed live traffic; pending reply as of this entry —
  see "Concrete decisions" above for the exact caveat, and update this history once resolved.
- 2026-08-03 (later still) — `number-hive-newvis`'s Lead replied with firsthand confirmation,
  checked directly against their repo/board/dev box: CHG-3977 is deployed to production
  (integrated 2026-08-01 02:05, staging 07:35, production 07:43), `eventsPushWorker.js` is
  live and wired exactly as designed, and the required env vars are set correctly on the dev
  box only (matching admin's CHG-3975 dev endpoint) — never on staging/production, per its
  intentional scope. However, the dev box's `fg_events` collection is empty and there's no
  `fg_events_push_state` cursor document, so the worker has never actually pushed anything;
  their dev server is also currently stopped. Their own verdict: "wired, unverified," not
  "confirmed live." Documentation Lead accepted their offer to run a smoke test (start the dev
  server, seed a real event, watch for a cursor doc / a 2xx in admin's ingest logs tagged
  `sourceRepo: "number-hive-newvis"`) — outcome pending, will be recorded here once reported
  back. See "Concrete decisions" above for full detail.
- 2026-08-03 (smoke test result) — Newvis's Lead ran the smoke test and reported back: the flow
  is **confirmed live end-to-end on the dev box**. The cursor doc already showed an earlier
  successful push that same day (`lastPushedAt: 2026-08-03T15:04:49Z`, not triggered by this
  test), and a fresh marked smoke-test event (`smoke.test.cross_repo_push`) was inserted,
  picked up by the next poll, POSTed to admin's dev ingest endpoint, and confirmed delivered —
  the cursor advanced to that event's exact `_id` at `2026-08-03T18:36:08.887Z`, which by
  design only happens on a 2xx response. This is the first confirmed proof the full mechanism
  (insert → poll → push → 2xx → cursor advance) works, not just that both ends independently
  exist. Documentation Lead asked `number-hive-admin`'s Lead to cross-check their ingest
  logs/DB for the same batch as independent corroboration from the receiving side; pending
  reply. Scope reminder: this confirms the **dev** environment only — the flow remains
  deliberately unconfigured for staging/production until `number-hive-admin` has those
  environments (`environment-urls.md` §4). See "Concrete decisions" above for the full trace.
- 2026-08-03 (fully closed out) — `number-hive-admin`'s Lead independently corroborated the
  smoke test by querying Postgres directly: the `fg_events` row's `ingested_at`
  (`2026-08-03T18:36:08.875Z`) matches newvis's cursor-advance timestamp to within 12ms, and
  its `idempotency_key` embeds the same Mongo `_id` twice as designed — a clean, two-sided,
  millisecond-matched confirmation. Flow status is now genuinely "confirmed live (dev)," not
  provisional. Two unrelated but useful corrections/cautions surfaced in the same reply,
  recorded in their respective homes: (1) the old dev-only ClickHouse container on `ripper`
  was **not actually decommissioned** when CHG-4093 migrated the app off it — only the app-level
  dependency/tooling was removed; the container itself is still running, just stale (see
  `environment-urls.md` §4 for the correction); (2) `number-hive-admin`'s dev server has been
  **flapping** (health-check failures) since ~18:30 on 2026-08-03, after an unrelated backend
  change — didn't affect this smoke test (landed cleanly at 18:36:08, before the flapping
  started) and production is unaffected, but hold off further high-volume dev-environment
  testing (from either repo) until that's confirmed stable. Smoke-test row is safe to
  delete/ignore; both Leads treated this as closed.
