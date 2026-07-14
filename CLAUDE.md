# Food Insulin Demand Ledger — Project Context

This file gives Claude Code everything needed to pick up development on this project.
It's a single-file HTML app (no build step, no dependencies) for looking up, tallying,
and estimating Food Insulin Demand (FID) values for PCOS / metabolic health nutrition
tracking. Read this file fully before making changes.

## What this app is

A reference tool built around the **Food Insulin Index (FII)** / **Food Insulin Demand
(FID)** system developed by Dr. Jennie Brand-Miller's group at the University of Sydney
and clinically packaged by Dr. Kirstine Bell. It has three parts:

1. **Food Ledger** — a searchable, filterable table of ~95 foods with directly-tested
   FID values, sourced from Kirstine Bell's patient booklet (see `DATA_SOURCES.md`).
2. **Today's Tally** — a running meal-builder: tap "+ Add" on any food, see a running
   FID total, optionally set a daily FID target with a progress bar. Persists across
   sessions via the artifact `window.storage` API (falls back to in-memory if
   unavailable, e.g. when the file is opened outside claude.ai).
3. **Estimate a Food's FID** — three calculator tabs for foods *not* in the ledger:
   - **Closest match** (default tab) — nearest-neighbor search against the ledger by
     macronutrient % of calories, then scales the matched food's tested FID to the
     user's portion size. This is the recommended method and mirrors the *official*
     guidance in the Bell booklet's FAQ ("if there's a similar food in the booklet, use
     its FID as a guide").
   - **I know the FII** — direct formula: `FID = calories × FII ÷ 239`.
   - **Macro regression** — estimates FII from carbohydrate and protein grams only,
     using the published Bell et al. 2016 (*Nutrients*) regression equation. Flagged
     as the fallback option because it has no fat term and runs high on high-fat foods
     (see `CALCULATOR_METHODOLOGY.md` for the full walnut/egg validation story).

## Current file

- `app/insulin-demand-ledger.html` — the entire app (single file: inline `<style>` and
  `<script>`, no external JS dependencies except Google Fonts). This is the canonical,
  most up-to-date version as of the end of the source chat. Treat it as the starting
  point, not something to rebuild from scratch.

## Design system (if extending the UI)

- **Palette**: paper background `#EEF1EE` with a faint horizontal ruled-line texture,
  ink `#1B1F1D`, primary accent deep teal `#2F6659` (with `#e2ece9` tint), amber
  `#C9932A` (`#f6ecd8` tint) for moderate/caution states, rust `#B4432E` (`#f4e2dd`
  tint) for high/warning states.
- **Type**: `Fraunces` (serif, display/headings/big numbers), `IBM Plex Sans` (body),
  `IBM Plex Mono` (labels, eyebrows, data/numerals) — loaded from Google Fonts.
- **Motif**: a lab-notebook / clinical-ledger aesthetic — mono numerals, thin rule
  lines, small-caps mono eyebrows, apothecary-style dosage bars next to each FID value.
- Two-column layout (`.grid`): Food Ledger on the left (wider), Tally + Calculator
  stacked on the right. Collapses to one column under 820px.

## Data model

Each food in the `FOODS` array has:
```js
{ name, qty, fid, cat, carb, protein, fat, cal, est?, free? }
```
- `fid` — tested Food Insulin Demand for the stated `qty` portion.
- `carb`/`protein`/`fat` — grams for that same portion (see `DATA_SOURCES.md` for how
  these were derived — they are **not** from the source booklet, which doesn't publish
  macros, and should be treated as reasonable USDA-typical estimates, not precise).
- `cal` — calories for that portion (used for scaling in the calculator tabs).
- `est: true` — FID value is composition-estimated, not directly tested in humans
  (currently only Almond Butter). Rendered with a `†` marker.
- `free: true` — food is officially exempt from FID testing because it's too low in
  kilojoules (per the Bell booklet FAQ: berries, leafy greens, salads, diet drinks).
  Rendered with a `‡` marker. FID is set to 0 for these.

`CATS` array drives the category filter pills: `all, protein, grain, vegetable, fruit,
dairy, legume, fat, snack, condiment, beverage, meal`.

## Key decisions worth knowing before changing anything

- **Why "Closest match" is the default calculator tab, not the regression**: the
  macro regression (carb + protein only, from Bell et al. 2016) was validated against
  walnuts (real FID 5 for a 240-cal portion) and came out at ~18 — badly wrong,
  because fat isn't a term in the published equation. A fat-penalty correction was
  tried and rejected: calibrating it to fix walnuts breaks the prediction for
  poached eggs (a different high-fat-but-protein-forward food). Full writeup in
  `CALCULATOR_METHODOLOGY.md`. Conclusion: don't force a fudge factor; prefer
  nearest-neighbor matching against real tested values, and flag the regression's
  output as unreliable for high-fat foods instead of silently "fixing" it.
- **Why Berries changed from `FID: 3, est: true` to `FID: 0, free: true`**: the
  original data (from a Dr. Fiona McCulloch blog post) listed berries as a
  composition-estimated FID of 3. The later, more authoritative Bell booklet FAQ
  explicitly states berries (like salads and diet drinks) are too low in kJ to be
  tested and don't need insulin counted for them at all. The booklet source should
  generally win on conflicts — it's the primary clinical/research source, the blog
  post is a secondary popularization of the same underlying research.
- **Why some foods have two near-duplicate entries were merged**: the original
  26-food ledger (from the blog post) and the ~90-food Bell booklet overlapped on
  several items (walnuts, poached eggs, chicken, rice, apple, banana, etc). Where
  they overlapped, the booklet's version was kept (more authoritative, more precise
  gram quantities) and the blog article's version was dropped, *except* where the
  blog article had a food the booklet doesn't cover (e.g. Almond Butter).
- **Macro values are estimates, FID values are not**: this is the single most
  important accuracy caveat in the whole app. Every `fid` number in the ledger traces
  back to a real human-tested source. Every `carb`/`protein`/`fat`/`cal` number was
  estimated from general USDA-style nutrition knowledge to make the "Closest match"
  and "Macro regression" calculator tabs possible — they were not in either source
  document. If a user reports a mismatch between the app's macro estimate and a food
  label, the macros are what's approximate, not the FID.

## Persistence

Shared `storageSet`/`storageGet` helpers use the Claude artifact `window.storage` API
(`get`/`set`) when hosted as a Claude artifact, and fall back to the browser's
`localStorage` otherwise — so the tally, saved recipes, and each card's collapse state
all persist whether the file is opened via claude.ai, self-hosted, or opened directly
as a local file. Both paths are wrapped in try/catch.

## Possible next steps (not yet requested, just logical extensions)

- Add a way to edit/re-order category pills or add new foods via the UI rather than
  only via the `FOODS` array in code.
- Consider surfacing the Bell booklet's other guidance already implemented
  conceptually but not documented in-app: doubling/halving FID for different serve
  sizes, and summing ingredient FIDs for home-made meals/recipes (the Tally feature
  already does this implicitly — could add a short in-app note crediting this as the
  "official" method per the source FAQ).
- No test suite exists. The one validation done so far was manual, via a Node script
  run against the `FOODS` array to sanity-check the nearest-neighbor matcher on
  walnuts (see `CALCULATOR_METHODOLOGY.md`). Consider formalizing this as an actual
  test file if the matching logic changes.
