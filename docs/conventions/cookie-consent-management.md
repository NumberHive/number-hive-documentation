# Cookie & tracking consent management — cross-repo convention

**Status:** Draft, with builds now in progress — central gathering point for the discussion started
2026-07-26. As of 2026-07-26, changes have been raised on both `number-hive-complete` and
`number-hive-newvis` against §6's shared contract (see History). Nothing beyond §1 (current state)
and whatever those two changes land is actually implemented yet; this document remains the
reference point both builds should be checked against, and should be updated (not treated as
historical) if either build needs to deviate from §6 while in progress.

## Why this lives here, not in one product repo

`www.numberhive.app`, `game.numberhive.app`, and `play.numberhive.app` are three different
codebases (WordPress, `number-hive-newvis`, `number-hive-complete`) that now sit under one
registrable domain (`numberhive.app`) and are documented in `architecture/platform-overview.md`
§3 and `architecture/subdomain-map.md` §3 as sharing (or wanting to share) a single visitor
identity for attribution. Consent is the gate that sits in front of any of that identity data —
if each repo decides its own consent story independently, the ecosystem ends up with three
different legal bases, three different banners (or no banner), and no way to reason about
whether a piece of attribution data was lawfully collected before it's used. That's exactly the
kind of cross-repo, jointly-accountable concern this repo exists to hold — same reasoning as
`docs/conventions/analytics-and-ops-logging.md`, which this document sits alongside and depends
on (that doc governs what happens to tracking data once collected; this one governs whether it
was lawful to collect it in the first place).

## 1. Current state (as of 2026-07-26)

| Property | Consent mechanism | Tracking that exists today | Consent-gated? |
|---|---|---|---|
| `www.numberhive.app` (WordPress) | **CookieYes** — granular banner with Necessary (always on), Functional, Analytics (toggle), Performance, Advertisement (toggle) | Unconfirmed exactly what fires under each category — no doc anywhere describes what's actually installed | Yes, in shape — categories exist and gate correctly. What's actually behind each category hasn't been audited |
| `game.numberhive.app` (`number-hive-newvis`) | **None** | `clientId` UUID in `localStorage`, `fg_visitors` collection — fires unconditionally on load | **No** — this is the live gap |
| `play.numberhive.org` → `.app` (`number-hive-complete`) | **None** (relies on school-level contractual consent) | `nh_vid` UUID in `localStorage`, `Visitor`/`TrackEvent` collections — fires unconditionally | Not consent-banner-gated; consent basis is the school agreement, not yet documented as covering this specifically |
| `admin.numberhive.org` (proposed) | N/A | N/A — staff-only, no customer visitor tracking by design | Out of scope for a public CMP |

CookieYes is a WordPress plugin — it cannot be dropped into `game` or `play`, both of which are
custom frontends (Phaser/Vite and React Native Web respectively). Each needs its own mechanism,
and — because each sits under a different legal basis (general public vs. anonymous
child-reachable vs. school-consented student data) — "what needs consent" is a different question
on each surface, not a single copy-pasteable answer.

`localStorage`-based identifiers (`clientId`, `nh_vid`) aren't literally cookies, but UK
ICO / EU ePrivacy guidance treats them the same as cookies for consent purposes whenever they're
used for anything beyond strict necessity (attribution/analytics counts as "beyond strict
necessity"). So the absence of a banner on `game` and `play` today is a live gap, not just a
future one to design around.

## 2. Per-surface legal basis and requirement

| Property | Audience | Governing regime | Consent model |
|---|---|---|---|
| `www` | General public, adults and minors | GDPR / PECR / ePrivacy | Full opt-in/opt-out banner (already in place — verify categories match reality) |
| `game` | Anonymous, child-reachable, no age gate | COPPA + UK Children's Code, on top of GDPR/PECR | Needs a consent banner: simple, no dark patterns, non-essential tracking **defaulted off** |
| `play` | Students (under school contract) + teachers/admins (individual accounts) | FERPA / GDPR, school-consent model for student data; ordinary individual consent for the teacher/admin who signs up | Student-side tracking: covered by school agreement (needs documenting as such, not yet done). Teacher/admin-side: ordinary account-holder consent applies — no ad cookies should ever be added here |
| `admin` | NH staff only | Internal policy, not public consent law | No public CMP needed |

## 3. Focus: `game` — enabling teacher attribution to a `play` subscription

This is the current priority. The requirement: a teacher may be advertised to and land on **any**
surface (an ad, `www`, or `game` itself — e.g. trying the free game before recommending it to
their class) and ultimately create a paid account on `play.numberhive.app`. We want to attribute
that subscription back to the original campaign/ad, which means the identity established at
first touch has to survive, unbroken, all the way to the `play` signup — potentially across
devices, sessions, and days.

### What's actually implemented today (verified against `number-hive-complete` code, 2026-07-26 — corrects the more optimistic framing in `subdomain-map.md`)

`subdomain-map.md` §3a describes `POST /api/visitor/link` as an existing endpoint. It isn't — that
was a misreading of the design spec vs. what's actually in the repo. Checked directly:

- `nh_vid` is **`localStorage`-only**, generated client-side by `getVisitorId()`
  (`frontend/src/utils/visitorIdentity.ts`). **There is no cookie anywhere in this codebase
  today** — the "shared cookie" idea below is genuinely new engineering on both `game` and
  `play`, not a small config change.
- `POST /api/visitor/identify` (`backend/src/router/api/visitor/index.ts`) is real, live, and
  **CORS-wildcarded** (`Access-Control-Allow-Origin: *`) — this one is genuinely callable
  cross-origin today. It only creates/touches an anonymous `Visitor` record (UTM/referrer/device
  fingerprint); it does not touch a `User`.
- **There is no `POST /api/visitor/link` route.** The anonymous→user stitching logic
  (`linkVisitor()` in `backend/src/services/visitor.service.ts`) is called directly, server-side,
  from inside the `signup`/`login` GraphQL resolvers (`auth.service.ts`), keyed by whatever
  `visitorId` the **play frontend itself** passes as a GraphQL argument — sourced from its own
  `localStorage`. No HTTP route, no CORS, nothing `game` can call.
- **One real, already-working cross-surface mechanism**: `adoptUrlVisitorId()`
  (`visitorIdentity.ts`) reads a `?nh_vid=<uuid>` URL param on page load and adopts it into
  `localStorage` if no local id exists yet (UUID-format validated). This is shipped code, not a
  design doc — and it's the cheapest lever available for the teacher-attribution requirement,
  since it needs no change on `play`'s side at all.
- **There is no centrally-allocated UID facility anywhere in this ecosystem.** Checked both
  identity implementations directly: `play`'s `getVisitorId()` (`visitorIdentity.ts`) and `game`'s
  `IdentityService` (`number-hive-newvis/src/analytics/Identity.ts`, `anon_client_id`) each
  independently call `crypto.randomUUID()` (with the same hand-rolled fallback) client-side and
  persist the result locally. Neither backend issues an id back to the client — `play`'s
  `POST /api/visitor/identify` and `game`'s `POST /visitors` both *require* the client to already
  supply a UUID (`Joi.string().uuid().required()` on `play`'s side; a plain string-presence check
  on `game`'s) and only validate/persist it, never generate one. Two independent, near-identical
  client-side generators, not one shared client library or a server-issued id. This is why Option A
  works without any central allocator: `game` can simply mint its own UUID and hand it to `play`
  via the URL param — neither backend cares who minted it, and UUIDv4 collision risk is
  negligible. Building an actual central issuance service would be a separate, larger
  architecture decision (a shared identity microservice all surfaces call), not something any
  existing doc plans for — flagged as a new open question, not assumed.

The gap isn't the attribution *model* — it's that (a) nothing outside `play` feeds into it today,
and (b) none of it is currently gated by consent anywhere.

### The proposed mechanism — two options, not one

**Option A — reuse the URL-param handoff that already ships (lower effort, no `play` changes needed):**

1. **Consent gate on `game`** first, regardless of option: a lightweight banner — necessary vs.
   non-essential (attribution/analytics), **defaulted off** for non-essential given the surface is
   child-reachable with no age gate. No "Advertisement" category — this is first-party attribution
   for marketing effectiveness, not third-party ad-tech, and the banner copy should say so.
2. On consent, `game` generates a UUID (or reuses its own `clientId`) and appends `?nh_vid=<uuid>`
   to any outbound link to `play` (e.g. a "recommend to your school" / sign-up CTA).
3. `game` can optionally call the already-CORS-open `POST /api/visitor/identify` on `play`'s
   backend directly, to register that id's first-touch UTM/referrer *before* the teacher ever
   clicks through — pure client-to-cross-origin-API call, no new backend work required.
4. When the teacher lands on `play` and signs up, `adoptUrlVisitorId()` (already shipped) picks up
   the id automatically; `linkVisitor()` (already shipped) stitches it to the new `User` at signup
   — with **no new code needed on `play` at all**.
5. **Limit**: this only survives a single click-through. A teacher who evaluates `game`, closes the
   tab, and signs up on `play` days later via a different path won't be attributed — the same limit
   `platform-overview.md` §3 already flags for the param handoff generally.

**Option B — the shared first-party cookie (`Domain=numberhive.app`), higher effort, closes the gap Option A leaves:**

1. Same consent gate as above.
2. New work on `game-api`: set a shared cookie server-side (not client JS — Safari ITP caps
   JS-set cookies at 7 days) on consent, reusing the `nh_vid` value/format rather than inventing a
   second identifier.
3. New work on `play`: today it has **no code path that reads a cookie for identity at all** —
   `getVisitorId()` would need to check for and adopt this cookie the same way
   `adoptUrlVisitorId()` adopts the URL param. This is real frontend/backend work on `play`, not
   "point it at an existing endpoint."
4. Once adopted into `play`'s `localStorage`, the existing `linkVisitor()` signup path handles the
   rest unchanged.
5. **Consent record**: whatever answer the visitor gives needs a timestamp and a record of what
   was actually offered/accepted, so any later attribution reporting is defensible if challenged.

**Recommendation**: start with Option A — it requires touching only `game` (and the CORS-open
endpoint `play` already exposes), ships something real quickly, and directly serves the "teacher
clicks through from game to play" journey. Option B is worth doing later for the "returns days
later without clicking a link" case, but it's materially more work on `play`'s side than the
architecture docs implied, and shouldn't block getting Option A live.

**Can each surface just mint its own shared cookie if one doesn't exist yet, with a UUIDv4?**
Yes — this is architecturally sound and needs no central coordinator. Each backend (WordPress,
`game-api`, `play-api`) checks incoming request cookies for the shared id; if absent, mints a
UUIDv4 server-side and sets it. Whichever surface a person hits first "wins," and every other
surface just adopts the existing value on the next request — the same reasoning already
established above (neither backend cares who minted the id; UUIDv4 collision risk is negligible).
No single "issuing" service is required for this to work correctly.

### Multi-touch attribution — the actual gap, and it predates any of this cookie work

The cookie/URL-param question above only answers *whether the same visitor id follows someone
across surfaces*. It does not answer the separate, harder problem raised directly: **knowing
every place/ad that brought someone in, not just the first.** Checked this against the actual
code and the spec that governs it (`docs/superpowers/specs/2026-05-16-attribution-architecture-design.md`,
`number-hive-complete`) rather than assuming:

- The spec's own stated goal already includes *"How many marketing touchpoints does a teacher
  typically have before signing up?"* — multi-touch was always the intent, not a new idea.
- The **`Visitor` collection freezes only the first touch, permanently** — `utmSource` etc. are
  explicitly "set once on creation, never overwritten" (`visitor.ts` schema comment;
  `identifyVisitor()` service — a repeat call only bumps `lastSeenAt`). This collection alone can
  never answer "all the places they came from."
- The spec's actual answer for multi-touch is the **`TrackEvent` collection**, which does have
  per-visit UTM fields (`utmSource`/`utmCampaign`/etc., indexed by `utmCampaign`+`action`+
  `createdAt`) intended to be attached to every `page_view`, not just the first.
- **That part isn't wired up.** `frontend/src/utils/analytics.web.ts`'s `logAnalytics()` — the
  function that fires every `page_view` from `RootNavigator.tsx` — sends only `sessionId, area,
  category, action, label, value, metadata, visitorId`. It never calls `getCurrentUtms()` (which
  exists and is exported, but is only ever used inside `identifyVisitor()`, for the one-time call
  to `/api/visitor/identify`). **No return visit under a different campaign is recorded anywhere
  today** — the UTM columns on `TrackEvent` sit unused for this purpose. This is a real, confirmed
  gap inside `number-hive-complete` as it stands, independent of anything to do with `game` or
  cookies.

So: even a perfect shared cookie across all three surfaces would, today, only ever show the very
first ad that brought someone in — the same frozen-first-touch limitation the current design
already has, just extended across more surfaces. Getting "know all the places the user came from"
means finishing the already-designed-but-unbuilt per-visit UTM capture, then extending it to
`game`/`www`.

**Call cadence isn't the problem — checked both call sites directly.** `play`'s `identifyVisitor()`
is deliberately gated to fire once per browser session (`sessionStorage` flag `nh_vid_identified`)
— a new session re-fires it with that session's current UTM params, which is actually the right
cadence for multi-touch. `game`'s equivalent (`main.ts`'s `POST /v1/visitors` call) isn't
session-gated at all — it fires on **every app boot**, with full UTM payload each time, more often
than `play`. In both cases the backend already receives fresh UTM data on every call; the bug is
purely that the write logic (`$setOnInsert` on `game`, "only `updateLastSeen`" on `play`) discards
it after the first insert, no matter how often or how freshly it arrives. So the multi-touch fix
needs **no frontend calling-cadence changes on either repo** — it's a server-side fix only. Worth
normalizing `game`'s every-boot trigger to roughly once-per-session when the append-log is
actually built, so a single person relaunching a PWA repeatedly doesn't inflate the touch count —
a refinement, not a blocker.

### What "recording centrally" should mean, given "one writer per data domain, no shared DB"

Two shapes — one recommended:

- **Recommended — single owner, complete what already half-exists.** `play`'s backend already
  owns the eventual subscription/`User` record and already has a CORS-open ingestion endpoint.
  Extend `POST /api/visitor/identify` (or add a sibling, e.g. `POST /api/visitor/touch`) to
  **append** every touch — surface, UTM/referrer/landingPage, timestamp — against the shared
  visitor id, instead of silently dropping repeats. `www` and `game` call it on every visit
  (consent-gated), and `play`'s own frontend needs the same fix (`logAnalytics()` should attach
  `getCurrentUtms()` too — this bug exists independent of any cross-surface work). At subscription
  time, querying by `visitorId` inside `play`'s own database gives the full multi-touch path with
  zero cross-service joins.
- **Alternative — distributed, joined at report time.** Each surface logs its own touches to its
  own store (stricter reading of "one writer per data domain"), and a read-only reporting layer
  (per `docs/conventions/analytics-and-ops-logging.md`) joins the collections by shared visitor id
  when someone wants the full picture. Lower coupling, but "central" only exists at query time, and
  needs scoped read access spanning otherwise-separate databases.

`play` already implements most of the single-owner shape (schema, CORS, intent) and just has an
unwired frontend — completing that is less new work than building the alternative from scratch.

### Session-start visibility on `play`'s DB requires `game` to call `play` directly — not just session-gating

A natural next question: once `game`'s call is session-gated to match `play`'s cadence, will every
session-start on *either* surface become visible in `play`'s `Visitor`/touch data? Not automatically
— session-gating alone only fixes *how often* `game` calls its own backend. Checked directly:
`game`'s `POST /v1/visitors` call (`main.ts`) goes to `game-api` and writes to `fg_visitors` — a
collection in `number-hive-newvis`'s own database, never `play`'s. The two are entirely separate
todays, regardless of cadence.

Getting genuine cross-surface visibility on `play`'s side needs a second, additional change beyond
cadence-matching: `game` must call `play`'s endpoint (the already CORS-open
`POST /api/visitor/identify`, or its append-only successor per the section above) directly,
cross-origin, tagged with a `surface: 'game' | 'play'` field so the record shows where each session
actually started. This is already implied by Option A's step 3 above, but worth stating explicitly:
"gate `game` on session start" and "make `game`'s sessions visible in `play`'s DB" are two separate
changes, and only the second one delivers the visibility being asked for here.

### Many cookie ids, one registered user — the roll-up problem

A related question: multiple visitor ids (different browsers, devices, or a cleared `localStorage`)
can end up linked to the same registered `User` over that account's lifetime — is that already
accounted for? Checked directly against `play`'s schema and auth code — it's already partially
built:

- `Visitor` carries a **non-unique** `@Index({ userId: 1 })`, with the schema's own comment: "W =
  linkVisitor reverse lookup by userId." A non-unique index on that field only makes sense if the
  intent is many `Visitor` docs pointing at one `userId` — `Visitor.find({ userId })` already
  returns the full set today.
- `stampVisitorUserId()` (`visitor.service.ts`, called from `login` in `auth.service.ts` on every
  sign-in) stamps the current cookie's `Visitor` doc with the account's `userId`, guarded only by
  "does *this* doc already have a `userId`" — it never checks whether some *other* `Visitor` doc is
  already linked to that same account. So a teacher who signs up on one device, then later logs into
  the same account from a second browser with a different `nh_vid`, ends up with two `Visitor` docs
  sharing one `userId` — exactly the many-to-one shape being asked about, and it already works this
  way today without any new code.

The gap is what happens to attribution *after* linking, not the linking itself:
`linkVisitor()` writes `User.acquisitionUtm*`/`acquisitionAt` **exactly once**, from whichever
`Visitor`/`DemoLead` has the earliest timestamp, guarded by the `acquisitionAt` sentinel so it's
never overwritten. So today, only the *first-ever-linked* cookie's attribution becomes the
account's official acquisition record — every subsequent device that links to the same account
produces a `Visitor` row that's discoverable via the `userId` index, but contributes nothing to any
roll-up.

The fix is a natural extension of the append-only touch log already proposed above: give every
touch row both `visitorId` (always present, from the moment it's captured) and `userId` (backfilled
the same way `Visitor.userId` already is — stamped in once known, never required at write time,
never blocking capture of anonymous touches). With that, a single `userId`-scoped query against the
touch log — not a merge or dedupe step, just a filter — returns the complete cross-device,
cross-surface advertising history for an account, for comparison against its eventual subscription
outcome. The existing `Visitor.userId` index is the proof this join key already works in this
schema; the touch log just needs to carry the same key.

### The tension this doesn't fully resolve

`game`'s core audience is children playing anonymously; the attribution requirement is really
about the **adult** (the teacher) evaluating or being referred to the product. The same consent
prompt and the same cookie mechanism will technically apply to both, because the surface can't
reliably distinguish "a teacher is looking at this" from "a child is playing this" at the point
consent is requested. Recommended mitigation: if `game` has (or can have) a distinct
teacher-facing entry point (a "for teachers" landing page, a referral link target, etc.) separate
from the core gameplay loop, put the consent prompt and attribution capture there rather than
inside the moment-to-moment play experience — reduces the chance of a child being shown a
tracking-consent decision that's really meant for an adult. This needs product input, not just
engineering — flagged as open below, not decided.

## 4. Open questions (not decided — flagging, not deciding, here)

1. **Reuse `nh_vid` across `game`/`play`, or mint a new shared identifier name?** Recommended:
   reuse `nh_vid` and its existing schema — avoids a fourth identity space on top of the three
   already documented in `subdomain-map.md` §3.
2. **Who builds the `game`-side consent banner?** This is `number-hive-newvis` repo work — out of
   scope for this documentation repo to implement, but should be raised as a change there.
3. **Does `www` need to emit/read the same shared cookie?** Out of scope for the current
   `game`-focused push, but the same mechanism should extend to `www` eventually for the "ad → www
   → play" journey, not just "ad → game → play".
4. **What happens when a teacher declines consent on `game`?** Recommendation: accept the
   attribution loss for that visitor rather than looking for a workaround — campaign-level
   aggregate reporting (no persistent identifier) is the legally-sound fallback, not a bug to
   route around.
5. **Does `play`'s school-consent model permit *receiving* a marketing-attribution identifier
   about a signing-up teacher?** Likely yes — the teacher/admin creating the paid account is an
   adult account-holder, not a student data subject — but this hasn't had explicit legal sign-off
   and should get it before the `link` call is wired up to accept externally-sourced `nh_vid`
   values.
6. **`www`'s CookieYes categories haven't been audited against what actually fires.** Lower
   priority than `game`, but still an open gap — see §1.
7. **`play`'s existing unconditional tracking (`nh_vid`/`Visitor`/`TrackEvent`) has never been
   documented as covered by the school consent agreement.** Someone with visibility into the
   actual school contracts needs to confirm this, or it's an undocumented assumption, not a
   verified compliance position.
8. **Is a real central identity-issuance service ever worth building, vs. the current
   independent-generators-plus-handoff pattern?** Not needed for Option A (client-side UUIDv4
   generation is fine for this purpose), but if the ecosystem ever wants a single point that
   issues, validates, and audits visitor ids across `www`/`game`/`play` (rather than three
   generators that happen to agree on a UUID format), that's a distinct, bigger architecture
   decision, not something currently planned anywhere.
9. **`number-hive-complete`'s own `page_view` tracking doesn't attach per-visit UTMs today**,
   despite the `TrackEvent` schema and the 2026-05-16 spec both intending it to
   (`logAnalytics()`/`RootNavigator.tsx` never call `getCurrentUtms()`). This is a pre-existing gap
   inside `play` itself, independent of the `game`/`www` work — worth its own fix regardless of
   what happens with cross-surface attribution.
10. ~~Should the append-only multi-touch record live on `TrackEvent` or a purpose-built
    `Touchpoint`/`AttributionEvent` collection?~~ **Resolved 2026-07-26 by the `number-hive-complete`
    build**, and for a stronger reason than the general-purpose-event-stream one this question
    originally flagged: `Visitor` carries a **unique** index on `visitorId`, and
    `attribution.service.ts::getVisitorStats()` runs a `$count` over that collection for
    unique-visitor totals and conversion-rate reporting. Turning `Visitor` itself into an
    append-a-row-per-call collection would silently break that stat. Decision: leave `Visitor`'s
    existing upsert/first-touch behaviour, unique index, `linkVisitor()`, and `stampVisitorUserId()`
    completely untouched, and add the append-only touch log as a **new, separate collection**. Not
    `TrackEvent` either, per the original reasoning (mixed audience/retention needs).
11. **`User.acquisitionUtm*`/`acquisitionAt` are frozen from the single earliest-linked `Visitor`
    and never revisited** once later devices link to the same account (see "Many cookie ids, one
    registered user" above). If the touch log ends up carrying `userId` as recommended, does
    `User.acquisition*` stay as a "first official touch" summary field (fine, if the full history
    lives in the touch log instead), or does it need to become a derived/rollup view itself? Not
    decided — depends on what reporting actually consumes this data.
12. **Read/query path for the new touch-log collection** — the objective ("query by `visitorId`
    gives the full multi-touch path") implies *some* query capability is the actual point of
    building this, not optional follow-up work. Scoped for the initial `number-hive-complete` build:
    a minimal internal query helper (e.g. "all touches for a `visitorId`/`userId`, ordered by
    timestamp") ships now; a full GraphQL resolver or admin-facing report view is legitimate
    follow-up scope, not required day one.
13. **Retention/TTL for the new touch-log collection** — no TTL exists on `Visitor` or `TrackEvent`
    today either, so shipping the touch log the same way (no retention policy for now) is consistent
    with current practice. Revisit if `game`'s (now session-gated, per §6) call volume makes this a
    storage concern in practice — not expected to be, but not verified either.
14. **Rate limiting on the now-more-heavily-used cross-origin endpoint** — `POST /api/visitor/identify`
    is unauthenticated and CORS-wildcarded today with no throttle, and adding `game` as a second
    caller doesn't change that exposure in kind, only in volume. Decision: document the risk, don't
    build a throttle as part of this change — matches the endpoint's existing posture. Worth a
    dedicated hardening pass across *all* the CORS-open endpoints later, not scoped to this change.
15. **Does `game`'s consent-gating scope cover its own pre-existing `/v1/visitors`/`EventTracker`
    tracking, or only the new `game`→`play` identity handoff?** Surfaced 2026-07-26 reviewing
    `number-hive-newvis`'s design: their data flow explicitly carves `/v1/visitors` out as "fires
    regardless of consent, per non-goals" — but no non-goal establishing that actually exists in
    this doc or the original brief, and §1 flags `game`'s *entire* current tracking (not just the
    play-handoff piece) as the live consent gap. Not resolved here — `number-hive-newvis` owes this
    an explicit classification (necessary/exempt, with a one-line rationale, vs. also-gated) in
    their spec, not an implicit default. Whichever way it's decided, "the consent banner shipped"
    and "`game`'s tracking is now consent-compliant" are different claims and shouldn't be
    conflated.

## 5. Recommendation / next steps

- Treat `game` as the priority, and start with **Option A** (§3): raise a change to (a) build the
  consent banner on `game`, and (b) have `game` append `?nh_vid=<uuid>` on outbound links to `play`
  and optionally pre-seed attribution via the existing CORS-open `POST /api/visitor/identify` call
  — both land entirely in `number-hive-newvis`, with **no changes required on `play`**, since
  `adoptUrlVisitorId()` and `linkVisitor()` already handle the rest.
- Treat the shared-cookie approach (**Option B**) as a later, separate piece of work — it requires
  new code on `play` (which has no cookie-reading identity path today) and shouldn't block Option A.
- **Separately, and regardless of the above**, raise a fix in `number-hive-complete` for the
  unwired multi-touch capture (open question 9) — this is a real bug in the existing product, not
  new scope, and blocks "know all the places the user came from" no matter what gets built on
  `game`.
- Get explicit sign-off on open question 5 (school-consent scope covering teacher-side
  attribution) before Option B work starts, since that's the point where `play` would begin
  accepting an externally-sourced identifier rather than only ones its own frontend generated.
- Revisit `www` and the audit in open questions 3 and 6 once the `game` work lands.

## 6. Shared contract — the one reference point both builds must code against

`number-hive-newvis` (`game`) and `number-hive-complete` (`play`) are being built as two
independent changes, in two separate repos, by (potentially) two separate people. They only
interoperate correctly if both build against the exact same contract rather than each inferring
the other side's shape. This section is that contract — pin changes here, not in either repo's
own docs, so there's one place either team can check "is this still what the other side expects."

| Element | Value | Who builds it |
|---|---|---|
| Endpoint | Extend the existing `POST /api/visitor/identify` on `play`'s backend (already live, already CORS-wildcarded) — **do not** stand up a new route. Simplest path per §3 "recording centrally." | `play` |
| `play`'s API origin (for `game` to call cross-origin) | **`https://api.numberhive.app`** in production, `https://api-staging.numberhive.app` in staging (confirmed directly in `number-hive-complete/frontend/env.ts` — this is the same `API_URL` `play`'s own frontend calls). **Not** `api.numberhive.org` — that string doesn't appear anywhere in either repo and was an incorrect assumption in an early `number-hive-newvis` design draft, caught 2026-07-26 before it shipped. `game`'s new `playApiUrl.ts` must resolve to an absolute origin (unlike `game`'s own `apiUrl.ts`, which safely falls back to a relative path only because it's same-origin with its own API) — a relative-path fallback here would silently hit `game`'s own domain instead of `play`'s. | `game` |
| New field: `surface` | `'www' \| 'game' \| 'play'`, optional on the request; **defaults to `'play'`** server-side if omitted, so `play`'s own existing frontend calls keep working unchanged during rollout. `game` must send `surface: 'game'` explicitly on every call. | `play` (accept + default), `game` (send) |
| Identifier | Reuse the `nh_vid` name and UUIDv4 format exactly — do not invent a second field name or id scheme (per §3, open question 1: resolved as reuse). `game` mints this client-side with `crypto.randomUUID()` (already has the identical fallback pattern in `Identity.ts`) if it doesn't already hold one. | `game` |
| Write behaviour | The endpoint appends a touch row on every call instead of freezing on first insert. **Resolved 2026-07-26: this lands in a new, separate collection, not `Visitor` itself** — `Visitor` keeps its existing unique `visitorId` index and upsert/first-touch behaviour untouched, because `attribution.service.ts::getVisitorStats()` depends on that index/behaviour for unique-visitor counts and would break if `Visitor` became append-only (see §4 open question 10). The touch row carries `visitorId`, `surface`, UTM fields, `referrer`, `landingPage`, `country`, timestamp. `game` doesn't need to know the storage details, only that repeat calls are safe and expected, not deduplicated away. | `play` |
| Read path | A minimal internal query helper (touches by `visitorId`/`userId`, ordered by timestamp) ships as part of this build — a full resolver/admin view is legitimate follow-up scope, not required day one (§4 open question 12). | `play` |
| Retention / rate limiting | No TTL on the new collection for now (matches `Visitor`/`TrackEvent`'s existing no-TTL posture); no new rate-limit/throttle added to the endpoint for now (matches its existing unauthenticated, CORS-wildcard posture) — both are documented risk notes, not blockers, for this build (§4 open questions 13–14). | `play` |
| `userId` backfill | Touch rows get `userId` stamped in later, same stamp-if-absent pattern as `Visitor.userId` today (via `stampVisitorUserId()`/`linkVisitor()`), so a `userId`-scoped query rolls up history across devices/surfaces (§3 "Many cookie ids, one registered user"). Nothing for `game` to do here. | `play` |
| URL-param handoff | Query param name is `nh_vid`, e.g. `https://play.numberhive.app/...?nh_vid=<uuid>`, appended by `game` to any outbound link/CTA pointing at `play`. `play`'s `adoptUrlVisitorId()` already reads this — **no change needed on `play`** for this specific piece. | `game` (send), already done on `play` |
| Consent gate | `game` must only mint/send the id, call `play`'s endpoint, and append the URL param **after** the visitor has accepted non-essential tracking on `game`'s own consent banner (§2, §3 — child-reachable surface, defaulted off). If declined, `game` sends nothing cross-origin and drops no `nh_vid` param on outbound links (§4 open question 4: accept the attribution loss). | `game` |
| Call cadence | Once per browser session, mirroring `play`'s existing `sessionStorage` flag pattern (`nh_vid_identified` in `identifyVisitor()`) rather than `game`'s current every-app-boot call. Confirmed in §3 that this is a cadence-only change and doesn't affect the multi-touch fix, which is entirely server-side. | `game` |

If either side needs to deviate from this table, update it here first — that's the point of it living in the shared docs repo rather than in either product's own README.

## History

- 2026-07-26 — Drafted, gathering the discussion started when the user shared a screenshot of
  `www.numberhive.app`'s CookieYes banner and asked what's needed across `www`, `play`, and
  `game`. Synthesizes `architecture/platform-overview.md` §3, `architecture/subdomain-map.md` §3,
  and the audience-separation reasoning in `docs/conventions/analytics-and-ops-logging.md`. Focus
  narrowed to `game` and the teacher-attribution requirement per the user's follow-up.
- 2026-07-26 (same day, follow-up) — §3 corrected after reading the actual `number-hive-complete`
  code (`backend/src/router/api/visitor/`, `backend/src/services/visitor.service.ts`,
  `frontend/src/utils/visitorIdentity.ts`): there is no `POST /api/visitor/link` endpoint and no
  cookie-based identity anywhere in that codebase today — `subdomain-map.md`'s description of a
  "link" endpoint was a misreading of the design spec vs. what's implemented. The real, already-
  shipped mechanism is the `?nh_vid=` URL-param handoff (`adoptUrlVisitorId()`), which needs zero
  changes on `play` to support the teacher-attribution requirement. Split the mechanism into
  Option A (URL param, low effort, ships now) and Option B (shared cookie, real new work on both
  repos, later).
- 2026-07-26 (second follow-up) — Confirmed each surface can symmetrically mint its own shared
  cookie with no central allocator needed (same reasoning as the no-central-UID finding). Investigated
  the separate multi-touch attribution question directly against `number-hive-complete`'s code and
  its own 2026-05-16 spec: found that per-visit UTM capture (the spec's actual mechanism for
  multi-touch, via `TrackEvent`) is designed and schema'd but **not wired up** in the current
  frontend (`logAnalytics()` never attaches UTMs to `page_view` events) — a real, pre-existing gap
  in `play` itself, not something introduced by the `game`/cookie work. Added the "Multi-touch
  attribution" subsection in §3 and open questions 9–10 to capture this.
- 2026-07-26 (third follow-up) — Clarified that matching `game`'s call cadence to `play`'s
  session-gating is not sufficient on its own for cross-surface visibility: `game` currently calls
  its own `game-api` backend, never `play`'s, so a second change (calling `play`'s endpoint directly,
  tagged by `surface`) is required regardless of cadence. Also confirmed, directly against
  `visitor.ts` and `auth.service.ts`, that the many-cookie-ids-to-one-`User` relationship already
  works today via `Visitor`'s non-unique `userId` index and `stampVisitorUserId()`'s stamp-if-absent
  behavior at login — but that `User.acquisitionUtm*`/`acquisitionAt` are frozen from only the
  first-ever-linked `Visitor` and never updated as further devices link. Recommended the proposed
  touch log carry `userId` (backfilled, same pattern as `Visitor.userId`) alongside `visitorId` so a
  `userId`-scoped query rolls up the full cross-device attribution history. Added both subsections to
  §3 and open question 11.
- 2026-07-26 (fourth follow-up) — User confirmed both `number-hive-complete` and `number-hive-newvis`
  need real builds and asked for a shared reference point so the two land compatible with each other.
  Added §6 "Shared contract" pinning the endpoint, `surface` field, identifier reuse, write-behaviour
  change, `userId` backfill, URL-param name, consent gate, and call cadence each build must honour —
  this table is now the single source either team checks before diverging from the other's
  assumptions. Two build briefs (for `number-hive-complete` and `number-hive-newvis`) given to the
  user directly in chat, citing this section and §2–§4.
- 2026-07-26 (fifth follow-up) — User raised both changes on their respective projects
  (`number-hive-complete` and `number-hive-newvis`), against the brief and §6 contract above.
  **Discovered during this step that the previous four follow-ups' edits had never been committed**
  and the working tree had reverted to the original single commit (`e07aadf`) between sessions —
  §3's multi-touch/session-visibility/roll-up subsections, open questions 8–11, §5's extra bullet,
  and §6 itself were all missing from disk despite being referenced in the briefs already sent.
  Reconstructed the full document from the conversation record and committed it, since the two
  builds now in flight depend on it existing and staying put.
- 2026-07-26 (sixth follow-up) — `number-hive-complete`'s assistant, restating the brief before
  starting work, independently caught the same doc-mismatch problem (their checkout still showed
  the pre-reconstruction, one-commit version) and raised it back — confirms the reconstruction/
  commit above was necessary, not precautionary. They also surfaced a genuinely new finding while
  reading the code: `Visitor`'s **unique** `visitorId` index feeds `attribution.service.ts`'s
  `getVisitorStats()` unique-visitor/conversion-rate counts, so the append-only touch log cannot
  live inside `Visitor` itself without breaking that stat — resolves open question 10 (new,
  separate collection; `Visitor`'s existing behaviour stays untouched). Confirmed their read-path,
  retention, and rate-limiting scoping questions and recorded the answers as open questions 12–14
  and in §6's table, rather than leaving them as undocumented verbal agreements.
- 2026-07-26 (seventh follow-up) — Reviewed `number-hive-newvis`'s consolidated design
  (`ConsentManager`/`ConsentBanner`/`playApiUrl`/`playIdentify`/`EducatorCTA` changes) against the
  actual repo code before sign-off. Caught a factual error before it reached a spec doc: their
  proposed `playApiUrl.ts` default (`https://api.numberhive.org`) doesn't exist anywhere in either
  repo — `play`'s real production API origin, confirmed in `number-hive-complete/frontend/env.ts`,
  is `https://api.numberhive.app` (staging: `https://api-staging.numberhive.app`). Added this as a
  row in §6. Also surfaced an unresolved scope question their design had implicitly answered rather
  than decided: whether `game`'s own pre-existing `/v1/visitors`/`EventTracker` tracking is in scope
  for consent-gating (per §1's original framing of the whole-surface gap) or only the new
  `game`→`play` handoff — recorded as open question 15, owed back from `number-hive-newvis` as an
  explicit classification in their spec, not left as an implicit non-goal.
