# Async Arcade Architecture

**Product:** Number Hive Arcade (the public free game; `number-hive-newvis` repository; pre-launch).
**Scope of this document:** how an asynchronous player-vs-player game moves from invite to a
resolved outcome without both players ever needing to be online at once — the concurrency cap that
bounds how many such games a player can juggle, the queue that holds redemptions that arrive over
cap, how a returning player is shown which of their games need a move, how anonymous ("guest")
identity interacts with all of the above, how rating (HIQ) is applied consistently regardless of how
a game closes, and the two background sweeps that close games nobody finishes.

This document is a **migration and consolidation** of what is actually live in
`number-hive-newvis`'s own in-repo documentation and code — every claim below is cited to a source
file. Where a `number-hive-newvis` doc describes a *plan* rather than *shipped behaviour* (this
applies to `docs/opponent-move-notification-spec.md` and
`docs/NumberHive_Free_Game_Product_Tech_Spec.md`, both product/UX specs, not code), it is named as
such and not treated as proof of what runs today.

For **document lifecycles, indexes, and the ADR-003 storage contract** (the entity-level state
machine each match/challenge/invite/rating record moves through, and why each index exists), see the
companion document [`arcade-data-model.md`](./arcade-data-model.md) — this document does not repeat
that state-machine detail; it covers the *mechanisms that move a match between those states* (the cap,
the queue, the sweeps, the rating helper) and the two areas the data model document deliberately
doesn't cover: how a returning player is shown "your turn," and how guest identity survives across
sessions and devices.

---

## 1. Game & match lifecycle — the creation paths

Two functions in `backend/src/lib/gameCreation.js` are the only "seed a fresh match for two known
players" entry points remaining in the backend, per that file's own header comment (lines 1–27): the
bare cold-invite creation route (`POST /matches`, the old "Play a Friend" button's backing endpoint)
"was retired as dead code by CHG-3775 — its only caller ... had already been unused."

- **`createSeededActiveMatch`** (`gameCreation.js:56-104`) — both players already known (a standing
  link redemption, or a promoted queue entry), so the match is inserted directly at `status: 'active'`
  with both `players` seeded. No lobby/join step. `gameState.currentPlayer` is set via
  `computeStarter(connectionGameNumber)` (`gameLifecycle.js:19-21`) — a mechanical per-pair
  alternation (odd game number → player 2 starts, even → player 1), not "redeemer always starts."
  Deliberately **does not set `expiresAt`** (`gameCreation.js:45-54`): an earlier version did, which
  combined with the shared `fg_matches` TTL index to silently delete active standing-link games at day
  7 regardless of activity, before the 14-day inactivity sweep (§8) could ever run and with no closing
  record left behind. Omitting `expiresAt` makes the TTL index skip these docs entirely.
- **`createSeededPendingOffer`** (`gameCreation.js:141-180`) — only the issuer is seeded, `status:
  'pending'`, `connectionOfferFor` set to the invited opponent's `clientId`. This is what lets the
  existing `POST /matches/:matchId/join` CAS path seat the opponent on accept, the `POST
  .../decline` endpoint identity-gate a decline to just that recipient, and the roster query
  (`connectionStats.js`) surface it as an incoming offer — "converges connections-originated offers
  onto the *same* accept/decline/expiry primitive as every other kind of pending offer" per REQ-013
  (`gameCreation.js:118-123`). `gameState.currentPlayer` is fixed at `2` — the eventual joiner always
  moves first, matching the rematch-offer convention elsewhere in the codebase. `expiresAt` **is** set,
  ~7 days out (`gameCreation.js:134-140`, `SEVEN_DAYS_MS` at line 106) — an unaccepted offer must
  eventually be cleaned up, via explicit decline, the hourly offer-expiry sweep (§8), or the TTL index
  as a last-resort backstop once status is already terminal.

Both creation paths write `ledgerTracked: true` (`gameCreation.js:90-93`, `166-169`) so
`connectionStats.js`'s head-to-head ledger counts them; pre-ship docs are not backfilled, by design.

Closing a match (moves-complete, manual resign, or auto-inactivity resign) always produces the same
field shape via `gameLifecycle.js`'s `resignGame()` (lines 39-48) — `status`, `resignReason`,
`resignedBy`, `winnerClientId`, `closedAt` — regardless of which of the three triggers closed it, "so
a manual resignation and [the] auto-inactivity sweep can never drift into different shapes for
downstream ledger / HIQ consumers" (`gameLifecycle.js:24-28`). See `arcade-data-model.md` §1.1 for the
full match state diagram.

---

## 2. The concurrency cap and its enforcement points

**Source:** `backend/src/lib/concurrencyCap.js` (CHG-3763/CHG-4591).

- **What it counts.** REQ-008: the cap counts a player's **open games** — `fg_matches` docs with
  `status: 'active'`, or (defensively) `status: 'pending'` with a second player already present — not
  roster/connection count (`concurrencyCap.js:6-16`, `countOpenGames` at 64-72). A player can have many
  *connections* (people they've played) but only a small number of games actually in flight.
- **Default and override.** `DEFAULT_CAP = 5` (line 18); `getConcurrencyCap()` reads
  `MATCH_CONCURRENCY_CAP` fresh on every call, not cached at module load, so it can be overridden per
  test/environment (lines 20-29).
- **A documented bug fix baked into the counting query.** The comment at `concurrencyCap.js:38-62`
  records that `countOpenGames` used to count *any* `pending` doc regardless of player count, which
  swept in single-player `pending` invites from the retired "Play a Friend" legacy flow — invites with
  no cap check on creation, no expiry, and no cleanup. On the project's own dev DB, one real player who
  repeatedly clicked "Play a Friend" while testing had accumulated 30+ such phantom `pending` docs,
  permanently exhausting their cap with slots they could neither see (the roster is built from
  `fg_connections`, which that legacy flow never populated) nor cancel. The fix: only a `pending` doc
  with `players.1` already present counts — a state the current schema cannot actually produce (the
  join handler seats player 2 and flips to `active` in the same atomic `findOneAndUpdate`), so the
  check is a defensive guard against that invariant ever breaking, not a live code path today.
- **Enforcement points.** `hasFreeSlot()` (73-83) is the free/blocked check; `getConcurrencyCap` is
  only ever checked against the **issuer** of a standing link at redemption time
  (`concurrencyCap.js:12-16`, enforced in `routes/links.js`'s redeem handler) — REQ-008's literal
  wording ("auto-starts a game if the issuer has a free slot, else queues FIFO") constrains only the
  issuer's count, not the redeemer's. `enforceConcurrencyCapOrReply()` (112-126) is the shared
  check-then-409 wrapper — extracted (CHG-4591) from four previously-duplicated inline copies at
  `routes/connections.js`'s start-game handler and `routes/matches.js`'s moves-auto-join/join/rematch
  handlers. It checks both `clientId` and `opponentClientId` in parallel and returns `{error,
  blockedSide}` on the first violation, `null` if both sides are free. It does not catch DB errors
  (they propagate) and adds no new input validation beyond the four original inline sites.
- **Read-only visibility.** `GET /connections/slots?clientId=` (`routes/connections.js:193-211`) wraps
  `countOpenGames()` read-only, for the "Games open: X / Y" UI element.

---

## 3. The queue-on-cap invite flow

**Source:** `backend/src/lib/pendingGameQueue.js` (CHG-3763). FIFO queue, collection
`fg_pending_games`, for standing-link redemptions arriving while the issuer is already at cap
(REQ-008's second branch). **No expiry on a queued row** in this build (documented as "spec
Assumption #3," line 11) — a redemption waits indefinitely until the issuer frees a slot.

- **`enqueue()`** (lines 21-33) — persists `{issuerClientId, redeemerClientId, redeemerNickname,
  sourceLinkId, queuedAt}`.
- **`promoteQueueForPlayer(fastify, clientId)`** (lines 53-100) — called whenever a game closes and
  might free a slot for `clientId`. Pops **at most one** queued row per call (`findOneAndDelete`,
  oldest first, line 63-66) — a single freed slot promotes a single queued game, never more, even if
  several rows are waiting. Best-effort: errors are logged and swallowed (fire-and-forget from every
  call site), matching the existing `sendTurnNotification` convention. On promotion it calls
  `createSeededActiveMatch()` (§1) — the promoted game starts immediately, both players seated, no
  further join step — then fires `sendGameStartedNotification()` (lines 84-93), a push notification.

**Every game-closing path must call `promoteQueueForPlayer` or a freed slot leaks.** Confirmed call
sites, one per closing path:

| Closing path | Call site |
|---|---|
| Moves-complete | `routes/matches.js:297` (both players) |
| Recipient declines / issuer cancels an offer | `routes/matches.js:562` (issuer only — the recipient never held a slot) |
| Manual resign | `routes/matches.js:649` (both players) |
| A third `routes/matches.js` site | `routes/matches.js:1034` |
| Auto-inactivity resign (sweep) | `autoResignSweep.js:135` (both players) |
| Offer expiry (sweep) | `offerExpirySweep.js:104` (issuer only, `players[0]`) |

**There is no dedicated in-app "your queued game just started" banner.** Beyond the push
notification, the newly-active match simply appears the next time the roster (`GET /connections`, §4)
or match-history (`GET /matches`) endpoint is polled, since it is now an ordinary `fg_matches` doc with
`status: 'active'`.

```mermaid
sequenceDiagram
    participant R as Redeemer
    participant API as routes/links.js (redeem)
    participant Cap as concurrencyCap.js
    participant Q as fg_pending_games
    participant M as fg_matches

    R->>API: redeem standing link
    API->>Cap: hasFreeSlot(issuer)?
    alt issuer has a free slot
        API->>M: createSeededActiveMatch() — status active, both players seated
        API-->>R: match starts immediately
    else issuer is at cap
        API->>Q: enqueue({issuerClientId, redeemerClientId, ...})
        API-->>R: queued (no game yet)
    end

    Note over M: later, some other game closes for the issuer
    M->>Q: promoteQueueForPlayer(issuerClientId)
    Q->>Q: findOneAndDelete oldest row for issuer
    Q->>M: createSeededActiveMatch() — promoted game starts
    M-->>R: push notification (sendGameStartedNotification); game visible next poll
```

---

## 4. "Your Turn" surfacing

Whose turn it is is **derived, not a stored per-player flag.** `fg_matches` stores
`gameState.currentPlayer` (1 or 2); the clientId-level "whose turn" value is resolved server-side
against `players[currentPlayer-1].clientId`.

- **`connectionStats.js:156-167`**, inside `buildRosterEntry()` — for every open (`pending`/`active`)
  match between a pair, computes `whoseTurnClientId` from `currentPlayer` and attaches it to that
  connection's `activeGames` entry (matchId, status, shareUrl, whoseTurnClientId).
- **`GET /connections?clientId=`** (`routes/connections.js:134-175`) is the endpoint the frontend
  actually uses for turn surfacing. Its own doc comment (113-133) states explicitly: flat-mapping the
  per-connection `activeGames` array client-side "is REQ-017's cross-opponent 'all current games'
  list, so no separate endpoint is needed for that." It is built fresh per request from
  `fg_connections` (roster edges) plus a live `fg_matches` query per opponent (`connectionStats.js`'s
  `fetchPairMatches`, lines 51-59) — **not** a cached/materialized roster.
- **`GET /connections/:opponentClientId?clientId=`** (`routes/connections.js:227-260`) — same
  `whoseTurnClientId` shape, single-opponent detail.
- **`GET /matches?clientId=`** (`routes/matches.js:1050-1095`) is a separate, older "match history"
  list (last 20 by `createdAt`) that returns `opponentNickname`, `status`, `result`, `createdAt`,
  `lastMoveAt` — **it does not expose `whoseTurnClientId` or any turn indicator.** The turn-surfacing
  UI (`GamesPanel`) uses `GET /connections`, not this endpoint; any other consumer of `GET /matches`
  expecting turn state will not get it.
- **`GET /matches/:matchId`** performs **no `clientId` ownership check** (`routes/matches.js:94-178`)
  — anyone holding the exact match URL can view its current state. Turn ownership is enforced only at
  the *mutating* endpoint: `POST /matches/:matchId/moves` rejects with "not your turn" if
  `match.gameState.currentPlayer !== playerNum` (`matches.js:254-257`).
- **Frontend.** `src/ui/GamesPanel.ts`, hosted by `src/scenes/GamesScene.ts:33-68`
  (`showGamesPanel`/`dismissGamesPanel`), backed entirely by `src/links/ConnectionsClient.ts`. Turn
  label logic is a two-line pure function, `turnLabel()` (`GamesPanel.ts:615-618`): returns `'Your
  turn'` if `whoseTurnClientId === clientId`, else `'Their turn'`, else `'Unknown'`. Rendered per row
  at `GamesPanel.ts:897`. The cross-opponent "All Current Games" section (`GamesPanel.ts:713-733`)
  flat-maps `activeGames` across every roster connection, matching the `connections.js` doc comment
  above. `loadRosterView()` (`GamesPanel.ts:641-669`) fetches `GET /connections` and `GET
  /connections/slots` in parallel. No auth header is sent on any of these calls — `clientId` travels as
  a query param/body field (`ConnectionsClient.ts:137-264`).
- **No push/badge "unread turn count" exists.** The main-menu scene (`ModeScene.ts`) has no your-turn
  badge tied to games waiting on the player — only a "My Games" entry point and separate push
  notifications (below).
- **Real-time delivery, supporting but distinct from the turn list itself:** long-poll on `GET
  /matches/:matchId?longpoll=1&...` (`matches.js:57-178`), woken by `notifyMatchChanged(matchId)` on
  every state-changing write; and push notifications from `lib/matchNotifications.js` —
  `sendTurnNotification()` after a move is submitted (`matches.js:286`), `sendResignNotification()`
  after a resign (`matches.js:655`), suppressed if the recipient already has the match open via
  long-poll presence (`lib/matchPresence.js`) or the pair is blocked (`lib/blockCheck.js`).

**Aspirational, not confirmed built:** `docs/opponent-move-notification-spec.md` describes an intended
badge-count system (app icon badge, tab title badge, per-game-row badge) and batched "your turn in 3
games" notifications. Its own "Implementation Status" section marks badge count as **not yet
implemented** — consistent with the "no push/badge unread-turn-count" finding above. This spec
document should not be read as proof any of that exists; it is a UX design document, not shipped code.

---

## 5. Guest versus account behaviour

Number Hive Arcade is **anonymous by default** — no signup is required to play.

- **Identity store.** `src/analytics/Identity.ts`'s `IdentityService` (constructor,
  `Identity.ts:264-308`). Primary store is `anon_client_id` in `localStorage`
  (`LS_CLIENT_KEY`). On first boot with no stored id: `generateUUID()` (80-91,
  `crypto.randomUUID()` with a manual fallback), persisted to `localStorage` immediately, then
  asynchronously reconciled against an IndexedDB backup (`_reconcileWithIdb()`, 341-352 — database
  `anon_identity`, store `kv`, key `client_id`), which survives some localStorage clears and is
  preferred on reconciliation, or seeded from the fresh id if empty. No cookies, no server-issued
  session for anonymous play.
- **Backend routes take a caller-supplied `clientId`, not a session.** Every PvP route
  (`matches.js`, `connections.js`) accepts `clientId` as a query param/body field and authorizes
  purely by membership check against `match.players` (e.g. `matches.js:220, 365, 483, 605, 712, 931`)
  — no `accountId` reference exists anywhere in `matches.js` or `connections.js`.
- **`POST /visitors`** (`routes/visitors.js:11-60+`) registers/looks up an `fg_visitors` doc keyed by
  `clientId`, assigning a generated nickname — the closest thing to a "profile" an anonymous player
  has, still purely clientId-keyed.

**A real account system exists in this repo — but it is not wired into PvP arcade matches.**

- `routes/auth.js` implements a genuine Google OAuth 2.0 Authorization Code flow (`GET /auth/google`,
  `GET /auth/google/callback`), upserting `fg_accounts` and issuing a session JWT via URL fragment
  (`auth.js:246-251`).
- `routes/accounts.js`'s `GET /accounts/me` (gated by `fastify.authenticate`, line 21) returns
  `accountId`, `nickname`, `roles`, `hiveIq`, `hiveIqGamesPlayed`, `progression`.
- On sign-in, the current `clientId` is added to `fg_accounts.clientIds` (capped at 50,
  `auth.js:39, 155-167`) — "anonymous-account stitching," used to (a) let a new account inherit the
  visitor's existing nickname (`auth.js:211-232`, citing "ADR-002 'anonymous play history stitched to
  new account'") and (b) track which devices have signed into the account. Indexed at
  `db/indexes.js:88`.
- **This account system is scoped to solo HiQ rating/progression, not PvP.** It is consumed by
  `routes/hiq.js`, `routes/gameSnapshot.js`, `routes/progression.js`, and the frontend's
  `src/auth/AccountService.ts` (syncs `PlayerProfile`/`HiqProfile`/`Progression`/rival cards across
  devices for a signed-in account) and `ModeScene.ts:1067-1090`'s `_tryRemoteResume()` — gated on
  `AccountService.isSignedIn()`, calling `GameSnapshotClient.getSnapshot()` for the **solo/vs-AI**
  game, not PvP.
- **No code path maps `fg_accounts`/`accountId`/`clientIds` onto `fg_matches` or `fg_connections`.**
  `matches.js`, `connections.js`, `connectionStats.js`, `hiqPvpClose.js` contain no `accountId` or
  `fg_accounts` reference. PvP rating (`fg_pvp_hiq`, `getOrCreatePvpHiq()` in `hiqPvpClose.js:47-53`)
  is itself a separate, clientId-keyed collection, distinct from the account's `hiveIq` field.
- **Practical consequence:** signing in with Google does not recover or re-link PvP arcade games onto
  a new device. If `anon_client_id` (and its IndexedDB backup) is lost, the player is, for PvP-match
  purposes, a new player — the account system was built for solo HiQ progression sync, and nothing
  stitches `fg_matches` history to `accountId`. This is drawn from the absence of any such linkage in
  the live route code (not from a doc), flagged here explicitly because both the opposite
  assumptions — "no account exists" and "signing in protects your PvP games" — are wrong.
- **Recovery mechanics for a cleared/switched device, PvP specifically.** `GET /matches/:matchId`
  performs no ownership check — anyone holding the exact match URL can *view* it regardless of
  `clientId`. **Mutating** as that player (move/resign/rematch/decline) requires `clientId` to already
  be present in `match.players` — a cleared/new `clientId` cannot act as the original player even with
  the link, short of the auto-join-as-new-player-2 path (`matches.js:225-244`), which only applies to
  a still-`pending`, one-player match. There is no password reset / account-recovery / "restore by
  email" flow for anonymous identity. The only continuity mechanisms are: (1) `anon_client_id`
  surviving in `localStorage`, (2) the IndexedDB backup surviving a localStorage-only clear, or (3)
  having signed into a Google account on the *same* device beforehand — which, per the point above,
  only protects the solo/HiQ profile, not PvP matches.

**Governing convention — ADR-003 (`number-hive-complete`).** Rule 1 of
`docs/adr/003-migration-safety.md` (lines 52-65) declares `anon_client_id`, `anon_session_id`,
`buzz_profile`, `game_snapshot`, `fg_nickname`, `hiq_profile` "permanent contracts. Never rename or
remove without a migration script that runs on page load." Its key table (lines 23-35) names
`localStorage: anon_client_id` explicitly, with migration risk "Key rename breaks identity
continuity." The account service's own session token key, `fg_session_token`
(`AccountService.ts:39`), is documented in that file (lines 35-38) as "a NEW, additive localStorage
key — not a rename of any key on the ADR-003 frozen-key list" — the live code is deliberately
ADR-003-compliant, not merely coincidentally consistent with it. ADR-003 is status **Accepted**,
dated 2026-06-27.

**Aspirational vs. built.** `NumberHive_Free_Game_Product_Tech_Spec.md` ("no install, no account,
playing in seconds," line 193; "anonymous `client_id`," line 308) accurately describes the PvP
arcade's live behaviour. It is silent on the account system entirely, which does exist and does ship
(`auth.js`, `accounts.js`, `AccountService.ts`, a "Sign in" UI element in `ModeScene.ts`) — just for a
different subsystem. A reader of the product spec alone would not know either that an account system
exists, or that it doesn't cover PvP.

---

## 6. Rating (HIQ) treatment on all close paths

**Source:** `backend/src/lib/hiqPvpClose.js` (CHG-3770, REQ-015). `applyPvpHiqOnClose(fastify, match,
trigger)` is documented as "the single shared helper that must be called at EVERY match-closing call
site ... so that an auto-resigned game gets byte-identical HIQ treatment to a manual resignation —
per Q15, this is what closes the 'go idle to dodge a rating hit' exploit" (lines 6-11). Server-side
only, symmetric two-player Elo update (`lib/hiqPvp.js`) — deliberately not the client-computed vs-AI
HIQ model, "since PvP results must never trust a client-submitted number."

**Idempotency.** `fg_hiq_audit` has a unique index on `matchId`. The audit doc is inserted **first**,
before any rating mutation; a duplicate-key error (Mongo code 11000) means this match's HIQ was
already applied and the call is a no-op (lines 18-22, 128-136). Only a successful insert proceeds to
mutate `fg_pvp_hiq`, via atomic `$inc` (commutative, order-independent — two different matches for the
same player closing near-simultaneously both still apply, rather than one clobbering the other; lines
28-37, 142-151).

**Guard conditions** (`applyPvpHiqOnClose`, lines 88-97): silently no-ops unless `match.status` is
`'complete'` or `'resigned'` **and** `match.players.length === 2` with both clientIds present — HIQ is
only defined for a determinate 2-player outcome.

**Confirmed call sites — every path that can produce a `complete`/`resigned` match calls this
helper:**

| Trigger label | Closing event | Call site |
|---|---|---|
| `'moves_complete'` | A move completes the game | `routes/matches.js:308` |
| `'manual'` | Player-initiated resign | `routes/matches.js:661` |
| `'auto_inactivity'` | 14-day inactivity sweep resigns the on-move player | `autoResignSweep.js:144` (awaited, not fire-and-forget — "so the audit write has actually landed before this sweep pass considers the match resolved") |

**Paths that do *not* call it, correctly.** A declined offer (`status: 'declined'`/`'cancelled'`,
`routes/matches.js:519`) and an expired pending offer (`status: 'expired'`, `offerExpirySweep.js:76-85`)
both terminate a match that never reached two seated players playing a determinate game — the guard
condition above (`status` must be `complete`/`resigned`) already excludes them, and neither call site
invokes `applyPvpHiqOnClose`. This is consistent, not a gap: there is no result to rate.

On success, `match.hiqDelta`/`match.hiqRatingAfter` are set in-memory (so callers returning `match`
directly to the client get the fields for free) and best-effort persisted onto the stored `fg_matches`
doc "for reconnect-resilience (a client that reloads and re-fetches the match later must still see the
outcome)" (lines 75-81). The function never throws — errors are logged and swallowed, matching the
fail-open convention of `promoteQueueForPlayer`/`sendResignNotification` at the same call sites.

---

## 7. The inactivity sweeps

Both sweeps share one structural pattern, explicitly documented as intentional in
`offerExpirySweep.js:1-11` ("mirrors `lib/autoResignSweep.js`'s structural pattern exactly"): an
in-process `setInterval`, started once at boot (catching up any backlog immediately) and re-run on a
configurable interval thereafter; every transition is an atomic `findOneAndUpdate` whose filter
re-derives the exact staleness condition that made a doc a candidate, so overlapping sweep runs or a
real concurrent user action simply lose the race and leave the doc untouched — "at most one write per
match ever lands." Both are justified as single-instance-only ("this backend runs as a single Fastify
instance, no clustering — see ADR-004 ... If a future change moves to multiple instances, this file is
where a lock ... would need to be added — not before," `autoResignSweep.js:13-19`).

| | `autoResignSweep.js` | `offerExpirySweep.js` |
|---|---|---|
| Governs | `status: 'active'` matches (REQ-018 part 2) | `status: 'pending'` offers past `expiresAt` (REQ-007) |
| Threshold | 14 days (`FOURTEEN_DAYS_MS`), override via `AUTO_RESIGN_INACTIVITY_MS` | Set at creation (`expiresAt`, ~7 days, `gameCreation.js`) |
| Interval | 1 hour default, override `AUTO_RESIGN_SWEEP_INTERVAL_MS` | 1 hour default, override `OFFER_EXPIRY_SWEEP_INTERVAL_MS` |
| "Stale" measured from | `lastMoveAt`, or `createdAt` if no move was ever made (lines 71-75) | `expiresAt` directly |
| Candidate query | `runSweepOnce`, lines 84-92 | `runSweepOnce`, lines 63-65 |
| CAS re-derivation | Filter re-checks `lastMoveAt`/`createdAt` against the same cutoff (lines 108-111) | Filter re-checks `status: 'pending'` (lines 74-75) |
| On transition | `resignGame()` closing shape, `resignedBy` = the on-move player ("their inaction is what's blocking the game, per Q15," lines 6-11) | Closing shape set inline (`status: 'expired'`, `resignReason: 'expired'`, `resignedBy: null`, `winnerClientId: null`) |
| Frees concurrency slot | Yes — `promoteQueueForPlayer` for **both** players (line 135) | Yes — `promoteQueueForPlayer` for the issuer only, `players[0]` (line 104) |
| Applies HIQ | Yes — `applyPvpHiqOnClose(..., 'auto_inactivity')`, awaited (line 144) | No — an unaccepted offer has no determinate 2-player result (§6) |
| Notifies | `sendResignNotification` (fire-and-forget; "a 14-day-stale opponent's win would never surface a push" otherwise, lines 146-156) | None documented |
| Also releases | — | The original match's `rematchMatchId` claim, if this was a rematch offer (lines 90-100) — "the pre-existing behaviour relied solely on the shared TTL index," which "never released the *original* match's `rematchMatchId` claim (permanently blocking future rematch attempts)" before this sweep existed (lines 12-24) |

Both sweeps' interval handles are `unref()`'d so they never keep the process alive on their own
(`autoResignSweep.js:184`, `offerExpirySweep.js:132`).

---

## 8. How this document supports the Launch Brief's Arcade "must verify" list

The CEO's Launch Brief (25 August 2026) lists, under Arcade "must verify": *challenge links get
recipients into the human match with minimal friction; async matches persist reliably across
sessions; multiple concurrent async games work properly; Your Turn accurately surfaces the correct
games when a player returns; guest/account behaviour does not cause matches to be lost; completed
games and Altitude changes resolve correctly.* This document maps directly onto that list:

| Must-verify item | Where this document addresses it |
|---|---|
| Challenge links → human match, minimal friction | §1 (creation paths), §3 (queue-on-cap sequence diagram) |
| Async matches persist reliably across sessions | §1 (`expiresAt` omission on active matches), §5 (identity persistence, ADR-003) |
| Multiple concurrent async games work properly | §2 (concurrency cap), §3 (queue) |
| Your Turn surfaces correctly on return | §4 |
| Guest/account behaviour does not lose matches | §5 |
| Completed games / Altitude resolve correctly | §6 (rating/HIQ = "Altitude" in product language — see `arcade-data-model.md` §1.4 for the audit/idempotency detail) |

None of these are *verified working end-to-end* by this document — it establishes what the code does
by design and citation; live functional verification (does it actually behave this way under test) is
explicitly out of scope, per the ground rules governing this document set.

---

## 9. What this document does not cover

- The full entity/field reference and document lifecycle state diagrams — see `arcade-data-model.md`.
- The challenge (seeded solo-vs-AI) lifecycle in detail — see `arcade-data-model.md` §1.3; challenges
  do not participate in the concurrency cap, the queue, or PvP HIQ (§6 above).
- Analytics/event tracking for any of the flows described here — see `handover/analytics-inventory.md`.
- The referral-link chain-attribution design sketch in `docs/referral-link-system-design.md` — that
  document is explicitly "design-capture only. Not scheduled, not build-ready," tracked as CHG-4738
  (idea priority, backlog), and describes no shipped mechanism; it is not part of this architecture.

---

*Sources cited: `backend/src/lib/gameCreation.js`, `gameLifecycle.js`, `concurrencyCap.js`,
`pendingGameQueue.js`, `hiqPvpClose.js`, `hiqPvp.js`, `autoResignSweep.js`, `offerExpirySweep.js`,
`connectionStats.js`, `matchNotifications.js`, `matchPresence.js`, `blockCheck.js`,
`routes/matches.js`, `routes/connections.js`, `routes/links.js`, `routes/visitors.js`,
`routes/auth.js`, `routes/accounts.js`, `db/indexes.js`; frontend `src/analytics/Identity.ts`,
`src/auth/AccountService.ts`, `src/ui/GamesPanel.ts`, `src/scenes/GamesScene.ts`,
`src/scenes/ModeScene.ts`, `src/links/ConnectionsClient.ts`; `number-hive-complete/docs/adr/
003-migration-safety.md`; `number-hive-newvis/docs/opponent-move-notification-spec.md` and
`NumberHive_Free_Game_Product_Tech_Spec.md` (cited only where explicitly marked as spec/design, not
shipped behaviour). Compiled 2026-08-31.*
