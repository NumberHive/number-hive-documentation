# Security Remediation Status — `number-hive-complete`

**Purpose:** status of the security remediation work tracked in `number-hive-complete`, as
recorded by the plan itself, cross-checked against the current code. Status only — no
recommendations, no estimates.

**Source documents (in `number-hive-complete`, not duplicated here):**
- [`SECURITY-REMEDIATION-PLAN.md`](../../number-hive-complete/SECURITY-REMEDIATION-PLAN.md) — the phase plan, dated 2026-03-29
- [`AUDIT-2026-03-23.md`](../../number-hive-complete/AUDIT-2026-03-23.md) — the audit the plan responds to, dated 2026-03-23

---

## Phase 1 — COMPLETE, per the plan (deployed to staging 2026-03-29)

### 1.1 GraphQL resolver auth guards

**Plan states:** 53 resolver methods across 17 files were missing `@Authorized()`, fixed in
commits `45e8482`, `c64d842`.

**Verified in code:** every `@Query`/`@Mutation` method across all 67 resolver files in
`backend/src/graphql/` carries `@Authorized()` or `@UseMiddleware(authenticateAdmin)`, with
one exception category: the 8 auth-flow methods in `backend/src/graphql/user/auth/auth.resolver.ts`
(`login`, `signup`, `requestSignUpCode`, `refreshAccessToken`, `requestResetPasswordCode`,
`verifyResetPasswordCode`, `resetPassword`, `signOut`) — these are the same methods both the
audit and the plan record as intentionally unprotected. No unguarded methods found outside that
list. Commit `45e8482` is present in history and touches the 17 resolver files the plan names.

**Divergence found — confirm with James:** the plan's own text says **53** methods; the
commit message on `45e8482` says *"add `@Authorized()` decorator to **69** unprotected GraphQL
resolver methods"*, and the audit (`AUDIT-2026-03-23.md`, Finding 4.1) also states **69**. The
file count (17) matches across plan, commit, and audit — only the method count in the plan's
prose disagrees with its own cited commit and the audit it's tracking. Not resolved here either
way.

### 1.2 AI move timing

**Plan states:** bundled into the same release, commit `886add8`, difficulty-scaled AI delay
config.

**Verified in code:** `886add8` is present in history; `backend/src/services/AI/ai-timing.config.ts`
exists. Not a security item — recorded here only because the plan lists it under Phase 1.

---

## Phase 2 — PLANNED, per the plan (not started)

### 2.1 JWT signature verification

**Plan states:** `validateAccessToken`/`validateRefreshToken` in `auth-service.ts` validate by
DB lookup only, never call `jwt.verify()`.

**Verified in code:** confirmed absent. `backend/src/services/auth-service.ts` — neither
`validateAccessToken` (line 29) nor `validateRefreshToken` (line 53) calls `jwt.verify()` or
any signature-checking function anywhere in the file. Both still validate purely via
`TokenPairDB.findOne(...)` lookups and an `expiresAtMs` comparison. Matches the plan's stated
status.

### 2.2 Rate limiting, security headers, CORS lockdown

**Plan states:** no `helmet()`, no rate limiting, CORS wildcard via an `allowOrigin`
monkey-patch.

**Verified in code:** confirmed absent/unchanged. `helmet` and `express-rate-limit` are not in
`backend/package.json`. `backend/src/server.ts` calls `server.applyMiddleware({ app })` with no
`cors` option passed. `backend/src/utils/express.ts` still defines the `allowOrigin` monkey-patch
(`Access-Control-Allow-Origin` defaulting to `*`), and it's still called from
`backend/src/router/index.ts` (two call sites). Matches the plan's stated status.

### 2.3 Frontend Dockerfile Node 18 → 20

**Plan states:** `frontend/Dockerfile` line 26 uses `node:18-alpine`, needs to move to
`node:20-alpine`.

**Divergence found — confirm with James:** `frontend/Dockerfile` line 26 currently reads
`FROM node:22-alpine AS builder` — not Node 18, and past the plan's own Node 20 target. The
plan lists this item as **PLANNED / not started**, but the code shows it already resolved (and
exceeded). Not clear from the repo alone whether this happened as deliberate, untracked
follow-up work or the plan status simply wasn't updated after the fact.

---

## Phase 3 — PLANNED, per the plan (not started, blocked on Phase 2 per the plan's own dependency note)

### 3.1 Refresh token expiry and rotation

**Plan states:** refresh tokens are issued with `'30y'` expiry (effectively permanent), no
rotation on refresh; needs `'30y'` → `'30d'`, a new refresh token issued on every refresh, and
a `refreshTokenExpiresAtMs` field added to the `TokenPair` model.

**Verified in code:** confirmed absent/unchanged on all three points.
- `backend/src/utils/jwtUtils.ts` line 96 — `generateRefreshToken` still calls
  `issueJWT(payload, config.tokenKeys.refreshToken.private, '30y')`.
- `backend/src/database/controllers/user.db.controller.ts`, `getUserTokenPair` (line 94) — on an
  existing token pair, the refresh token is reused unchanged: `refreshToken: tokenPair ?
  tokenPair.refreshToken : refreshTokenObject.token`. No new refresh token is issued or stored on
  refresh.
- `backend/src/database/models/tokenPair.ts` — the `TokenPair` model has `expiresAtMs` for the
  **access** token only. There is no `refreshTokenExpiresAtMs` field or equivalent, and
  `validateRefreshToken` in `auth-service.ts` performs no expiry check on the refresh token at
  all — only a DB-existence and access/refresh pairing check. **Confirmed: refresh tokens
  currently carry no expiry check anywhere in the validation path.**

Matches the plan's stated status.

---

## Summary

| Phase | Item | Plan status | Code status |
|---|---|---|---|
| 1.1 | Resolver auth guards | Complete | Confirmed present (method-count divergence noted above) |
| 1.2 | AI move timing | Complete (bundled) | Confirmed present |
| 2.1 | JWT signature verification | Planned | Confirmed absent |
| 2.2 | Rate limiting / headers / CORS | Planned | Confirmed absent |
| 2.3 | Frontend Node 18→20 | Planned | **Diverges — already on Node 22** |
| 3.1 | Refresh token expiry/rotation | Planned | Confirmed absent |

**Two findings flagged "confirm with James"** (see 1.1 and 2.3 above): a method-count
inconsistency internal to the plan document, and a Dockerfile Node version that's already ahead
of what the plan describes as not-yet-started. Neither has been corrected in the source
documents — this file reports the divergence, it doesn't resolve it.

---

*Compiled 2026-08-31 by direct inspection of `number-hive-complete`: `git log` for the cited
commits, and direct reads of `backend/src/graphql/**/*.resolver.ts` (all 67 files, scripted
scan for `@Authorized`/`@UseMiddleware` on every `@Query`/`@Mutation`), `backend/src/services/auth-service.ts`,
`backend/src/utils/jwtUtils.ts`, `backend/src/database/controllers/user.db.controller.ts`,
`backend/src/database/models/tokenPair.ts`, `backend/package.json`, `backend/src/server.ts`,
`backend/src/utils/express.ts`, and `frontend/Dockerfile`.*
