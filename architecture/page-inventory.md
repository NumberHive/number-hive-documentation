# NumberHive — Complete Page & Feature Inventory

> **Purpose**: Reference document for briefing design tools (e.g. Stitch by Google) and for team architecture discussions. Lists every screen and major UI area from the current paid app (`number-hive-complete`), annotated with user role, purpose, and key features.
>
> **Context**: The paid app is a React Native / React Native Web hybrid. We are considering a future architecture split into two separate web apps — a **Game App** (Phaser + Vite, players only) and a **Dashboard App** (standard React, educators + NH staff). See [`platform-strategy.md`](./platform-strategy.md) for the strategic rationale.
>
> **Lifecycle**: Point-in-time snapshot — update when the screen inventory changes significantly or a new external briefing is required.

---

## User Roles

| Role | Description |
|---|---|
| **Anonymous** | Visitor with no account — can play the free game |
| **Student (Free)** | Registered player in a teacher's hive |
| **Student (Paid)** | Same as above, with access to premium game types |
| **Teacher** | Educator who creates and manages hives (classrooms), buys subscriptions |
| **NH Staff (Admin)** | NumberHive internal staff — platform oversight, analytics, CRM |

---

## Architecture Summary (Current vs Target)

**Current**: Single React Native codebase serving all three interface styles — game, educator dashboard, and admin panel — bundled together.

**Target**: Two web-only apps:
- **Game App** (Phaser + Vite) — serves Anonymous, Student Free, Student Paid
- **Dashboard App** (React) — serves Teacher and NH Staff; the two admin levels are likely similar enough to share one codebase

---

## Section 1: Game / Player Screens

These screens serve players (anonymous, free, and paid). In the target architecture these all belong in the **Game App**.

---

### 1.1 Splash
- **Route**: `/splash`
- **Roles**: All users
- **Purpose**: Boot screen shown briefly while auth state resolves. Gates the ForceUpdate (minimum app version) check on mobile.
- **Features**: Brand logo, loading indicator, version enforcement logic

---

### 1.2 Lobby (Main Hub)
- **Route**: `/lobby`
- **Roles**: All users (signed-in and guest)
- **Purpose**: Central home screen — the player's starting point after loading.
- **Features**:
  - Nectar (points) counter widget
  - Dropdown navigation menu
  - "My Hives" button (students only)
  - "Admin Panel" shortcut (teachers/admins)
  - "Nectar Nook" shortcut
  - Login prompt (guests)
  - App version display
  - Three sub-view modes: Main, Play-a-Friend, Hive Gym

---

### 1.3 Main Lobby (sub-view)
- **Route**: within `/lobby`
- **Roles**: All users
- **Purpose**: Primary mode-selection panel. Entry point to all game types.
- **Features**:
  - Play Online button
  - Host Game button
  - Play with AI button
  - Pass & Play (local two-player) button
  - Journey mode button
  - Gym mode button
  - My Hives button (students)
  - Admin Panel button (teachers/admins)
  - Tooltip onboarding sequence for new users

---

### 1.4 Select Game Type
- **Route**: `/selectGameType`
- **Roles**: All users
- **Purpose**: Game configuration — choose arithmetic operation and difficulty/number range before entering any game mode.
- **Features**:
  - Operation selector: Addition, Subtraction, Multiplication, Division
  - Difficulty / number range tier selection
  - Premium lock UI (shows "Go Premium" popup for non-paid users on locked tiers)
  - Entry point for: Play Online, Host Game, Play with AI, Pass & Play, Waiting for Hive Mate, Gym Assessment

---

### 1.5 Play Online (Matchmaking)
- **Route**: `/playOnline`
- **Roles**: All users
- **Purpose**: Joins the public matchmaking queue and waits for an opponent.
- **Features**:
  - Matchmaking queue status
  - Cancel / back action
  - Auto-transitions to Game screen when opponent found

---

### 1.6 Host Game (Private Room)
- **Route**: `/hostGame`
- **Roles**: All users
- **Purpose**: Creates a private game room and displays a join code for sharing with a friend.
- **Features**:
  - Room join code display
  - Copy / share join code
  - Waiting indicator
  - Cancel action

---

### 1.7 Join Game (Enter Code)
- **Route**: `/joinGame`
- **Roles**: All users
- **Purpose**: Join a private game by manually typing a join code.
- **Features**:
  - Join code input field
  - Validation and error feedback
  - Submit action

---

### 1.8 Join Code (Deep Link Handler)
- **Route**: `/joinCode`
- **Roles**: All users
- **Purpose**: Handles the deep-link / URL invite flow. Validates a join code from a URL and enters the game directly.
- **Features**:
  - URL parameter parsing
  - Code validation feedback
  - Auto-redirect to game on success

---

### 1.9 Play with AI
- **Route**: `/playWithAI`
- **Roles**: All users
- **Purpose**: Configure and start a single-player game against an AI bot.
- **Features**:
  - AI difficulty level selector
  - Game type (inherited from SelectGameType)
  - Start game action

---

### 1.10 Pass & Play (Local Two-Player)
- **Route**: `/passAndPlay`
- **Roles**: All users
- **Purpose**: Local two-player mode — two players share one device and take turns.
- **Features**:
  - Player name entry (optional)
  - Turn-passing prompt between moves
  - Shared game board

---

### 1.11 Game Screen (Live Board)
- **Route**: `/game`
- **Roles**: All users
- **Purpose**: The core gameplay experience. The hexagonal Number Hive board where all game modes play out.
- **Features**:
  - Hexagonal tile grid (`TilesView`)
  - Operand / number selector controls
  - Turn indicator (whose turn)
  - Working bar — shows calculation being built
  - In-game hint system (`HintPopup`)
  - Win / loss / draw result modal (`ResultPopup`)
  - Journey-specific result modal (`JourneyResultPopup`)
  - Exit confirmation modal (`ExitPopup`)
  - Reconnection modal on disconnect (`ReconnectPopup`)
  - Real-time game state via GraphQL subscription
  - Streak tracking and Gym stats recording
  - Supports modes: Online, AI, Pass-and-Play, Journey, Gym Assessment

---

### 1.12 Waiting for Hive Mate
- **Route**: `/waitingForHiveMate`
- **Roles**: Teacher, Student
- **Purpose**: Waiting room shown when a teacher or student has invited a hive member to a game.
- **Features**:
  - Invite status display
  - Cancel action
  - Auto-transitions to Game when opponent accepts

---

### 1.13 Nectar Nook (Achievements & Rewards)
- **Route**: `/nectarNook`
- **Roles**: All signed-in users
- **Purpose**: The player's achievement hub — shows accumulated nectar points, streak badges, and game achievements.
- **Features**:
  - Current nectar (points) count
  - Streak badges list (sorted by count)
  - Game achievements panel across all game type categories
  - Visual reward display

---

## Section 2: Journey / Learning Path Screens

The Journey is a structured learning progression within the game. Part of the **Game App** in the target architecture.

---

### 2.1 Journey Map (Learning Path)
- **Route**: `/lobby/journey`
- **Roles**: All signed-in users
- **Purpose**: Visual roadmap of the player's learning journey — a path of connected stage nodes.
- **Features**:
  - Graphical path with stage nodes (assessment, survey, game levels)
  - Node status: locked, available, completed
  - Animated bee mascot
  - Progress bar
  - Stage selection and navigation
  - Streak celebration modal
  - Nectar counter
  - Journey detail overlay modal (`JourneyModal`)
  - Hive-awareness (fetches user's hive membership)

---

### 2.2 Journey Survey
- **Route**: `/lobby/journey/survey`
- **Roles**: All signed-in users
- **Purpose**: Self-assessment survey step within the Journey.
- **Features**:
  - Multi-stage questionnaire (confidence, enjoyment, etc.)
  - Progress indicator
  - Nectar points awarded on completion
  - Saves answers to journey history

---

### 2.3 Journey Assessment
- **Route**: `/lobby/journey/assessment`
- **Roles**: All signed-in users
- **Purpose**: Timed or un-timed arithmetic assessment — 25 questions — within the Journey.
- **Features**:
  - 25-question arithmetic set
  - Optional timer
  - Score result popup on completion
  - Nectar points award
  - Badge unlocking on milestones
  - Journey stage advancement
  - Integrates with badge system

---

### 2.4 Gym Assessment (Practice)
- **Route**: `/lobby/gym/assessment`
- **Roles**: All signed-in users
- **Purpose**: Practice assessment mode — 25 questions with optional 60-second timer.
- **Features**:
  - 25-question arithmetic set
  - 60s optional timer
  - Per-answer time tracking
  - Result popup with score
  - Increments gym practice stats

---

### 2.5 Gym Stats
- **Route**: `/lobby/gym/stats`
- **Roles**: All signed-in users (can view other users' stats via query param)
- **Purpose**: Statistics hub for Gym mode performance.
- **Features**:
  - Three tabs: Play-a-Friend stats, AI Practice stats, Buzz Game stats
  - Tabular stat display per game type
  - View other user's stats (when `queryUserId` param provided)

---

## Section 3: Teacher / Educator Screens

These screens help teachers manage their classrooms (hives). In the target architecture these belong in the **Dashboard App**.

---

### 3.1 My Hives (Hive List)
- **Route**: `/myHives`
- **Roles**: Teacher, Student
- **Purpose**: Shows all hives the user is associated with.
- **Features** (Teacher):
  - List of all owned hives with name, size, and subscription status
  - Create new hive button
  - Navigate to individual hive management
- **Features** (Student):
  - List of enrolled hives
  - Navigate to student hive view

---

### 3.2 Hive Management Dashboard
- **Route**: `/myHiveData`
- **Roles**: Teacher
- **Purpose**: The teacher's primary classroom management screen for a specific hive.
- **Features**:
  - Paginated student roster with edit / remove actions
  - Hive analytics (games played, time stats, seats used/available)
  - Date range filter for analytics
  - Hive leaderboard overlay
  - Hive rename
  - Join code display and copy
  - Subscription and payment management (PaymentPopup)
  - Force activate hive (admin-granted)
  - Navigation to: Hive Settings, Hive Reports, Hive Journeys
  - Invite individual student to a game

---

### 3.3 Student Hive View
- **Route**: `/myHiveDataStudent`
- **Roles**: Student
- **Purpose**: Student's read-only view of their enrolled hive.
- **Features**:
  - Fellow hive members list
  - Class leaderboard
  - Invite hive mate to a game button

---

### 3.4 Hive Journey Overview
- **Route**: `/hiveJourneys`
- **Roles**: Teacher
- **Purpose**: Shows the Journey progress for the whole class in one view.
- **Features**:
  - Game type (operation) selector
  - Per-student journey progress summary
  - Links through to individual student stats

---

### 3.5 Hive Settings
- **Route**: `/hiveSettings`
- **Roles**: Teacher
- **Purpose**: Configure which game types and difficulty tiers are available to students in this hive.
- **Features**:
  - Class-wide game type restrictions (board selection modal)
  - Per-student overrides table
  - Save class-wide settings
  - Save individual student overrides

---

### 3.6 Hive Reports
- **Route**: `/hiveReports`
- **Roles**: Teacher
- **Purpose**: Per-student journey progress report for the class.
- **Features**:
  - Game type filter dropdown
  - Per-student journey completion data table
  - Filterable by operation type

---

### 3.7 Select Hive Size (Plan Selection)
- **Route**: `/selectHiveSize`
- **Roles**: Teacher
- **Purpose**: Choose a subscription plan before creating a hive.
- **Features**:
  - Plan tier selector: Family, Class, School, Single User
  - Billing period toggle: Monthly / Annual
  - Price display
  - Proceeds to Create Hive

---

### 3.8 Create Hive
- **Route**: `/createHive`
- **Roles**: Teacher
- **Purpose**: Name the hive and initiate payment.
- **Features**:
  - Hive name input
  - Plan summary card
  - Stripe checkout trigger (opens browser window)
  - Handles Stripe redirect callback

---

### 3.9 Order Summary
- **Route**: `/orderSummary`
- **Roles**: Teacher
- **Purpose**: Post-purchase confirmation screen shown after successful Stripe checkout.
- **Features**:
  - Order details summary
  - Proceeds to My Hives

---

### 3.10 Join Hive (Student)
- **Route**: `/joinHive`
- **Roles**: Student
- **Purpose**: Student enters a teacher's hive join code to enrol in a class.
- **Features**:
  - Join code input
  - Validation feedback
  - Submit action

---

## Section 4: Auth / Account Screens

Shared across all user types. In the target architecture, these could live in either app (or a shared component library).

---

### 4.1 Choose User Type
- **Route**: `/chooseUserType`
- **Roles**: Unauthenticated
- **Purpose**: Registration entry point — choose to sign up as Teacher or Student.

---

### 4.2 Sign In
- **Route**: `/signIn`
- **Roles**: Unauthenticated
- **Purpose**: Email + password login.
- **Features**:
  - Username/email field
  - Password field with show/hide toggle
  - Error popup on failure
  - Visitor identity tracking on success

---

### 4.3 Sign Up
- **Route**: `/signUp`
- **Roles**: Unauthenticated
- **Purpose**: Account registration. Fields differ by user type.
- **Features** (Teacher): first name, last name, email, password, confirm password, T&C checkbox
- **Features** (Student): first name, last name, username, hive join code, password, confirm password
- Password strength indicator popup
- Terms of Service link

---

### 4.4 Verify Email
- **Route**: `/verifyEmail`
- **Roles**: Unauthenticated (post-signup)
- **Purpose**: Enter the one-time code sent to the new user's email to verify their address.

---

### 4.5 Request Password Reset
- **Route**: `/requestForgetPasswordCode`
- **Roles**: Unauthenticated
- **Purpose**: Step 1 of password reset — enter email to receive a reset code.

---

### 4.6 Verify Reset Code
- **Route**: `/verifyForgetPasswordCode`
- **Roles**: Unauthenticated
- **Purpose**: Step 2 — enter the 6-digit code received by email.

---

### 4.7 Reset Password
- **Route**: `/resetPassword`
- **Roles**: Unauthenticated
- **Purpose**: Step 3 — set a new password.

---

### 4.8 Reset Password Confirmation
- **Route**: `/resetPasswordSuccessful`
- **Roles**: Unauthenticated
- **Purpose**: Confirmation screen after successful reset, with Sign In CTA.

---

### 4.9 Subscriptions / Pricing (Public)
- **Route**: `/subscriptions`
- **Roles**: Unauthenticated
- **Purpose**: Public-facing pricing page. Shows plans with monthly/annual toggle. CTA leads to sign-up as teacher. Signed-in users are redirected to Select Hive Size.

---

## Section 5: NH Staff / Admin Screens

All admin screens require `isAdmin === true`. They use a shared sidebar layout. In the target architecture these belong in the **Dashboard App**, likely sharing the same shell as Teacher screens but with additional navigation items.

---

### 5.1 Admin Dashboard (KPI Overview)
- **Route**: `/admin/dashboard`
- **Roles**: NH Staff
- **Purpose**: Platform health at a glance.
- **Features**:
  - KPI stat cards: Total Teachers, Total Students (with monthly deltas), Active Hives, Games Today/Yesterday, Active Subscriptions, MRR, Active Users 24h/30d, Games 24h/30d
  - Recent Purchases table
  - Recent Logins table

---

### 5.2 Query Dashboard (Pinned Analytics Widgets)
- **Route**: `/admin/analyser/dashboard`
- **Roles**: NH Staff
- **Purpose**: Live dashboard of pinned saved Analyser queries — custom KPI boards built from natural language queries.
- **Features**:
  - Auto-runs all pinned queries on load
  - Each query renders as a widget (table or chart)
  - Empty state prompts to pin from the Analyser

---

### 5.3 Users List
- **Route**: `/admin/users`
- **Roles**: NH Staff
- **Purpose**: Browse and search all user accounts.
- **Features**:
  - Paginated table: Name, Email, Type (with Admin badge), Joined date, Hive count
  - Search by name/email
  - Filter by type: All / Students / Teachers / Admins
  - Click through to User Detail

---

### 5.4 User Detail
- **Route**: `/admin/users/:userId`
- **Roles**: NH Staff
- **Purpose**: Full profile view for any user account.
- **Features**:
  - User info card (name, email, type, auth method, joined date, last login, nectar points)
  - Hive membership table (name, student count, payment status)
  - Actions: Impersonate user, Promote/Remove Admin, Delete User
  - Billing panel — Stripe billing and subscription history (teachers only)
  - AI Follow-Up Report panel — Claude-generated follow-up email preview with raw activity data (teachers only)
  - Confirmation modals for destructive actions

---

### 5.5 Hives List
- **Route**: `/admin/hives`
- **Roles**: NH Staff
- **Purpose**: Browse all hives across the platform.
- **Features**:
  - Paginated, searchable table
  - Columns: Hive name, Teacher (linked), Student count, Created date, Last Active, Payment status
  - Click through to Hive Detail

---

### 5.6 Hive Detail (Admin)
- **Route**: `/admin/hives/:hiveId`
- **Roles**: NH Staff
- **Purpose**: Admin view of a specific hive.
- **Features**:
  - Hive info: name, join code, created date, teacher (linked)
  - Student roster (name, username, nectar points — each linked to User Detail)
  - Impersonate Teacher action

---

### 5.7 Analytics Hub
- **Route**: `/admin/analytics`
- **Roles**: NH Staff
- **Purpose**: Comprehensive platform analytics — multiple filterable panels covering acquisition, activation, retention, and commercial signals.
- **Features**:
  - **Event Funnel Table** — raw analytics events by area (Demo Funnel, Gameplay, Auth) and time range (7d/30d/90d/All)
  - **Acquisition Panel** — signups by UTM source/campaign with avg days-to-signup
  - **Countries Panel** — visitor and signup breakdown by country
  - **Multi-touch Panel** — avg touchpoints before signup, linked visitors, conversion rate, attribution breakdown
  - **Activation Funnel (B2B)** — weekly cohort funnel: signup → hive created → first game hosted → game completed (with conversion %)
  - **At-Risk Teachers Panel** — teachers inactive for configurable window (7/14/30/60d), paginated, shows last activity and hive/game flags
  - **Session Frequency Panel** — weekly histogram of sessions/user, avg and total active users
  - **Hive Activity Panel** — per-hive game counts, last game date, active/dormant status
  - **District Signals Panel** — new teacher signups and hive activations by country (last 30d)

---

### 5.8 AI Analyser (Natural Language Query Interface)
- **Route**: `/admin/analyser`
- **Roles**: NH Staff
- **Purpose**: Ask questions about platform data in plain English; get structured results powered by Claude AI querying MongoDB directly.
- **Features**:
  - Chat thread with streaming AI responses
  - Response types: pipeline (table + CSV + optional chart), guidance (explanation), clarify (follow-up question)
  - Agent Tray — multiple concurrent streaming sessions
  - Saved Queries panel — save and reload previously run pipelines
  - Artifact Library — save named query results for reuse in reports
  - Collections Browser — browse MongoDB collections and their schemas
  - Collection schema detail modal
  - Follow-up prompt chips
  - Chart rendering on pipeline results
  - Re-run as chart option
  - Multi-session concurrent querying

---

### 5.9 Schema Annotations Manager
- **Route**: `/admin/analyser/annotations`
- **Roles**: NH Staff
- **Purpose**: Manage field-level annotations that inform the AI Analyser about what database fields mean.
- **Features**:
  - Annotations listed grouped by collection
  - Inline add / edit / delete per annotation
  - Free-text annotation content

---

### 5.10 Subscriptions CRM
- **Route**: `/admin/subscriptions`
- **Roles**: NH Staff
- **Purpose**: Full subscription management — track, filter, and act on all teacher subscriptions.
- **Features**:
  - Summary KPI bar with clickable cards: Active, Payment Failed, Lapsed, Never Converted
  - Filterable table:
    - Search by teacher name/email
    - Status filter (Active, Payment Failed, Cancelled, Not Paid, Pending)
    - Renewal type filter (Monthly / Annual)
    - Hive size filter (Class / Family / School / Single)
    - Last game activity filter (Active / At Risk / Dormant / No games)
  - Sortable columns: Last Game, # Hives, Renewal, Price, Created
  - Columns: Hive Name, Teacher, Email, Size, Type, Last Game, # Hives, Renewal, Status, $/mo, Created
  - Click-through to User Detail
  - CSV export of full filtered dataset

---

### 5.11 Demo Leads
- **Route**: `/admin/demo-leads`
- **Roles**: NH Staff
- **Purpose**: Sales lead management — view and action demo sign-up leads from the public demo site.
- **Features**:
  - Table: Email, Source, Pathway, Country, Date Requested
  - "Resend" action per lead to re-trigger demo welcome email

---

### 5.12 Platform Settings
- **Route**: `/admin/settings`
- **Roles**: NH Staff
- **Purpose**: Platform-wide configuration.
- **Features**:
  - **Add Admin** — look up teacher by email and promote to admin
  - **Admin Accounts** — table of all admins with remove action (cannot remove self)
  - **AI Follow-Up Email Prompt** — editable system prompt for Claude when generating weekly teacher follow-up emails
  - **Daily Briefing Email** — manual trigger button for the daily admin briefing email

---

## Section 6: Utility / System Screens

---

### 6.1 Stripe Redirect Handler
- **Route**: `/serverRedirect`
- **Roles**: All
- **Purpose**: Landing page for Stripe checkout redirect. Communicates success or error back to the main app window via `postMessage`, then closes the tab.

---

### 6.2 Web-Only Interstitial
- **Route**: `/webOnly`
- **Roles**: All
- **Purpose**: Shown on mobile when a user tries to access a web-only feature (e.g. Stripe checkout). Prompts them to use the browser version.

---

### 6.3 Contact Us
- **Route**: `/contactUs`
- **Roles**: All
- **Purpose**: Shows the support email address. Copies to clipboard (web) or opens native email client (mobile).

---

## Page Count Summary

| Area | Screens |
|---|---|
| Game / Player | 13 |
| Journey / Learning Path | 5 |
| Teacher / Educator | 10 |
| Auth / Account | 9 |
| NH Staff / Admin | 12 |
| Utility / System | 3 |
| **Total** | **52** |

---

## Target Architecture Mapping

| App | Screens |
|---|---|
| **Game App** (Phaser + Vite) | Game/Player (1–13), Journey/Learning Path (2.1–2.5), Auth shared components |
| **Dashboard App** (React) | Teacher/Educator (3.1–3.10), NH Staff/Admin (5.1–5.12), Auth shared components |
| **Shared / Undecided** | Auth/Account screens (4.1–4.9), Utility screens (6.1–6.3) |

---

## Appendix: Free Game (`number-hive-newvis`) — Delta

The rest of this document is a point-in-time snapshot of the **paid app**
(`number-hive-complete`) — see the header note. It does not cover
`number-hive-newvis`'s own Phaser scenes. This appendix is scoped narrowly to
new screens/states that repo gained from **CHG-2330** (seeded challenge
mode), so the addition doesn't get lost, without attempting a full inventory
of that repo's UI.

| Screen/state | Where | Purpose |
|---|---|---|
| Challenge deep-link loading | `ModeScene._handleChallengeDeepLink` | Shown while `?challenge=<id>` is resolved via `ChallengeClient.getChallenge` |
| Challenge not found / expired | `ModeScene._handleChallengeDeepLink`'s `showNotFound` | Graceful fallback for an unknown/expired challengeId or a malformed seed — never a raw error |
| Challenge offline / retry | `ModeScene._renderOfflineRetry` (shared with the pre-existing `?match=` deep-link) | Offline guard + Retry button for both deep-link flows |
| Challenge Result end-screen | `src/ui/ChallengeResultCard.ts` (`showChallengeResultCard`) | Replaces the standard `ResultCard`/BUZZ post-game card for `gameMode === 'challenge'` — shows Challenge Score, best-of-N, creator comparison, retry/share |
| "Challenge a Friend" action | `src/ui/ResultCard.ts` (`_doChallengeCreate`), vs_ai result card only | Creates a new seeded challenge and shares its link |

*For `number-hive-newvis`'s full architecture, see its [`docs/architecture.md`](../../number-hive-newvis/docs/architecture.md); for the authoritative product/architecture decisions behind CHG-2330 see [`number-hive-complete`'s ADR-002](../../number-hive-complete/docs/adr/002-free-game-data-architecture.md).*

---

*Inventory extracted from `/home/james/dev/number-hive-complete` — June 2026.*
*For strategic context see [`platform-strategy.md`](./platform-strategy.md), moved alongside this document.*

---

*Migrated from `number-hive-newvis/docs/page-inventory.md` to the central NumberHive documentation repo on 2026-07-24, alongside `platform-strategy.md` (the two are kept together — this file is supporting evidence for that discussion). Pre-migration history lives in that repo's git log. The "Appendix" section above documents screens local to `number-hive-newvis` itself and was carried over unchanged rather than split out, since nothing else currently links into it directly.*
