# NumberHive brand — canonical home

This is the canonical, single-source-of-truth home for NumberHive's brand guidelines and the
handful of key brand assets, established 2026-07-26 (CHG-3615) so this material lives in the
central docs repo rather than only inside `number-hive-newvis`'s own docs.

**Provenance note:** everything under `brand/assets/` and everything transcribed on the pages
below was copied from, or derived from, `number-hive-newvis` — the only repo in the ecosystem
with a complete, brand-guide-adopted implementation. Nothing here originated in this repo; this
is a canonical *copy*, not a new authoring location for brand decisions.

## Source-of-truth chain

1. **`brand/assets/Brand Guidelines Number Hive.pdf`** (v1.0) — the official brand guide. Where
   anything else disagrees with this PDF, the PDF wins.
2. **`number-hive-newvis`'s `src/theme/colors.ts`** — the living *code* representation of the
   palette, richly commented with migration history. This repo's `brand/color-tokens.md` is a
   point-in-time copy of it, not a live sync — for current values, check `colors.ts` itself.
3. **This repo (`number-hive-documentation`)** — the canonical *documentation* copy: the PDF
   itself, the key assets, and authored summaries, kept here so the whole ecosystem has one place
   to look rather than needing to know `number-hive-newvis`'s internal layout.

## What's here

| Page | Contents |
|---|---|
| [`brand-guidelines.md`](brand-guidelines.md) | Summary of the PDF: logo variants, clearspace/don't-do rules, typography, official 5-swatch colour palette |
| [`brand-voice.md`](brand-voice.md) | The tagline and loading-line copy, locked in by `number-hive-newvis`'s own test suite |
| [`color-tokens.md`](color-tokens.md) | The extended code-level colour token system (`ORANGE`, `GRAY`, `NEUTRAL`, `EXCEPTIONS`, `MONO`, `CERTIFICATE`, `HEX_FRAME`, `ICON_SURFACE`) with rationale preserved |
| [`cross-repo-consistency.md`](cross-repo-consistency.md) | Status of brand adoption across `number-hive-newvis`, `number-hive-complete`, and `number-hive-admin` — what's aligned, what still drifts, and the open font-licensing gap |

## Assets

| File | What it is |
|---|---|
| [`assets/Brand Guidelines Number Hive.pdf`](assets/Brand%20Guidelines%20Number%20Hive.pdf) | The canonical brand guide PDF, v1.0 |
| [`assets/number-hive-logo.svg`](assets/number-hive-logo.svg) | The canonical logo (icon + wordmark lockup) |
| [`assets/favicon.svg`](assets/favicon.svg) | The canonical standalone icon |

## Scope note

This change gathers and documents what already exists — it does not invent new brand decisions,
resolve the font-licensing gap, or unify `number-hive-complete`'s divergent colour systems with
the brand guide. Those would each be separate future changes if the user wants them pursued.

## History

- 2026-07-26 — Established (CHG-3615), following the user's confirmation that this repo should
  hold canonical copies of brand guidelines/assets (not just links), given there are only a
  handful of key assets. Content sourced from and verified directly against `number-hive-newvis`
  (assets, `colors.ts`, `design.md`, `brand-voice-copy.test.ts`) and `number-hive-complete`
  (cross-repo drift documentation only); `number-hive-admin` checked and confirmed to have
  nothing relevant.
