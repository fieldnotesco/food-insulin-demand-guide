# Data Sources

This app draws on four sources. The first three feed the main `FOODS` ledger and are
used in the following priority order when they conflict (highest authority first). The
fourth is kept intentionally separate — see its section below.

## 1. Kirstine Bell patient booklet (primary source for FID values)

Uploaded to the source chat as `Pages_from_Bell_KJ_thesis_1.pdf` — pages from a
patient-facing booklet built on Dr. Kirstine Bell's University of Sydney PhD thesis
work on the Food Insulin Index, produced with Dr. Jennie Brand-Miller's group. This is
the **most authoritative source** in the project: ~90 foods with directly human-tested
FID values (average response across at least 10 people, per the booklet's own FAQ),
specific gram/serving quantities, and organized into categories (Breads, Cereals, Rice
& Pasta, Fruit, Free Vegetables, Other Vegetables, Starchy Vegetables, Dairy Products,
Meats & Chicken, Seafood, Nuts/Eggs/Meat Alternatives, Meals & Convenience Foods,
Chips & Crackers, Confectionary, Baked Goods, Sauces & Spreads, Beverages).

Every `fid` value currently in the `FOODS` array traces back to this document, **except**
Almond Butter (see source 2 below, marked `est: true`).

### Useful facts from this document's FAQ (worth preserving if the app is extended)

- FID is tested in people with diabetes; FII is the average insulin response across
  ≥10 people to a 1000kJ (239 cal) portion.
- FII **cannot** be derived from a food label alone — composition alone misses too
  many factors (nutrient type/interactions, cooking/prep, digestion). This is direct
  textual support for treating the app's "Macro regression" tab as a fallback/estimate
  only, never as equivalent to a tested value.
- **Official method for foods not in the list**: use the FID of the most similar food
  in the booklet as a guide (this is exactly what the "Closest match" calculator tab
  does). Example given in the source: pear ≈ apple, mince ≈ beef steak.
- **Official method for recipes/meals made of known ingredients**: sum the FID of each
  ingredient, adjusted for the amount used (example given: a sushi roll = rice +
  chicken/fish + avocado + cucumber). This is exactly what the "Today's Tally" feature
  already does.
- **Adjusting for different serving sizes**: FID scales linearly with quantity — double
  the serving, double the FID; halve it, halve the FID.
- **Foods with no FID value on purpose**: berries, salads, and diet products (diet
  soft drinks, diet cordials) are too low in kilojoules to be tested at the 1000kJ
  reference portion, and the booklet states you don't need to count insulin for them.
  This is why Berries and Leafy Green Vegetables are marked `free: true` with `fid: 0`
  rather than treated as untested/unknown.
- **Alcohol has no FID on purpose** (not an omission): the booklet explicitly advises
  against dosing insulin for alcohol, since alcohol suppresses the liver's glucose
  release and combining that with insulin raises hypoglycemia risk. If the app is ever
  extended to cover alcoholic drinks, do not add FID values for them — flag this as
  a deliberate exclusion, not a gap to fill.
- **Hypoglycemia treatment**: FID is for dosing insulin for food, not for treating low
  blood sugar — fast carbs (glucose tablets/jellybeans, juice, regular soft drink) are
  what's recommended for that, not FID-counted foods.

## 2. Dr. Fiona McCulloch blog post (secondary source, original 26-food ledger)

`https://drfionand.com/insulin-counting-new-dietary-system-womens-health/` — a blog
article (from *8 Steps to Reverse Your PCOS*) that introduced FID to a PCOS/women's
health audience with a smaller 26-food table and the core formula explanation
(`FID = calories × FII ÷ 239`, `FII` referenced to 239 cal / 1000kJ portions).

This was the **first source used** to build the app, before the Bell booklet was
provided. Where the two sources cover the same food, the Bell booklet's value was
kept and this source's was dropped. Two foods from this source remain in the ledger
because the Bell booklet doesn't cover them:
- **Berries** — kept, but changed from `FID: 3 (est.)` to `FID: 0, free: true` once
  the Bell booklet's FAQ explained why berries specifically don't get tested (see
  above). This is a case where source 1 corrected source 2.
- **Almond Butter** — kept as `FID: 2, est: true` (composition-estimated, not directly
  tested in either source).

## 3. Bell, Petocz, Colagiuri & Brand-Miller (2016), *Nutrients* — "Algorithms to
   Improve the Prediction of Postprandial Insulinaemia in Response to Common Foods"

Peer-reviewed paper, same University of Sydney research group. Fetched via web search
during the source chat: `https://pmc.ncbi.nlm.nih.gov/articles/PMC4848679/` (PMC4848679)
— linked directly from the "Macro regression" calculator tab's formula note. Provides
the published regression equation used in that tab:

```
FII ≈ 10.4 + (1.0 × carbohydrate g, scaled to a 1000kJ/239cal portion)
           + (0.4 × protein g, scaled to a 1000kJ/239cal portion)
```

r² ≈ 0.49 (explains about half the variance in directly-tested FII) using nutrition
composition alone. Fat was tested in the stepwise regression but wasn't selected as a
significant independent term once carb and protein were already in the model — this
is *why* the app's macro regression tab has no fat term, and why it's flagged as
running high on high-fat foods rather than "fixed" with an ad hoc fat coefficient.
See `CALCULATOR_METHODOLOGY.md` for the validation that led to this decision.

## 4. Foodstruct.com Insulin Index compilation (separate reference tool, `INSULIN_INDEX_REF`)

`insulin-index-food-list.pdf` — a 14-page, ~140-food "Insulin Index Chart" compiled by
Foodstruct.com (Victoria Mazmanyan, last updated 2021-09-20), which aggregates Insulin
Index (II) values from many individual published studies (Holt et al. 1997 AJCN
— `https://ajcn.nutrition.org/content/66/5/1264.abstract`, the foundational paper that
coined "insulin demand," linked from the app's footer as "Dr. Jennie Brand-Miller's
team" — the Bell thesis, and numerous others — each row in the source PDF cites its
own study).

**This is kept as a separate lookup, not merged into the `FOODS` ledger**, and is only
surfaced via the search-by-name field in the "I know the FII" calculator tab
(`INSULIN_INDEX_REF` array, searched by `renderFiiSuggestions()`). Reasons:

- It's Insulin Index (a relative score vs. a reference food, generally at ~250 kcal /
  1000 kJ), which is compatible with the app's existing FII formula
  (`FID = calories × FII ÷ 239`) — but it is **not** the Bell booklet's tested FID
  methodology, and different studies in the source disagree with each other by wide
  margins for the same food (e.g. Beer is cited as both 20 and 130).
- The source has no macro/portion-gram data per food, so it can't slot into the
  `FOODS` data model or the Closest Match calculator.
- Many rows are messy: several entries have no isolated value for the named food (only
  a proxy/substitute food's value, e.g. "Chicken soup: II for roast chicken is 23"), and
  several near-duplicate food names cite the same underlying number (e.g. Beef,
  Beefsteak, Flank Steak, Rib Eye Steak, Steak all cite "topside lean beef steak" = 51).

**Curation applied when building `INSULIN_INDEX_REF`**: rows were included only when the
source directly attributed a number to the named food (or an unambiguous restatement of
it, e.g. "Rye bread has an II of 73" → Rye Bread). Rows with no directly-attributed
number were dropped rather than backfilled with a proxy food's value (e.g. Cassava,
Vinegar, Molasses, Cornmeal, Common Fig, Rose Hip — all excluded). Near-duplicate rows
citing the same underlying number were consolidated into one entry with an `aliases`
list for searchability (Beef Steak cuts; White Fish species). A `note` field preserves
cases where studies disagree by a wide margin, so the number in the UI isn't presented
as more precise than it is.

**Known gap — legumes**: this source does not have an isolated Insulin Index value for
plain **chickpeas** specifically (the source PDF only gives values for Hummus = 52 and
for chickpeas eaten with Lebanese bread = 243, both compound/prepared foods, so neither
was used to stand in for "chickpeas" alone). It also has no entry at all for **white
beans** or **pinto beans** — no study in the source's citation list covers them, so
they were not fabricated. What the source *does* cover for legumes: Hummus, Kidney
Beans, Navy Beans, Baked Beans, Mixed/Four Bean Salad, Lentils, Lentil Soup, Tofu,
Peanut, and Cellophane (mung bean) Noodles — all included in `INSULIN_INDEX_REF`. This
mirrors a pre-existing gap in the main ledger too: `FOODS` only has three `legume`-
tagged foods (Baked Beans, Navy Beans, Tofu) because the Bell booklet itself doesn't
test many legumes. If chickpea/white bean/pinto bean values are needed later, they'd
have to come from a different, food-specific study — not fabricated from this source.

### Update: 7 entries added directly from the Bell PhD thesis (stronger source than Foodstruct)

`Bell_KJ_thesis_11.pdf` — the full 2014 PhD thesis behind the Bell booklet (Kirstine
Bell, *Clinical Application of the Food Insulin Index to Diabetes Mellitus*, University
of Sydney), obtained after `INSULIN_INDEX_REF` was first built. Also openly available
via the University of Sydney's repository:
`https://ses.library.usyd.edu.au/handle/2123/11945` — linked from the app (header intro
and footer citation) so users can go straight to the source. Chapter 3 (`Table 3.1`)
directly tested 26 single foods for FII, with full macro data, as part of the thesis
research itself — a primary source, not a secondary compilation like Foodstruct.

Cross-checking all 26 against the existing `INSULIN_INDEX_REF`: **18 already matched
almost exactly** (e.g. Broccoli 29=29, Sweet Potato 96=96, Couscous 84=84, Chocolate
Milk 46=46, Custard 57=57), confirming Foodstruct had itself cited this same thesis
accurately for many entries. The remaining 8 were genuine gaps; one of those (Mixed/
Four Bean Salad, thesis calls it "4 Bean Mix") turned out to already be in the array
under a different name with the identical value (34), so a confirmation `note` was
added to that entry instead of duplicating it. The other **7 were added as new
entries**, each flagged with a `note` identifying the thesis as the source rather than
Foodstruct, since it's a meaningfully stronger citation:

- Cream (plain) — 8
- Beef Sausage (grilled) — 7
- Weetbix — 41
- Tim Tam — 27
- Butter Chicken Sauce — 16
- Beef Meat Pie — 41
- Sushi (Chicken Roll) — 48

**Not fully mined**: the thesis references "Appendix 3" for the complete 121-food
original FII database that these 26 were added to (147 foods total), but that appendix
isn't present as extractable text in this particular PDF — likely bound as a separate
document in the original submission. Figure 3.3 shows a bar chart of all 147 foods by
name, but only ~51 of the bar labels extracted as readable text, and without numeric
values. So this cross-check covers the 26 directly-tested foods only, not the full 147.

## 5. Dr. Fiona McCulloch, *8 Steps to Reverse Your PCOS* (full book excerpts)

Three documents, supplied after the app was already built: two pages (160-161) scanned
directly from the book's Step 8 ("Eat a Balanced Diet") chapter, plus two short
companion documents (`The Ideal Macronutrient Structure for Your Meals` and `Sample
PCOS Meal Plans`) that appear to be her own supplementary handout material covering
the same system. Unlike source 2 above (the blog article that introduced FID to this
audience), these are the book itself — the practical "how to actually build a meal"
layer that sits on top of the FID/FII values already in the app.

This didn't add new food data. It added two pieces of practical guidance, each
surfaced as a collapsible reference (`toggleInsulinTargets` / `togglePlateGuide`) so
they don't clutter the UI by default:

- **Target insulin count per meal/snack**, by category (breakfast lower at 35-45
  because a higher-fat breakfast is linked to better metabolic outcomes for the rest
  of the day; 50-90 for other meals depending on insulin resistance/weight/activity
  level; under 15-20 for snacks) — shown next to the "Optional daily FID target" field
  in Today's Tally. Deliberately kept separate from that field rather than auto-summed
  into it: the book's numbers are per-meal, the field is a daily total, and conflating
  them would mean guessing how many meals/snacks someone eats in a day.
  - **Post-workout snack**, added later directly from the user quoting the book (not
    from the two scanned pages already in hand): "all women with PCOS should have a
    snack within 15 to 20 minutes of working out. This snack should be low in fat and
    contain carbs and protein. An insulin count of around 20-30 is best." Listed as a
    third sub-item under the Snack subheading, alongside insulin-resistant / non-
    insulin-resistant.
- **Plate-building structure** (palm-sized protein, half-plate vegetables, ~1 tbsp
  healthy fat, small carb serving prioritizing berries/sweet potato/squash over
  grains) — shown in the Recipes card, since that's where ingredients actually get
  assembled into a meal.

One thing worth noting for future consistency: the book's own stated method for
insulin counting — "add up the count of each food in your meal... substitute the most
similar food's count for anything not listed" — is exactly what the app's Closest
Match and Recipes features already do. This wasn't news requiring a code change, just
confirmation the app's core mechanic already matches the source's own methodology.

**Not incorporated**: the sample meal ideas in `Sample PCOS Meal Plans` (smoothie
template, egg dishes, quick meals, snack ideas) were left out of the Recipes feature.
Several reference an "Appendix D" for exact recipes/quantities that wasn't supplied,
and turning a named idea like "Egg Muffins with Prosciutto" into an actual ingredient
list with weights would mean guessing quantities the source doesn't give — the same
kind of fabrication this project has consistently avoided elsewhere.

## Terminology: PCOS → PMOS

In May 2026, an international expert consensus (56 organizations, 14,300+ survey
respondents, published in *The Lancet*) renamed PCOS (polycystic ovary syndrome) to
PMOS (polyendocrine metabolic ovarian syndrome) — the original name was considered
misleading since ovarian cysts aren't the defining feature and aren't always present.
ICD adoption is expected by 2028; the app was updated immediately rather than waiting.

Every running-text mention of the condition in the app was changed from PCOS to PMOS
(the eyebrow tag, the header intro paragraphs). **Exact titles of existing published
works were left as-is**, since they're direct quotes of works that were actually
published under the name "PCOS" and changing them would misrepresent the citation:
Dr. Fiona McCulloch's book *8 Steps to Reverse Your PCOS*, and her article
"Food Insulin Demand – A Dietary Metric for PCOS and Metabolic Health." If either
source is ever reprinted/retitled under the new terminology, update the citations to
match the reprint, not preemptively.

## Macro data (carb/protein/fat/calories per food) — not from any of the above

**None of the three sources above publish macronutrient breakdowns.** The `carb`,
`protein`, `fat`, and `cal` fields in the `FOODS` array were estimated by Claude during
the source chat using general USDA-typical nutrition knowledge for each food at its
stated portion size (e.g. "130g cooked chicken breast" → typical USDA cooked-chicken
macros scaled to 130g). These are reasonable estimates for the purpose of nearest-
neighbor matching and regression input, **not precise lab values**. If precision here
ever matters (e.g. someone wants to cite exact macro numbers), re-derive them from an
actual nutrition database (USDA FoodData Central is the standard free source) rather
than trusting the current numbers as authoritative — they were never meant to be.
