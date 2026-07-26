# Brand voice

Two pieces of copy are locked in as exact, tested strings in `number-hive-newvis` (CHG-2815) —
treat both as byte-exact quotes, not paraphrasable marketing copy.

## Tagline

> Strategy over speed. Commitment over perfection.

Appears in `number-hive-newvis`'s `<title>` and `public/manifest.json` `description` field.

## Loading line

> The math is the game.

Appears on `number-hive-newvis`'s loading screen (`index.html`).

## Source of truth

Both strings are locked in by `number-hive-newvis/src/brand-voice-copy.test.ts`, which asserts
them byte-exact against `index.html` and `public/manifest.json`. That test suite is the
authoritative check that these haven't drifted in the reference implementation — if either
phrase is used elsewhere in the ecosystem, it should be copied verbatim from here (or from that
test file), not re-typed from memory.

## History

- 2026-07-26 — Copied verbatim into this repo for CHG-3615, source-verified directly against
  `number-hive-newvis/src/brand-voice-copy.test.ts`.
