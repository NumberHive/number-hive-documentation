# Cross-repo brand consistency

**Status: point-in-time snapshot, verified 2026-07-26.** This page reports what was found in
each repo on that date — it has no mechanism to stay in sync as those repos change further.
Treat it as a starting point for the next audit, not a live dashboard. If a repo's brand
adoption changes materially, update this table (and cite the change/commit that did it) rather
than assuming it's still accurate.

## `number-hive-newvis` (public game) — fully adopted, reference implementation

The only repo checked with a complete, brand-guide-adopted implementation. Adopted starting
CHG-2409 ("Brand palette & logo adoption — theme foundation"). Its own `docs/design.md` declares
`docs/Brand Guidelines Number Hive.pdf` (v1.0) and `src/theme/colors.ts` as **dual sources of
truth** — where they disagree, the PDF wins and `colors.ts`/`design.md` get updated to match.

Migration is still **partial** per that repo's own `design.md` migration-status table (e.g.
`index.html`, `public/manifest.json`, `src/main.ts`, and `GameScene.ts`'s decorative
background-variety array were intentionally not migrated to `NEUTRAL` tokens as part of
CHG-2409) — see `design.md` directly for the current per-file status rather than duplicating
that table here.

## `number-hive-complete` (school product) — two divergent, non-brand-compliant systems

Neither of the following references the brand PDF at all:

### (a) `frontend/src/constants/colors.ts` — legacy prototype-era palette

A pre-brand-guide palette, still actively imported across the app. Confirmed importers (via
direct search, 2026-07-26): `navigators/Header.tsx`, `navigators/AdminNavigator/AdminNavigatorDrawer.tsx`,
`mightyByteLibraries/MB_Modal/MB_Modal.tsx`, `mightyByteLibraries/MB_AnimatedSpinnierSquare.tsx`,
`mightyByteLibraries/MB_Button.tsx`, `utils/utils.tsx`, `components/screens/VerifyEmail/VerifyEmail.tsx`,
`components/screens/WaitingForHiveMate/WaitingForHiveMate.tsx`, `components/screens/HiveReports/HiveReports.tsx`,
`components/screens/ChooseUserType/ChooseUserType.tsx`.

Sample values (confirmed against the file directly):

| Name | Hex |
|---|---|
| `AllportsBlue` | `#03769e` |
| `englishViolet` | `#4D2D52` |
| `plum` | `#9A4C95` |
| `darkPurple` | `#1D1A31` |
| `primaryPink` | `#FF239A` |
| `pumpkinYellow` | `#E89823` |

None of these correspond to the brand guide's orange/gray palette.

### (b) `docs/game/design-system.md` — separate semantic system for the free-game "NH shell"

A distinct system for the "Number Hive on the outside, your game on the inside" free-game shell,
using **Fredoka** as its typeface (not Hot Grill/Kurri Island/Noto Sans). Semantic colour roles:

| Role | Colour | Hex |
|---|---|---|
| Primary / teacher / growth | Amber | `#F59E0B` |
| Player / identity | Cyan | `#22D3EE` |
| Achievement / monetisation | Green | `#10B981` |
| Admin / expansion | Purple | `#8B5CF6` |
| Clash / competitive | Pink | `#EC4899` |
| Live / urgent | Red | `#EF4444` |

Partially brand-aligned by coincidence rather than derivation: the amber/cream tone is in the
same family as brand orange, and this system's cream value (`#FCEFC9`, per its own doc) is
identical to `number-hive-newvis`'s `CERTIFICATE.cream` token — but neither system was built by
referencing the PDF, and the overlap isn't a deliberate cross-repo alignment.

## `number-hive-admin` — no brand assets or colours found

Checked directly (2026-07-26): no brand/colour source files exist in this repo (only
`node_modules` documentation incidentally matched a text search for "brand"/"colour"). Likely
predates any visual design work on this service.

## Open gap — logo display fonts not self-hosted anywhere

Hot Grill (wordmark) and Kurri Island (numeral-flourish accents) are not bundled or self-hosted
in any repo checked. The brand PDF only links to third-party `dafont.com` download pages, and no
licence check has been done (confirmed per `number-hive-newvis/docs/design.md`, which explicitly
flags this and says "if licensed font files are obtained, revisit in a follow-up change"). This
is flagged here as an open cross-repo question — **not resolved** as part of this change.

## Summary table

| Repo | Brand-guide adopted? | Divergence |
|---|---|---|
| `number-hive-newvis` | Yes (CHG-2409+), reference implementation | Migration partial — see its own `design.md` |
| `number-hive-complete` | No | Two separate non-brand palettes (legacy prototype colours + free-game semantic system) |
| `number-hive-admin` | N/A | No brand assets exist yet |

## History

- 2026-07-26 — Authored for CHG-3615, based on direct verification against all three repos
  (file reads, import search) rather than secondhand summary. Scope is documentation only — no
  attempt made here to unify `number-hive-complete`'s palettes or resolve the font-licensing gap;
  either would be a separate future change.
