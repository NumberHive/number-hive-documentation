# NumberHive Platform Strategy — Architecture Discussion

**Status: Working document — decisions under consideration, not yet finalised.**
**Date: June 2026**

This document records a strategic architecture discussion covering the relationship between the free game, the paid game, and the educator/admin experience. It is intended as a reference for team discussion before formal implementation decisions are made.

---

## 1. Background & Motivation

The current paid game (`number-hive-complete`) was built as a hybrid React Native / web application to serve both mobile and web from a single codebase. This introduced constraints:

- UI framework choices (React Native) not well-suited to game juice or animation richness
- All three user types (players, teachers, staff) tangled into one codebase
- Mobile-first architecture imposes bundle and API overhead inappropriate for a high-volume free game

The free game (`number-hive-newvis`) was started as a Phaser/Vite exploration. It has demonstrated that a web-only, Phaser-based game frontend is a better foundation for the player experience going forward — for both free and paid.

**The conclusion from this discussion: the free game's tech stack is the right direction for the player experience across the whole platform, not just the free tier.**

---

## 2. The Three User Types

We have identified three distinct user audiences with fundamentally different needs:

### 2.1 Players (Free and Paid)

- Children and casual adult players
- Primary goal: play an engaging, rewarding game
- Need: fast load, low friction, high game juice, fun animations
- Auth state: may be anonymous (free) or registered (free or paid)
- Interface style: **game-first, immersive, Phaser-rendered**

### 2.2 Teachers / Educators

- Teachers, tutors, parents acting as class organisers
- Primary goal: understand student performance, manage class groups (Hives), guide learning
- Need: data clarity, class oversight, student progress patterns, subscription management
- Auth state: always authenticated (paid subscription)
- Interface style: **data-dense, analytical, standard web UI**
- Future scope: personalised tuition plan suggestions; surfacing anonymous players who match a class

### 2.3 NumberHive Staff (Admin)

- Internal team
- Primary goal: platform oversight, analytics, support tooling
- Need: platform-level metrics, user management, subscription health
- Auth state: always authenticated (staff role)
- Interface style: **standard admin web UI — similar in character to educator dashboard**

---

## 3. The Core Architectural Insight

> **The real split is not "free vs paid" — it is "player experience vs educator/admin experience".**

These two experience types have different UI technology requirements, different performance profiles, and different design languages. Keeping them in the same codebase (as the current paid app does) forces compromises in both directions.

The educator and staff interfaces (types 2 and 3) are likely similar enough in character to share a single dashboard codebase with role-based feature gating, rather than being built as entirely separate applications.

---

## 4. Interface Areas

This leads to **two frontend applications** (plus the existing backend):

### 4.1 Game App (Phaser + Vite)

- Serves all players: anonymous free, registered free, paid subscribers
- Feature flags gate capabilities based on user state (see Section 5)
- Optimised for: fast initial load, smooth animation, low friction
- Target initial bundle: < 80kb gzipped (game core + tracking only)
- Teacher/admin UI: **none** — no educator features live here
- This repo (`number-hive-newvis`) is the prototype and intended foundation

### 4.2 Dashboard App (Standard React)

- Serves teachers and NumberHive staff
- Role-based: teacher sees class/hive management; staff see platform analytics
- Subscription and payment management lives here (Stripe flows)
- Optimised for: data density, clarity, standard web conventions
- Technology: standard React — no Phaser, no game assets
- Does not need to be low-friction or small-download; users are authenticated and returning

---

## 5. Feature Flag Model (within Game App)

Within the game app, user capabilities are determined by what the backend reports about the current session:

| User State | Capabilities |
|---|---|
| Anonymous visitor | Play game (solo or via invite link), UTM + country captured, play history in session |
| Registered free user | All above + persistent history, class join, nickname saved |
| Paid subscriber | All above + full hive games, premium game modes |
| Teacher | All above + cross-link to dashboard; "your students are playing" notifications |

This is **feature flagging within one app** — not separate builds or separate routes. The game UI is the same; capabilities expand based on auth state.

---

## 6. Backend Strategy

The paid game backend (`number-hive-complete`) already has a well-built anonymous user tracking stack:

- `Visitor` model: UUID-based identity, first-touch UTM capture, country (Cloudflare header), device fingerprint
- `TrackEvent` model: UTM fields, country, viewport context, visitorId + userId (both optional)
- `Session` model: supports anonymous sessions (visitorId only, no userId required)
- Visitor → User linkage on signup: UTM attribution preserved through conversion

**Decision under consideration: extend the paid backend rather than build a separate free-game backend.**

This means:
- The stub backend in this repo (`/backend/`) is retired
- The free game frontend calls the paid backend API directly (free-tier endpoints to be added)
- Anonymous game state, UTM tracking, country, and device data all flow into the same database
- The anonymous → class member → paid subscriber conversion funnel is a single, queryable identity chain

**UTM tracking and country capture for anonymous free players is essentially zero additional work** — the paid backend already handles this via `/api/visitor/identify` and the `CF-IPCountry` Cloudflare header.

---

## 7. Anonymous → Registered Conversion Flow

The "join a teacher's class" flow, which represents a key commercial moment, works as follows:

```
Teacher shares link: numberhive.app/join/ABC123
    ↓
Kid clicks link — downloads ~60–80kb game shell
    ↓
Anonymous session created
UTMs + country captured server-side
Play history begins (tied to visitorId cookie)
    ↓
Optional: "Save your score / join [Teacher]'s class" prompt
    ↓
Kid provides email or username
    ↓
Visitor record promoted to registered_player
Anonymous play history migrates (same visitorId → now linked to userId)
    ↓
Teacher sees student in dashboard immediately
Prior anonymous game history is visible
```

This relies on the existing `linkVisitor()` function in the paid backend, which already handles the identity migration. No new data migration logic is required.

---

## 8. Cross-Property Tracking (Cookie Architecture)

With everything under `*.numberhive.app`, a single first-party cookie set with `Domain=numberhive.app` is shared across:

- `numberhive.app` (marketing)
- `free.numberhive.app` (free game)
- `play.numberhive.app` (paid game / dashboard)

This means:
- A user's identity is consistent from marketing site through free game to paid conversion
- Google Analytics 4 can be configured with `cookie_domain: 'numberhive.app'` for automatic cross-subdomain tracking
- No cross-domain linking hacks required (those are only needed for entirely separate domains)

**Caveat:** Safari ITP caps JS-set cookie lifetimes at 7 days if set via redirect. Server-side `Set-Cookie` headers are not subject to this cap. Tracking cookies should be set server-side where possible.

---

## 9. Proposed Monorepo Structure (Under Consideration)

If the decision is made to consolidate codebases:

```
number-hive/  (new monorepo)
  apps/
    game/         ← number-hive-newvis (Phaser + Vite — players)
    dashboard/    ← new React app (teachers + staff)
  packages/
    game-engine/  ← pure JS game logic (no UI, shared)
    tracking/     ← visitor identity, UTM, analytics client (shared)
    api-client/   ← GraphQL / REST client (shared)
  backend/        ← number-hive-complete backend (unchanged)
```

**Two migration path options:**

**Option A — Evolve gradually:** Keep this repo as-is, continue shipping game features, migrate into the monorepo structure later when the dashboard app needs to be built.

**Option B — Restructure now:** Pay the structural debt upfront before two codebases diverge further.

Both are valid. Option A preserves momentum on the game. Option B avoids compounding technical debt.

---

## 10. New Game Modes (Future Scope)

The discussion noted that future games (e.g. teaching the *why* behind multiplication) fit cleanly into this architecture:

- New game modes = new Phaser scenes in the game app
- Educator analytics for those modes = new views in the dashboard app
- Shared backend captures game session data regardless of which mode was played
- The game layer and insight layer remain decoupled

---

## 11. Open Decisions

| Question | Status |
|---|---|
| Confirm domain structure: `free.numberhive.app`, `play.numberhive.app`, etc. | Superseded — see `architecture/subdomain-map.md` (2026-07-24), which records the current view: `www.numberhive.app` (WordPress marketing, live), `game.numberhive.app` (free game, not yet live), `play.numberhive.app` (education app, live), `admin.numberhive.app` (proposed, per ADR-005 in `number-hive-complete`). Still not fully finalised — `.org` references linger in some code/docs. |
| Monorepo now (Option B) vs evolve gradually (Option A) | Under discussion |
| When to retire the stub backend in this repo | Depends on backend integration decision |
| When to begin dashboard app (educator interface) build | Depends on paid game migration timeline |
| Handling PWA installs on `play.numberhive.org` during migration | Needs comms plan — existing installs break on domain move |

---

## 12. Change Log

| Date | Note |
|---|---|
| 2026-06-02 | Document created from architecture discussion session |

---

*Migrated from `number-hive-newvis/docs/platform-strategy.md` to the central NumberHive documentation repo on 2026-07-24. Pre-migration history lives in that repo's git log. See `page-inventory.md` (moved alongside this document, same date) for the supporting screen-inventory evidence referenced by this discussion.*
