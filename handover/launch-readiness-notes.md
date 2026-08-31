# Launch Readiness Notes — Arcade & Education

**Purpose:** a factual map only — for each "must do before launch" and "must verify" item in
the CEO's Launch Brief (25 August 2026, Email A) and follow-up (31 August 2026, Email B), where
in the code that area currently lives and what the code currently shows exists. **No estimates,
no recommendations, no readiness judgements.** Where nothing exists yet for an item, that is
stated plainly rather than left blank.

This document does not repeat ground covered in full elsewhere — it points at
[`async-arcade-architecture.md`](async-arcade-architecture.md) for match/rating/concurrency
mechanics, [`arcade-data-model.md`](arcade-data-model.md) for the data layer, and
[`analytics-inventory.md`](analytics-inventory.md) for all event-tracking questions across both
products, rather than re-deriving any of that here.

---

## 1. Number Hive Arcade (`number-hive-newvis`)

### Must do before launch

| Brief item | Where it lives | What the code currently shows |
|---|---|---|
| Replace Player Bee with the attached final SVG | `src/ui/PlayerMascotData.ts` (style definition, lines 44–102), instantiated in `src/scenes/ModeScene.ts:3929` (`buildProfileRoundel()`) | The player's own mascot is rendered procedurally via Phaser Graphics (`BeeMascot`), not from an SVG asset — style is `getDefaultPlayerMascotStyle()` ("Classic": amber body, player-coloured scarf accent). The style definition has a reserved `rivalId` field intended for future SVG rig art, but the current default path uses native Graphics only. The two attached files (`player-bee-exploded.svg`, `player-bee-layered.svg`) are not present anywhere in the repo and are not referenced by any code. |
| Make Player Bee clearly look clickable where interactive | `src/scenes/ModeScene.ts:3905–3952` (`buildProfileRoundel()`) | Player Bee sits inside a circular, interactive profile roundel (cyan ring, 44px diameter, top-right). Current affordances: `pointer` cursor on hover (line 3940), an 8% scale-up tween on pointerover reverting on pointerout (lines 3941, 3945), a `Phaser.Geom.Circle` hit-area (line 3935, 22px radius), and a tap handler opening the StatsPanel (lines 3947–3948). This is the only place Player Bee is currently interactive — it is not clickable on `GameScene`. |
| New visitors land directly into the Bumble game — no mode-selection/signup first | `src/main.ts:45–50` (scene registration), `src/scenes/ModeScene.ts:919–1230` (`create()`) | `ModeScene` is the first registered scene and auto-starts on load (no title/mode-select screen precedes it, no signup gate). A brand-new visitor with no existing matches sees a button grid ("Play Locally", "Play a Friend", "Quick Match") and can start a game immediately from any of them — none of the three currently drop the player directly into a Bumble game without a button tap first. |
| Simplify the main Arcade screen: top teacher/school CTA only, remove duplicated lower CTA | `src/ui/EducatorCTA.ts`, mounted via `EducatorCTAService.mountStrip()` at `src/scenes/ModeScene.ts:1294` | Exactly one teacher/school CTA currently exists — a persistent footer strip (`EducatorCTA.ts:55–80`) linking to `https://play.numberhive.org`. No second/duplicated CTA was found in the current code. |
| Altitude should not change when playing locally | `src/scoring/HiqProfile.ts:113–201` (`applyHiqGameResult()`), `src/scenes/GameScene.ts:5158–5215` (`_trackGameFinished()`) | `vs_ai` games (the "Play Locally" mode) currently **do** call `applyHiqGameResult()` and change the local Altitude value (`GameScene.ts:5174`) — the same is true for `vs_human` local play (lines 5168–5169, against a 'human_opponent' anchor rating). `pvp_challenge` (rated PvP) games do not compute Altitude client-side at all — they only read back the server-computed `hiqDelta`/`hiqRatingAfter` from the match document (lines 5264–5265, 5242–5250). The current code does not distinguish "local" from "rated" for the purpose of gating Altitude changes — both `vs_ai` and `vs_human` currently affect it. |
| Returning users land on their async games: Your Turn most prominent, opponent-waiting games visible, clear Challenge someone action | `src/ui/GamesPanel.ts:640–743`, `src/scenes/GamesScene.ts:33–69` | Returning users with existing matches reach `GamesPanel` via the "Play a Friend" button (not automatically on load). The panel shows, in order: a "My Games" header, a "My Play-Me Link" button, then (if the roster is non-empty) a per-opponent roster list — nickname, W/L/D record, active-game count per opponent, and a "Games open" concurrency counter (lines 631–637) — with drill-down into a per-opponent detail view showing active games with a turn indicator ("Your turn" vs "Their turn") and historic games. Roster data comes from `ConnectionsClient.listConnections()` (line 656), which returns a `RosterEntry[]`; `GamesPanel.ts` does not itself reorder or group by turn state — see [`async-arcade-architecture.md`](async-arcade-architecture.md) §4 for the full "Your Turn" derivation mechanics this panel consumes. No explicit "Your Turn games most prominent" sort/grouping was found in this file. |
| Simplify post-game screen: remove Challenge Score/breakdown, keep Altitude where relevant, ensure Next up shows correct next Rival | `src/scoring/PostGameCard.ts` (DOM overlay), data assembled in `src/scenes/GameScene.ts:5158–5433` | **Challenge Score:** currently shown for `vs_ai` games — `PostGameCard.ts:67` (`challengeScore?: ChallengeScoreResult`), rendered as a titled card with an itemized score breakdown (lines 265–289, `.nh-postgame-challengescore` / `-breakdown` CSS classes). Not currently removed. **Altitude:** shown for `vs_ai`/`vs_human` via `hiq?: PostGameCardHiqMovement` (lines 46–47) as `Altitude: [before] → [after] (+/-delta)`; omitted entirely for `pvp_challenge` (server-side only, consistent with the row above). **Next up:** already implemented — `GameScene.ts:5228–5230` calls `getNextRivalToPass()` to resolve the next rival in the ladder, exposed as `nextRivalName?: string` on `PostGameCardData` (lines 69–80) and rendered as a "Next up: <name> →" button when populated; the resolution rule (only set when the player beat someone other than their actual next-to-pass rival) is commented at `GameScene.ts:5228`. |

### Must verify before launch

| Brief item | Where it lives | What the code shows |
|---|---|---|
| Challenge links get recipients into the human match with minimal friction | Queue-on-cap invite flow, `routes/links.js` → `concurrencyCap.js` | Full sequence diagram and code citations in [`async-arcade-architecture.md`](async-arcade-architecture.md) §3. |
| Async matches persist reliably across sessions | `Identity.ts` (`localStorage` + IndexedDB backup), ADR-003 frozen key set | [`async-arcade-architecture.md`](async-arcade-architecture.md) §5. |
| Multiple concurrent async games work properly | `concurrencyCap.js`, `DEFAULT_CAP=5`, `MATCH_CONCURRENCY_CAP` override | [`async-arcade-architecture.md`](async-arcade-architecture.md) §2. |
| Your Turn accurately surfaces the correct games when a player returns | `connectionStats.js`'s `buildRosterEntry()`, `GET /connections` | [`async-arcade-architecture.md`](async-arcade-architecture.md) §4 — note the gap flagged in the must-do row above: `GamesPanel.ts` itself does not reorder by turn state, it consumes whatever order the roster endpoint returns. |
| Guest/account behaviour does not cause matches to be lost | Anonymous `clientId` identity vs. separate Google OAuth account system (not PvP-linked) | [`async-arcade-architecture.md`](async-arcade-architecture.md) §5. |
| Completed games and Altitude changes resolve correctly | `hiqPvpClose.js`'s `applyPvpHiqOnClose()`, 3 confirmed closing-trigger call sites, `fg_hiq_audit` idempotency | [`async-arcade-architecture.md`](async-arcade-architecture.md) §6. |
| Analytics/event tracking is working end-to-end | — | [`analytics-inventory.md`](analytics-inventory.md) — full event inventory and funnel-by-funnel comparison against this exact "at minimum" list. |

---

## 2. Number Hive Education (`number-hive-complete`)

### Must do before launch

| Brief item | Where it lives | What the code currently shows |
|---|---|---|
| Replace the sign-in-only landing page with something along the mockup lines | `frontend/src/navigators/RootNavigator.tsx:409–420` | When `!isSignedIn`, the navigator renders `SignIn` as the sole initial screen — there is no landing/marketing page component today, and no route for one in `RootStackParamList` (lines 75–156). A `Splash.tsx` component (`frontend/src/components/screens/Splash.tsx`) exists — a 2-second animated loading transition — which could serve as an interstitial, but is not a landing page. |
| Keep as much of the existing visual styling/bones as practical | `frontend/tailwind.config.js:18–22`, `frontend/src/css/`, `frontend/src/constants/textStyles`, `frontend/src/constants/colors` | Styling is React Native `StyleSheet.create()` throughout the main app (cross-platform); Tailwind CSS is scoped only to `src/components/screens/AdminPanel/**` per `tailwind.config.js`. Base CSS files (`AppStyles.css`, `TextStyles.css`, `admin.css`) exist but are minimal. Text-style and colour constants are centralised in `src/constants/`. |
| Primary message/CTA structure (Build a culture of mathematics / Try the Demo / Sign Up / Sign In retained) | `frontend/src/components/screens/ChooseUserType/ChooseUserType.tsx:36–65`, `frontend/src/components/screens/Login/Login.tsx:120–121` | No landing-page copy exists anywhere in the repo. `ChooseUserType` currently shows "Choose User Type" / "Who are you signing up as?" with Student/Teacher/Admin options and a sign-in link (line 65). `Login.tsx` has a hardcoded "Sign In" title. `MenuButton` (`frontend/src/components/helperComponents/MenuButton`) is the existing reusable primary-CTA button component. |
| Communicate Play / Learn / Understand | Searched `frontend/src/components/screens/Lobby/`, `Journey/`, `Gym/` | No copy referencing "Play", "Learn", "Understand" as a named three-part structure was found anywhere in the repo. The app does have a game-mode hub (`Lobby.tsx`), a `Journey/` assessment area, and a `Gym/` practice/assessment area, but no consolidated narrative copy tying them to this framing exists today. |
| Retain social proof (150+ countries / millions of games played) | Repo-wide search for "countries", "played", "proof" | No matches found — this copy does not exist anywhere in the current codebase (main app or `AdminPanel`). |
| Demo should finish with a path into the existing free-trial/signup flow | `frontend/public-demo/index.html`, `public-demo-v2/index.html`, `frontend/src/components/screens/AdminPanel/demo-leads/DemoLeadsList.tsx`, `frontend/src/scripts/copyDemo.js` | The demo exists as two standalone, self-contained static HTML prototypes (~147KB / ~157KB, client-side game logic only, no backend calls), copied into the build output by `copyDemo.js` (per `buildWeb.ts:11`'s comment). Demo-lead capture exists on the admin side (`DemoLeadsList.tsx`, plus `useAdminDemoLeads.query`/`useAdminResendDemoEmail.mutation` GraphQL operations). No code-level navigation currently routes from demo completion into the app's signup flow — the demo is not integrated into `RootNavigator`. |

### Must verify

| Brief item | Where it lives | What the code shows |
|---|---|---|
| Education landing page works well across desktop/mobile | — | No landing page component exists yet (see must-do row above) — nothing to verify responsiveness of today. Responsive plumbing used elsewhere in the app (`useWindowDimensions`, e.g. `Lobby.tsx:48–50`'s `width < 1251` breakpoint) would be the pattern any new page would follow. |
| Demo works reliably | `frontend/public-demo/index.html` | Self-contained playable prototype (dual pads, "change one factor" mechanic per inline comments); all logic is client-side JavaScript, no backend calls. No automated tests were found for it. |
| Demo CTA correctly leads into signup/free trial | — | No such transition exists in code today (see must-do row above). |
| Teacher signup works | `frontend/src/components/screens/SignUp/SignUp.tsx`, `backend/src/graphql/user/auth/auth.resolver.ts:35–73` | `SignUp.tsx` requests a signup code (`useRequestSignUpCodeMutation`, lines 70–74) then calls `useSignUpMutation` (lines 57–68) with teacher email/password/signup code; the backend `signup` mutation accepts `SignupArgs` (userType, name, email, password, signUpCode). On success, `MB_accessTokenUtils.setAccessToken()` stores the token and the app navigates to `Lobby` (line 101). Client-side validation exists (password length, email pattern, lines 109–125). |
| Hive creation works | `frontend/src/components/screens/CreateHive/CreateHive.tsx`, `backend/src/graphql/hive/create-hive/create-hive.resolver.ts:11–14` | `CreateHive.tsx` takes hive size/subscription type as route params, collects a hive name, and calls `useCreateHiveMutation`. If `envs.BYPASS_PAYMENT` is set, it navigates directly to `MyHiveData` (line 95); otherwise `handlePay()` opens a Stripe checkout window, with `stripeRedirectCallback` (lines 54–68) handling the return and navigating to `MyHives` on success. |
| Hive Key/student join flow works | `frontend/src/components/screens/SignUp/SignUp.tsx` (student branch, line 42/46, hive-key validation lines 98–102), `frontend/src/components/screens/JoinHive/JoinHive.tsx`, `backend/src/graphql/hive/join-hive/join-hive.resolver.ts:9–12` | Two entry points: (1) at signup time, a student supplies a 6-character hive key inline; (2) post-signup, `JoinHive.tsx` collects a 6-char code and calls `useJoinHiveMutation`, navigating to `MyHives` with `shouldRefetch: true` on success (line 48). Both call the same backend `joinHive` mutation. |
| Existing teacher Sign In works correctly | `frontend/src/components/screens/Login/Login.tsx`, `backend/src/graphql/user/auth/auth.resolver.ts:17–26` | `Login.tsx` collects email/username + password, calls `useLoginMutation` with a `visitorId` (lines 50–51); backend `login` mutation returns a `UserTokenPair` including `sessionId`. Client-side validation exists (lines 57–74). On success, token/refresh-token are stored and the app navigates to `Lobby` (line 101). |
| Analytics let us follow: Education landing → Demo → Demo completion → signup/free trial → Hive creation | — | [`analytics-inventory.md`](analytics-inventory.md) — full event inventory and funnel-by-funnel comparison against this exact five-stage list. |

---

## What this document doesn't cover

- Any judgement on whether an item is "done enough" to ship, how long remaining work would
  take, or which order to tackle it in — that's explicitly out of scope per the brief that
  commissioned this document.
- The two attached image assets referenced in the brief (Education landing mockup PNG, Arcade
  notes JPGs, final Player Bee SVGs) — these were email attachments, not part of any repository,
  and are not reproduced or assessed here.
- Anything already covered in full elsewhere: match/rating/concurrency mechanics
  ([`async-arcade-architecture.md`](async-arcade-architecture.md)), the Arcade data model
  ([`arcade-data-model.md`](arcade-data-model.md)), and all analytics/event-tracking questions
  for both products ([`analytics-inventory.md`](analytics-inventory.md)).

---

*Compiled 2026-08-31 against the live `number-hive-newvis` and `number-hive-complete` repos,
directly ahead of the Dave handover session, per the CEO's Launch Brief (25 August 2026) and
follow-up (31 August 2026). Citations are file paths and line numbers as of this pass — line
numbers will drift as the code changes.*
