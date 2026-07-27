# Colour tokens

The five base swatches in the brand PDF (see `brand-guidelines.md`) are extended, in code, into a
richer token system. This page is a point-in-time copy of `number-hive-newvis/src/theme/colors.ts`
(re-read in full for this change) — **it is not live-synced.** For any future work, treat
`number-hive-newvis/src/theme/colors.ts` and `number-hive-newvis/docs/design.md` as the living
sources of truth; this page will drift out of date as that codebase evolves and should be
re-copied periodically rather than trusted as current forever.

Every token in the source file is exported in both CSS hex-string form and Phaser numeric
(`0x...`) form (suffixed `Hex`) — only the hex form is shown below for brevity.

## `ORANGE` — brand orange/gold gradient

Color chips section of the brand PDF. `light` is the default fill/hover/highlight stop; `dark`
is the pressed/shadow/stroke-emphasis stop.

| Token | Hex |
|---|---|
| `ORANGE.light` | `#FFCF38` |
| `ORANGE.dark` | `#F26D24` |

## `GRAY` — brand gray gradients

The brand guide defines two gray ramps sharing a common midpoint stop (`#949494`): dark-gray
(`#949494` → `#595959`) and light-gray (`#949494` → `#DEDEDE`). Exposed as one three-stop ramp
since `mid` is shared.

| Token | Hex |
|---|---|
| `GRAY.mid` | `#949494` |
| `GRAY.dark` | `#595959` |
| `GRAY.light` | `#DEDEDE` |

## `NEUTRAL` — unified dark-theme neutral (CHG-2409 decision 6)

A single unified dark-theme neutral, derived by extending the brand dark-gray gradient toward
near-black for full-screen background use. Replaces three previously-conflicting navy/purple
neutrals that used to coexist in `number-hive-newvis` (dead `ModeScene.ts` `DARK_BG`/`index.html`
`#bg-gradient` family; `StatsScene.ts`/`BeeCTAScene.ts`'s purple-navy family; `public/manifest.json`/
`src/main.ts`'s `#1a1030`/`#1a1a2e` family). Pinned values, not placeholders.

| Token | Hex |
|---|---|
| `NEUTRAL.bgDark` | `#1F1F1F` |
| `NEUTRAL.panelDark` | `#2E2E2E` |
| `NEUTRAL.borderDark` | `#595959` (= `GRAY.dark`) |

Migration status (per `number-hive-newvis/docs/design.md`): `StatsScene.ts`, `BeeCTAScene.ts`,
and `ModeScene.ts`'s dead `DARK_BG` constant were repointed to these tokens by CHG-2409.
`index.html`, `public/manifest.json`, `src/main.ts`, and `GameScene.ts`'s decorative
background-variety array were intentionally **not** migrated — see `design.md`'s own migration
table for current status rather than assuming it's complete.

## `EXCEPTIONS` — sanctioned non-brand colours (CHG-2409 decision 4)

Intentionally **not** part of the brand palette and must never be "corrected" toward brand
orange/gray — each carries its own semantic meaning:

| Token | Hex | Meaning |
|---|---|---|
| `player2Blue` | `#2563EB` | Second-player identity colour (`TileStyles.ts`) |
| `player2BlueAccent` | `#BFDBFE` | Accent variant of the above |
| `correctGreen` | `#10B981` | Correct-answer / win-state colour (`TileStyles.ts`) |
| `correctGreenAccent` | `#6EE7B7` | Accent variant of the above |
| `identityAccentCyan` | `#22D3EE` | CHG-2407 "player/identity — personal stats" accent (StatsScene roundel, edge-peek sliver, nudge) |

The `StatsScene` progress-bar red/amber/green semantic lerp is a function over a colour range,
not a fixed constant, and is intentionally not reproduced here.

## `MONO` — flat monochrome logo variant (CHG-2409 decision 1)

| Token | Hex |
|---|---|
| `MONO.black` | `#000000` |
| `MONO.white` | `#FFFFFF` |

Not wired into any screen yet in the reference implementation — reserved for future
share-card/print use.

## `CERTIFICATE` — HIQ certificate surface (CHG-2422)

A warm, NH-shell light-theme surface (cream card + gold seal/ornament), deliberately distinct
from `StatsScene`'s dark/cyan palette. Achieves a "certificate of achievement" read through
layout/typography/ornament, not literal parchment texture.

| Token | Hex | Purpose |
|---|---|---|
| `cream` | `#FCEFC9` | Card background |
| `gold` | `#D4A72C` | Seal/ornament |
| `ink` | `#3B2F1E` | Body text, chosen for contrast against `cream` |

## `HEX_FRAME` — hex-frame bee mascot rim (CHG-3514)

Approximates a redesign handoff's gold-gradient rim (`~#FFE79A → #F0A81C`) as two flat stops,
per the same "gradients become flat tokens" rule as `ORANGE`/`GRAY` above.

| Token | Hex | Purpose |
|---|---|---|
| `rimLight` | `#FFE79A` | Thin outer stroke ("lit edge") |
| `rimDark` | `#F0A81C` | Rim's own fill |

## `ICON_SURFACE` — favicon/PWA-icon cream background (CHG-3594)

A flat approximation of the checked-in favicon/PWA-icon PNGs' subtle diagonal-gradient cream
background (sampled range approx. `#FBF6EC` top-left to `#F1E6D2` bottom-right; this token is the
midpoint). Used by `public/favicon.svg` and `scripts/generate-favicon.js` — the SVG/raw-PNG
generation path can't import this module, so those two literal duplicates must be kept in sync
manually if this value ever changes.

| Token | Hex |
|---|---|
| `cream` | `#F6EEDF` |

## History

- 2026-07-26 — Copied for CHG-3615, transcribed directly from `number-hive-newvis/src/theme/colors.ts`
  (re-read in full, all 161 lines / 8 exported token groups) and cross-checked against
  `number-hive-newvis/docs/design.md`. Re-verified this is the complete current token list, not
  just the list from earlier research.
