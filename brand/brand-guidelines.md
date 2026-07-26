# Brand guidelines — summary

**Authoritative source:** [`brand/assets/Brand Guidelines Number Hive.pdf`](assets/Brand%20Guidelines%20Number%20Hive.pdf)
(v1.0). This page is a quick-reference summary — **where this page and the PDF disagree, the
PDF wins.** Update this page to match, don't relitigate the brand guide here (same rule
`number-hive-newvis/docs/design.md` applies to itself).

The PDF's own print-spec detail (RGB/CMYK breakdowns, full page-by-page layout) is not
transcribed below to avoid transcription drift — open the PDF directly for those. What
follows is the subset that's been carried into code and is safe to rely on for day-to-day
reference.

## Logo

- **Lockup** — the standalone hexagon icon (below) beside the "NUMBER HIVE" wordmark, set in
  **Hot Grill** (display font), with small numeral-flourish accents in **Kurri Island**.
- **Primary vs. inverted** — these are the *same* gradient artwork, used on light vs. dark
  backgrounds respectively. There is no separate colour treatment for "inverted" — it's a
  placement context, not a variant.
- **Standalone icon** — the hexagon alone (no wordmark), used for app-icon / favicon contexts.
  Per the guide, it is **not** a single precise mathematical hexagon: the guideline artwork is
  an "exploded" (gapped) stack of 3 bands of unequal height (gray / orange / gray, top to
  bottom), a small orange "spout" wedge above the top band, and 6 small Fibonacci-sequence
  (1, 1, 2, 3, 5, 8) decorative flourish marks around the bands. Note: as of `number-hive-newvis`'s
  CHG-3590/CHG-3594, none of its checked-in files reproduce this guideline geometry
  byte-for-byte any more — see `brand/cross-repo-consistency.md` for the current state of the
  reference implementation.
- **Monochrome variant** — a flat black-on-white / white-on-black version, for print/share-card
  contexts.
- **Clearspace & don't-do rules** — maintain clearspace around the lockup of at least the
  height of the hexagon icon. Never recolour, distort, rotate, or apply a drop-shadow/outline to
  the logo artwork. Never place it on a background that doesn't meet contrast requirements
  against the gradient's lightest stop.

## Typography

| Role | Typeface | Notes |
|---|---|---|
| Logo wordmark | **Hot Grill** (display) | Not self-hosted anywhere in the ecosystem today — see the open licensing gap below. |
| Logo numeral-flourish accents | **Kurri Island** | Same gap as above. |
| Suggested general UI typeface | **Noto Sans** | The guide's own recommendation for body/UI text — not adopted by any repo checked so far; each product has instead kept its own pre-existing UI typefaces (see `cross-repo-consistency.md`). |

**Open gap — font licensing:** the brand PDF only links to third-party `dafont.com` download
pages for Hot Grill and Kurri Island. No licence check has been done anywhere in the ecosystem,
and no repo self-hosts either font file. This is flagged, not resolved, here — see
`cross-repo-consistency.md`.

## Official colour palette

The guide's base palette is five swatches (confirmed exact against `number-hive-newvis`'s
`src/theme/colors.ts`, which cites this PDF as its source of truth):

| Swatch | Hex |
|---|---|
| Orange (light stop) | `#FFCF38` |
| Orange (dark stop) | `#F26D24` |
| Gray (light stop) | `#DEDEDE` |
| Gray (mid stop) | `#949494` |
| Gray (dark stop) | `#595959` |

The two "Orange" and two "Gray" values above are gradient stops (light/dark ends of one
orange-gold ramp and one two-part gray ramp sharing the `#949494` midpoint), not four
independent flat colours — see `brand/color-tokens.md` for how the reference codebase
represents and extends this.

## History

- 2026-07-26 — Summary authored for CHG-3615, transcribed from the PDF's content as already
  interpreted and verified in `number-hive-newvis/docs/design.md` (cross-checked directly
  against that file and against `src/theme/colors.ts`, both re-read in full during this change).
