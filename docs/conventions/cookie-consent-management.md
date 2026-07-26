# Cookie & tracking consent management — cross-repo convention

**Status:** Draft — central gathering point for the discussion started 2026-07-26. Nothing in
this document beyond §1 (current state) is implemented; it records decisions made so far and
flags what's still open.

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

## 5. Recommendation / next steps

- Treat `game` as the priority, and start with **Option A** (§3): raise a change to (a) build the
  consent banner on `game`, and (b) have `game` append `?nh_vid=<uuid>` on outbound links to `play`
  and optionally pre-seed attribution via the existing CORS-open `POST /api/visitor/identify` call
  — both land entirely in `number-hive-newvis`, with **no changes required on `play`**, since
  `adoptUrlVisitorId()` and `linkVisitor()` already handle the rest.
- Treat the shared-cookie approach (**Option B**) as a later, separate piece of work — it requires
  new code on `play` (which has no cookie-reading identity path today) and shouldn't block Option A.
- Get explicit sign-off on open question 5 (school-consent scope covering teacher-side
  attribution) before Option B work starts, since that's the point where `play` would begin
  accepting an externally-sourced identifier rather than only ones its own frontend generated.
- Revisit `www` and the audit in open questions 3 and 6 once the `game` work lands.

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
