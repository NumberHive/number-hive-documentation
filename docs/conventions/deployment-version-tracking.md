# Deployment version tracking — cross-repo convention

## Why this lives here, not in one product repo

Every NumberHive repo (`number-hive-complete`, `number-hive-newvis`, `number-hive-admin`, and
`amber` once it's building) tracks events per `docs/conventions/analytics-and-ops-logging.md`
and pushes some of that data cross-repo per `docs/conventions/cross-repo-data-push.md`. Neither
of those documents fixes *which build of the code* produced a given event. Without that, two
recurring problems show up as the ecosystem grows:

- **User-facing timing confusion.** "Did I see the fix yet?" is unanswerable from inside the
  product without knowing when the currently-running code actually went live, versus when a
  ticket was closed or a change was merged.
- **Engineer debugging drag.** When an event looks wrong (unexpected shape, a bug reproducing
  intermittently, a regression report), the first two questions are always "what code was
  running when this happened?" and "when did that code go live?" Without both answers attached
  to the event itself, answering requires cross-referencing deploy logs, `git log`, and
  timestamps by hand, per repo, every time — and that reconstruction gets slower and less
  reliable the more repos and environments exist.

Reference implementation: `number-hive-admin` (CHG pending on that repo's board as of
2026-08-05). This document generalises that pattern so `number-hive-newvis` and future repos
adopt the same shape rather than inventing their own.

## The convention

1. **Every tracked event carries both a deployment timestamp and a commit hash.** Not just
   error/ops events — product/growth events too (per `analytics-and-ops-logging.md`'s two
   audiences; this applies to both). The two values answer different questions and neither
   substitutes for the other:
   - **`deployedAt`** — *when did this running code go live* (Unix ms). Answers the user-facing
     "have I got the update yet" question, and lets you bucket events by deploy window without
     joining against a separate deploy log.
   - **`versionHash`** — *which code is this* (the deployed commit SHA). Answers the engineer's
     "what was actually running" question — the thing `deployedAt` alone can't tell you, since a
     redeploy of the same commit (e.g. an env var change or a platform restart) gets a new
     `deployedAt` but should keep the same `versionHash`, and vice versa: same `deployedAt`
     doesn't guarantee same code if two services deploy in the same window.

   Together they let you distinguish "this got worse because new code shipped" from "this got
   worse for some other reason after the same code had already been running a while" — a
   distinction a single value (either one alone) cannot make.

2. **Injected at build time, not read at runtime from the deploy platform.** The value must be
   baked into the build artifact itself, not fetched from an API or environment lookup at
   request time. This keeps the value available even if the deploy platform's own metadata API
   is slow, rate-limited, or unreachable, and guarantees client and server agree on what shipped
   (both read the same baked-in artifact, not two independent runtime queries that could race
   or disagree during a rollout).

   Pattern, in order:

   ```
   CI/deploy platform sets env vars at build time
            │  (RENDER_GIT_COMMIT, a deploy-time `date +%s%3N`, etc. — platform-specific)
            ▼
   Build step writes them into a build artifact
            │  (e.g. a generated version.json, or inlined as a bundler define)
            ▼
   Server/client reads the artifact once on startup
            │  (not per-request, not per-event — read once, cache in memory)
            ▼
   Every tracked event attaches the cached { deployedAt, versionHash } from that read
   ```

   Reading once on startup (not per-event) matters for the same reason as the build-time
   injection itself: it keeps event-tracking on the hot path independent of any external
   lookup, and it's the only way a redeploy is guaranteed to be reflected — a value cached at
   process start naturally rolls over on the next restart, no manual invalidation needed.

3. **Payload shape — two fields, always named exactly this way:**

   ```json
   {
     "deployedAt": 1754398800000,
     "versionHash": "a1b2c3d"
   }
   ```

   - `deployedAt` — Unix milliseconds (`number`), not ISO8601. This deliberately differs from
     `cross-repo-data-push.md` §4's envelope, which uses ISO8601 for `occurredAt` — that field
     times the *event*; `deployedAt` times the *deploy* and is intended for fast numeric
     bucketing/comparison (deploy-window grouping, "how long has this build been live") rather
     than human-readable display, so milliseconds-since-epoch was chosen deliberately, not by
     accident. Convert for display at the UI layer, not at the tracking layer.
   - `versionHash` — the deployed commit's SHA, short form (7 chars) is sufficient for display
     and correlation; store the full SHA if the repo's own tooling already has it cheaply
     available, but don't require a lookup to get it.
   - These two fields sit **alongside** whatever a specific tracked event already carries, and
     alongside the `cross-repo-data-push.md` envelope where that convention also applies (an
     event can be both "tracked locally per `analytics-and-ops-logging.md`" and "pushed
     cross-repo per `cross-repo-data-push.md`" — `deployedAt`/`versionHash` travel with it
     either way, added once at the tracking layer, not re-derived at the push layer).

4. **Same field names, every repo, every event.** No repo-specific renaming (`deployTime`,
   `commitSha`, `buildVersion`, etc.) — the whole point is that anyone debugging across repos
   can grep for the same two field names anywhere in the ecosystem's event stores.

## Node/Render example (`preDeployCommand` approach)

Render runs `preDeployCommand` after build, before the new instance takes traffic — the right
point to generate a version artifact, since it has access to the build environment and runs
exactly once per deploy (not once per instance, if a service scales to multiple instances).

**`render.yaml`:**

```yaml
services:
  - type: web
    name: number-hive-admin
    preDeployCommand: node scripts/write-version.js
```

**`scripts/write-version.js`** (runs at deploy time, writes the artifact both server and client
will read):

```js
const fs = require('fs');

const version = {
  deployedAt: Date.now(),
  // RENDER_GIT_COMMIT is populated automatically by Render's build environment —
  // no manual wiring needed on Render specifically; other platforms will expose
  // their own equivalent (CI-provided commit SHA env var) and should use that instead.
  versionHash: (process.env.RENDER_GIT_COMMIT || 'unknown').slice(0, 7),
};

fs.writeFileSync('./dist/version.json', JSON.stringify(version));
```

**Server/client, on startup (read once, cache, attach to every event):**

```js
const version = JSON.parse(fs.readFileSync('./dist/version.json', 'utf8'));

function track(eventName, payload) {
  emit({
    eventName,
    ...payload,
    deployedAt: version.deployedAt,
    versionHash: version.versionHash,
  });
}
```

For a frontend bundle rather than a Node server, the same `version.json` gets fetched once on
app load (or inlined via the bundler's `define`/`replace` config at build time instead of a
runtime fetch) and cached in memory the same way — the principle (bake at build, read once,
cache) is platform- and runtime-agnostic; only the mechanics of *how* the value gets into the
running process differ between a Render `preDeployCommand`, a different CI's build step, or a
bundler define.

## Per-repo instantiation

| Repo | Status | Notes |
|---|---|---|
| `number-hive-admin` | Reference implementation, CHG pending as of 2026-08-05 | First adopter — this document generalises that repo's approach |
| `number-hive-newvis` | Not yet adopted | Next in line — prioritised given the upcoming friends-and-family launch and the existing `number-hive-newvis` → `number-hive-admin` events flow (`cross-repo-data-push.md`), where having `deployedAt`/`versionHash` on those pushed events will matter for debugging as soon as real usage starts |
| `number-hive-complete` | Not yet adopted | — |
| `amber` | Not yet adopted (repo still early/shell stage) | Adopt when event tracking is built |

Add a row here (or update an existing one) when a repo adopts this pattern, same as
`analytics-and-ops-logging.md`'s per-repo instantiation table.

## What this does not decide

This document fixes the two field names, their types, and the build-time-injection principle.
It does not mandate a specific artifact format (`version.json` vs. a bundler `define` vs.
something else) or a specific CI/deploy platform's exact mechanics beyond the Render example
above — those are implementation details per repo, so long as the injection happens at build
time (not runtime lookup) and the two field names/types match exactly.

## History

- 2026-08-05 — Established at the request of `number-hive-admin`'s Assistant, who is building
  the reference implementation there (CHG pending on that repo's board). Documented here per
  this repo's charter (`README.md`: "shared conventions... multiple repos need to agree on")
  so `number-hive-newvis` and future repos follow the same shape rather than each inventing
  their own timestamp/version fields independently.
