# Analytics Inventory — Arcade and Education

**Purpose:** answers the Launch Brief's request (25 August, §"James: please show Dave what
analytics/event tracking already exists, where it lives and how it is currently accessed") and
the follow-up's clarification that "funnel lists" means both the Arcade funnel (the brief's
eight-item "at minimum" list) and the Education funnel (the brief's five-stage flow). Two parts:
a full inventory of what's emitted today in each product, then an item-by-item comparison
against each funnel list — **exists today / partially exists (what's missing) / would be new
work**. No estimates, no recommendations, per the brief's ground rules — status and citations
only. Written for a reader new to either product.

**Compiled 2026-08-31** by direct inspection of `number-hive-newvis` and `number-hive-complete`
source (not from either repo's own docs alone — several claims in this document correct or
supersede what those repos' own analytics docs currently say; see the flagged discrepancies in
§1.4 and §4).

---

## Part A — Number Hive Arcade (`number-hive-newvis`)

### A.1 How tracking works

One function does all client-side tracking: `EventTracker.track<K>(eventName, props)`
(`src/analytics/EventTracker.ts`), a singleton that batches events and `POST`s them to
`/v1/events` (`backend/src/routes/events.js`), which inserts into MongoDB's `fg_events`
collection with `w:0` (fire-and-forget — analytics writes never block or fail the request
they're attached to). No other tracking helper exists anywhere in the frontend. The canonical
list of valid event names is `EVENT_NAMES` / `AnalyticsEventMap` in `src/analytics/eventCatalog.ts`,
drift-tested against `docs/analytics-data-dictionary.md` by an in-repo test.

**`docs/architecture.md` §6 (the repo's own older event-taxonomy doc) is stale and should not be
used.** It lists ~28 flat snake_case event names (`app_loaded`, `game_started`, `invite_created`,
etc.); none of them have a live call site anywhere in current source — they were superseded by
the dot-namespaced catalog below in an earlier change (CHG-2864) and the doc was never updated
to match. `docs/analytics-data-dictionary.md` is the accurate one.

### A.2 Full event inventory

93 event names are catalogued in `eventCatalog.ts`; 91 have a confirmed live call site. Grouped
by domain (event name — emitting file:line — trigger):

| Domain | Events | Emitted from | Trigger (summary) |
|---|---|---|---|
| App/session | `app.loaded`, `app.launched_standalone`, `session.started`, `session.ended`, `engagement.funnel_stage` | `src/main.ts:221,251,257,242`; `EventTracker.ts:260` | App boot, standalone launch, session start/end (tab hidden) |
| Screens | `screen.viewed`, `screen.dismissed` | `GameScene.ts:688`, `ModeScene.ts:1150,1178,3869`, `GamesPanel.ts:577`, `StatsScene.ts:2043` | Generic overlay/screen navigation in/out |
| Install/PWA | `install.prompt_shown/accepted/dismissed` | `src/ui/InstallPrompt.ts:214,224,226` | Add-to-homescreen prompt lifecycle |
| Consent | `consent.banner_shown`, `consent.decision_made` | `src/ui/ConsentBanner.ts:183,194` | Cookie/tracking consent banner |
| Promo / Education CTA | `promo.shown/clicked/dismissed`, `promo.bee_cta_tapped` (**not live — see A.4**), `teacher.landing_viewed`, `teacher.signup_started`, `teacher.link_opened` | `EducatorCTA.ts:303,313,376-377,417-418`; `BeeCTAScene.ts:184-185,295-296,373` | Arcade→Education promo surfaces shown/clicked/dismissed |
| Invites / standing links | `invite.shared`, `invite.link_opened`, `link.created/shared/cycled/revoked/redeemed/game_started/queued` | `ResultCard.ts`, `ModeScene.ts`, `StandingLinkPanel.ts`, `ShareLinkDisplay.ts`, `GamesPanel.ts` | Play-Me link lifecycle |
| Matches (Live Play) | `match.joined`, `match.reconnected`, `match.rematch_created/cancelled`, `match.declined`, `match.invited_player_first_game`, `match.poll_degraded/rate_limited`, `match.start_blocked` | `ModeScene.ts:2196,3567,3373`; `GameScene.ts`; `RematchWaitingPanel.ts`; `LiveMatchPoller.ts`; `GamesPanel.ts` | Async human-vs-human match lifecycle |
| Challenges | `challenge.created/shared/link_opened/accepted/not_found/completed/move_made/score_recorded` | `ResultCard.ts:1042`, `ChallengeResultCard.ts:953`, `ModeScene.ts:3676`, `GameScene.ts:1621` | Seeded solo-vs-AI, shareable challenge lifecycle |
| Games | `game.first_started`, `game.started`, `game.abandoned`, `game.finished`, `game.resigned`, `move.made` | `ModeScene.ts:372,395`; `GameScene.ts:1204,4882,1542,2578`; `GamesPanel.ts:946` | Every game's start/completion/abandon/resign, across every mode |
| AI | `ai.move_computed`, `ai.game_summary` | `GameScene.ts:6278,5512` | AI move computation + per-game performance summary |
| Rival progression | `rival.selected`, `rival.unlocked`, `rival.intro_accepted/declined`, `rival.next_up_selected`, `difficulty.selected`, `quickmatch.default_advanced` | `ModeScene.ts:2857,3004`; `GameScene.ts:4955,4262,4270,4331,4941`; `RivalSelector.ts:849` | Rival-ladder selection and unlock progression |
| Retention/progression | `referral.converted`, `milestone.confidence_reached`, `achievement.unlocked`, `progression.milestone_reached`, `session.progression_summary`, `retention.checked_in` | `ModeScene.ts:386`; `GameScene.ts:4905,5065,5445`; `EventTracker.ts:264,276` | Referral attribution, confidence/achievement milestones, per-session summaries, returning-session flag |
| Account | `account.signin_started/completed/failed`, `account.signout_clicked`, `account.created` | `AccountService.ts:317,367,230,348,372` | Google OAuth sign-in lifecycle |
| Notifications | `notif.permission_requested/granted/denied`, `notif.subscription_created`, `notif.opened` | `PushSubscriptionManager.ts`; `main.ts:278` | Web Push permission/subscription/open lifecycle |
| Other | `onboarding.tutorial_started/completed`, `hiq.certificate_opened/shared/dismissed`, `hint.shown/dismissed`, `sw.update_shown/refresh_clicked/dismissed`, `asset.rig_load_failed`, `error.shown`, `opponent.nickname_set/removed/blocked/reported`, `settings.changed` (**not live — see A.4**) | Various, see `eventCatalog.ts` | Tutorial, HIQ certificate, hints, service-worker update, safety actions |
| **Uncatalogued** | `notif_sent` | `backend/src/lib/matchNotifications.js:87-97` (backend, direct `insertOne`, bypasses `EventTracker`/the catalog entirely) | Server-side, after a successfully delivered Web Push notification (turn or resign) |

Every catalogued event carries the standard envelope (`clientId`, `sessionId`, `eventName`,
`ts`, `appVersion`, `platform`, `props`); `notif_sent` is the one exception, inserted directly
with `sessionId:'server'`/`appVersion:'server'`.

### A.3 How Arcade events are read/consumed

- **No dashboard or aggregation UI exists inside `number-hive-newvis` itself.**
- `backend/src/lib/eventsPushWorker.js` pushes **every** unsent `fg_events` row (no `eventName`
  filter — confirmed by reading the query, `find({_id:{$gt:cursor}}).sort({_id:1}).limit(batchSize)`)
  to `number-hive-admin`'s ingest endpoint, cursor-based, at-least-once, opt-in via env vars —
  see [`../docs/conventions/cross-repo-data-push.md`](../docs/conventions/cross-repo-data-push.md)
  for the envelope format.
- **On the receiving side**, `number-hive-admin/server/src/analytics/eventValidation.ts` was
  checked directly for this document: it validates envelope **shape** only (non-empty strings,
  parseable timestamp, platform enum, JSON-serializable `props`) — **there is no event-name
  whitelist in current admin code.** This corrects `number-hive-newvis/docs/ANALYTICS_ROADMAP.md`,
  which describes specific events (`engagement.funnel_stage`, `retention.checked_in`,
  `ai.game_summary`) as blocked by an admin-side whitelist gap (tracked there as CHG-4180/
  CHG-4171) — that whitelist mechanism does not exist in the admin code as it stands today, so
  either it was removed since that roadmap entry was written, or the roadmap entry was never
  accurate against shipped code. **Unverified: confirm with James** which of the two it is; this
  document reports what the code shows now, not the history.
- `docs/analysis-process.md` (newvis) documents `fg_events` (product/growth) and
  `fg_system_events` (ops/security) as deliberately separate stores, "do not join or blend the
  two." It also records that no scoped read-only Atlas user exists yet for either collection,
  which is blocking an already-built analysis script (`npm run analyze:ai-wallclock`) from ever
  being run against production.

### A.4 Documented but not live — explicit flags

- **`promo.bee_cta_tapped`** — in the current catalog and its JSDoc claims a call site in
  `ModeScene.ts`, but no such call exists anywhere outside the catalog file and a synthetic unit
  test. Likely drifted after a refactor.
- **`settings.changed`** — explicitly self-documented in the catalog as forward-looking
  scaffolding with no UI yet; not a drift bug.
- Conversely, **`notif_sent`** (A.2) is live in production but appears in none of
  `eventCatalog.ts`/`EVENT_NAMES`/`docs/analytics-data-dictionary.md`.

### A.5 Arcade funnel comparison

The Launch Brief's core funnel narrative is *visitor → Bumble → challenge → recipient → human
match → return*; the "at minimum" list below is the eight concrete signals it asks for
visibility on.

| # | Signal (from the brief) | Status | Detail |
|---|---|---|---|
| 1 | Game starts/completions | **Exists today** | Start: `game.started` (`ModeScene.ts:395`, `GameScene.ts:4218`; first-ever start also gets `game.first_started`). Completion: `game.finished` (`GameScene.ts:4882`, fires for every mode). Mid-play abandon: `game.abandoned` (`GameScene.ts:1204`). |
| 2 | Bumble completion | **Would be new work** | No event or prop is scoped specifically to "beat Bumble." `game.started` carries an optional `rivalId`, but `game.finished` does not carry a matching join key for vs-AI games, so a specific Bumble win can't be reliably reconstructed from current events. The nearest proxies — `achievement.unlocked` (`first_win`, rival-agnostic) and `rival.unlocked` (fires when the ladder advances past Bumble, but only on a player's *first* such win) — are not the same signal and are noted in code comments (`src/scoring/Achievements.ts:305-307`) as an acknowledged overlap, not a substitute. |
| 3 | Challenge creation | **Exists today** | `challenge.created` — `ResultCard.ts:1042`, `ChallengeResultCard.ts:953`. |
| 4 | Challenge-link opens | **Exists today** | `challenge.link_opened` — `ModeScene.ts:3676`, fired on arrival via `?challenge=<id>`. |
| 5 | Human matches started/completed | **Exists today** | Live Play (async 1v1): start = `match.joined` (`ModeScene.ts:2196,3567`; note `game.started` is not fired for this mode — verified `launchGame('pvp_challenge', ...)` does not call the vs_ai/vs_human-only start tracker); completion = `challenge.completed` and `game.finished` both fire from the same closing path (`GameScene.ts:1621,1636`). Local pass-and-play: `game.started`/`game.finished` with `mode:'vs_human'`. |
| 6 | Returning users / retention | **Exists today**, as a mix of a stored field and a discrete event | Primary signal: `fg_visitors.sessionCount`/`lastSeenAt`, incremented on every visitor upsert (`backend/src/routes/visitors.js:37-66`). Discrete event: `retention.checked_in` (`EventTracker.ts:276`), fires at session end only when `sessionNumber > 1`. Caveat found in `docs/ANALYTICS_ROADMAP.md`: the `daysSinceInstall` prop on that event is lazy-set for older clients, no backfill — under-counts that cohort. |
| 7 | Rival progression | **Exists today** | `rival.selected/unlocked/intro_accepted/intro_declined/next_up_selected`, `quickmatch.default_advanced`, `difficulty.selected` — see A.2. |
| 8 | Education CTA clicks | **Exists today** | Every Arcade→Education click fires both `promo.clicked` and `teacher.link_opened` at all three CTA surfaces (persistent strip, post-game overlay, bee CTA) — `EducatorCTA.ts:376-377,417-418`; `BeeCTAScene.ts:184-185,295-296,373`. Further down-funnel: `teacher.landing_viewed`, `teacher.signup_started`. |

**6 of 8 exist today without gaps; 1 (returning users) exists with a noted data-quality caveat
for one cohort; 1 (Bumble completion) would be new work.**

---

## Part B — Number Hive Education (`number-hive-complete`)

### B.1 How tracking works

A first-party system, no third-party analytics SDK on the frontend (checked `frontend/package.json`
and grepped for `gtag`/`dataLayer`/`googletagmanager`/Segment/Amplitude/PostHog — none found).
Mixpanel (`backend/package.json`, `"mixpanel": "^0.18.0"`) exists but is used narrowly, inside
`logGameEnded()` (`backend/src/services/analytics.ts`) for in-app gameplay analytics — unrelated
to the launch funnel below.

| Component | File | Role |
|---|---|---|
| `Visitor` model | `backend/src/database/models/visitor.ts` | Typegoose, collection `visitors`. Unique `visitorId`, first-touch UTM/referrer/landingPage fields, `userId`/`linkedAt`/`linkedVia` once linked to an account. Analogous to Arcade's `clientId`/`fg_visitors`. |
| `TrackEvent` model | `backend/src/database/models/analytics/track-event.ts` | Collection `trackEvents` in `school_hive`. Generic `{sessionId, area, category, action, label, value, metadata, visitorId, userId, utm*, country, hiveId, viewport*}` shape. |
| Tracking service | `backend/src/services/tracking.ts` | `trackEvent()` (client-submitted), `trackServerEvent()` (server-generated), `getTrackFunnelStats()` (groups `trackEvents` by `action`+`label` within an `area`). |
| Ingest endpoint | `backend/src/router/api/track/track.handler.ts` | `POST /api/track` — plain REST endpoint (not GraphQL), Joi-validated. |
| Visitor identify endpoint | `backend/src/router/api/visitor/visitor.handler.ts`, `backend/src/services/visitor.service.ts` | `POST /api/visitor/identify` — upserts `Visitor` (first-touch fields) or bumps `lastSeenAt` on repeat. |
| Attribution linking | `backend/src/services/visitor.service.ts` (`linkVisitor`, `stampVisitorUserId`) | Stamps acquisition data from the matching `Visitor` (or a matching `DemoLead` by email, as fallback) onto the `User` doc at signup/login. |
| Frontend web tracking | `frontend/src/utils/analytics.web.ts` (`logAnalytics`), `frontend/src/utils/visitorIdentity.ts` | Web-only; the native equivalent (`analytics.ts`) is a no-op stub. |
| Admin funnel queries | `backend/src/graphql/admin/analytics/analytics.resolver.ts` | `adminTrackFunnelStats`, `adminVisitorStats`, `adminAcquisitionStats` — GraphQL, admin-gated, already aggregate `trackEvents`/`Visitor` by area/action/label and by UTM/country/conversion-rate. |

**A real gap found for anonymous visitors:** the frontend's route-change page-view tracker
(`RootNavigator.tsx`) calls `logAnalytics`, but `logAnalytics` (`analytics.web.ts` line 18) no-ops
immediately if `nh_session_id` isn't already present in `AsyncStorage` — and that key is only
ever set **after** login/signup succeeds (`Login.tsx`, `SignUp.tsx`, `VerifyEmail.tsx`). So an
anonymous visitor viewing the current sign-in/landing screen produces **no page-view trackEvent
today** — only the one-time `Visitor` upsert. This matters directly for stage 1 below, and for
whatever replaces the landing page under the brief's "must do" item.

**Two independent demo prototypes exist**, each with its own inline tracking, separate from the
main app's `analytics.web.ts`/GraphQL layer entirely:
- `frontend/public-demo-v2/index.html` — current, a static standalone HTML/JS prototype (not
  part of the deployed React Native Web app or its router). Fires `_track('page_view')` on load
  and `_track('stage_entered', 'screen_N')` on every screen transition, posting to `/api/track`
  with `area:'demo', category:'funnel'`.
- `frontend/public-demo/index.html` — an older v1, same pattern.

### B.2 Education funnel comparison

| # | Stage (from the brief) | Status | Detail |
|---|---|---|---|
| 1 | Education landing page views | **Partially exists — missing:** an actual per-view tracking event for anonymous visitors. A `Visitor` record is created on first touch (`identifyVisitor()`, unconditional on web app boot, capturing `landingPage`), but this is a single stored field per visitor, not a repeat-view event log, and the route-based page-view tracker is gated on a session cookie anonymous visitors don't have yet (§B.1). Whoever builds the brief's replacement landing page should not assume the existing `logAnalytics` plumbing works for anonymous visitors — confirmed it doesn't, today. | `frontend/src/navigators/RootNavigator.tsx`, `frontend/src/utils/analytics.web.ts:18`, `frontend/src/utils/visitorIdentity.ts` |
| 2 | Demo starts | **Exists today, but only for the standalone demo prototype** — `_track('page_view')` on load and `_track('stage_entered','screen_1')` when the flow starts, both in `frontend/public-demo-v2/index.html`, posted to `/api/track` with `area:'demo'`. Not wired into the main app/router at all. | `frontend/public-demo-v2/index.html` (`_track()`, `goto(1)`) |
| 3 | Demo completion | **Would be new work.** No dedicated "demo completed" action exists anywhere. The demo tracks `stage_entered` per screen number, so completion can only be inferred by which screen a session last reached — there is no discrete completion event, and the demo's own final "stage" for one path (teacher) *is* the sign-up form itself, not a separate completion state. | `frontend/public-demo-v2/index.html` (`goto()`, screen `s11`) |
| 4 | Signup / free trial signup | **Exists today**, for both surfaces. Demo-prototype signup: `_track('signup_submitted', ...)` posts to `backend/src/router/api/demo/demo.api.ts`'s `signup()`, which both stores a `DemoLead` doc and fires a server-side `trackEvent` (`action:'signup_submitted'`). Real account signup: the GraphQL `signup` mutation calls `linkVisitor()` (stamping acquisition fields onto the new `User`) and fires `trackServerEvent(... action:'signup' ...)`. | `backend/src/router/api/demo/demo.api.ts:26-85`; `backend/src/graphql/user/auth/auth.resolver.ts` → `auth.service.ts:55-147` |
| 5 | Hive creation | **Exists today.** `createHive()` fires `trackServerEvent(... area:'onboarding', action:'hive_created' ...)` immediately after the `Hive` document is written, and separately sets a one-time `firstHiveCreatedAt` milestone on the owning `User` if not already set. | `backend/src/graphql/hive/create-hive/create-hive.service.ts:52-66`; `backend/src/database/models/user.ts:100-105` |

**2 of 5 stages (signup, Hive creation) exist today without gaps. 2 (landing views, demo starts)
partially exist — the demo-start tracking works but is isolated to a standalone prototype not
wired to the app that will presumably replace the landing page; landing-page tracking has a
concrete blocking gap for anonymous visitors. 1 (demo completion) would be new work.**
`adminTrackFunnelStats`/`adminVisitorStats`/`adminAcquisitionStats` already exist as a plausible
place to extend rather than building new aggregation infrastructure, once events for all five
stages exist consistently — noted as an existing tool, not a recommendation to use it.

### B.3 ADR-003's proposed acquisition tracking — implemented differently than specified

[`docs/adr/003-migration-safety.md`](../../number-hive-complete/docs/adr/003-migration-safety.md)'s
"Migration sequence — school product correlation" section proposes a nested
`teacher.acquisition = {source, free_game_ref, referral_url, signup_date}` object, populated by
capturing a `?ref=` query parameter at signup.

**What was actually built is different in both shape and mechanism, though the underlying intent
was delivered:** the `User` model (`backend/src/database/models/user.ts:131-143`) has flat
`acquisitionUtmSource/Medium/Campaign/Content/Term`, `acquisitionReferrer`, `acquisitionLandingPage`,
`acquisitionAt`, `acquisitionVisitorId`, `acquisitionLinkedVia` fields — not the nested shape the
ADR sketches, and there is no `teacher` collection at all (`teacher` is a `UserType` enum value
on the unified `User` model). There is no `free_game_ref` field and no `?ref=` query-parameter
capture anywhere in either repo's source (checked both `backend/src` and `frontend/src`).
Correlation is instead done via `visitorId` (a `nh_vid` key, cross-domain-adoptable via a URL
param) plus a full UTM parameter set, joined at signup time by `linkVisitor()`
(`backend/src/services/visitor.service.ts:58-131`) — which also falls back to matching a
`DemoLead` by email if no `Visitor` match is found. **This document reports this as a factual
divergence, not a defect** — the built mechanism is arguably broader than the ADR's sketch — but
if `arcade-data-model.md` or any other document is read as implying ADR-003 was executed
literally, this section corrects that.

---

## Summary — confirm-with-James markers in this document

1. **§A.3** — whether the admin-side event-name whitelist gap described in
   `number-hive-newvis/docs/ANALYTICS_ROADMAP.md` (CHG-4180/CHG-4171) was since removed from
   `number-hive-admin`'s code, or whether that roadmap entry was never accurate against what
   shipped. Current admin code has no such whitelist.
2. **§A.4** — `promo.bee_cta_tapped`: documented as live with a specific call site that doesn't
   exist in current code. Not investigated further than confirming the discrepancy.

No other item in this document required a "confirm with James" marker — every other exists/
partial/new-work classification was resolved directly from source.

---

*Compiled 2026-08-31. Arcade findings from direct inspection of `number-hive-newvis/src/analytics/`,
`backend/src/routes/events.js`, `backend/src/routes/visitors.js`, `backend/src/lib/eventsPushWorker.js`,
`backend/src/lib/matchNotifications.js`, and `docs/analytics-data-dictionary.md`/`ANALYTICS_ROADMAP.md`/
`analysis-process.md`. Education findings from direct inspection of `number-hive-complete/backend/src/database/models/visitor.ts`,
`backend/src/database/models/analytics/track-event.ts`, `backend/src/services/tracking.ts`,
`backend/src/services/visitor.service.ts`, `backend/src/router/api/track/track.handler.ts`,
`backend/src/router/api/demo/demo.api.ts`, `backend/src/graphql/user/auth/auth.service.ts`,
`backend/src/graphql/hive/create-hive/create-hive.service.ts`, `backend/src/database/models/user.ts`,
`frontend/src/utils/analytics.web.ts`, `frontend/src/utils/visitorIdentity.ts`,
`frontend/src/navigators/RootNavigator.tsx`, `frontend/public-demo-v2/index.html`, and
`docs/adr/003-migration-safety.md`. Admin whitelist claim checked directly against
`number-hive-admin/server/src/analytics/eventValidation.ts`.*
