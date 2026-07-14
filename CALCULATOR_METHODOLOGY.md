# Calculator Methodology

The "Estimate a Food's FID" card has three tabs. This doc explains the reasoning
behind each, in the order they were built, because the reasoning is why the UI looks
the way it does (which tab is default, why there's a warning flag, etc).

## Tab 1 (originally the only tab): "I know the FII"

Direct application of the core formula from the McCulloch blog post:

```
FID = calories × FII ÷ 239
```

239 calories ≈ 1000 kJ, the reference portion size FII is measured against. Trivial —
just multiplication — but included because it's the base case and some users may
already know a food's FII from another source (e.g. the Sydney FII database / Marty
Kendall's "FII all foods" Tableau public visualization, which the user linked but which
couldn't be scraped for raw data — Tableau Public dashboards are JS-rendered and don't
expose a static data export via simple fetch).

## Tab 2: "Macro regression"

Added when the user wanted to compute FID from macros (protein/carb/fat) directly,
e.g. "if I have 30g of walnuts, what's the FID?" Implements the Bell et al. 2016
published equation (see `DATA_SOURCES.md` §3):

```
1. Scale carb and protein grams to a 239-cal (1000kJ) reference portion:
     scale = 239 / calories
     carb_scaled = carb_g × scale
     protein_scaled = protein_g × scale
2. FII ≈ 10.4 + 1.0 × carb_scaled + 0.4 × protein_scaled
3. FID = calories × FII ÷ 239
```

### Validation: walnuts and eggs (important — do not silently "fix" this without re-reading this section)

The user provided real tested reference values: walnuts FII = 5 (at the 240-cal
reference portion, ≈35–40g / ⅓ cup), and the Bell booklet gives walnuts FID = 4 for
¼ cup (30g).

Plugging 30g walnuts' macros (≈4g carb, 4.6g protein, 19.6g fat, 196 cal) into the
equation above gives an estimated FII of **≈17.7** — more than 3x too high.

An attempt was made to fix this with a linear fat-penalty term, calibrated so walnuts
comes out right. This was then checked against poached eggs (2 large, tested FID 14,
~9.5g fat in a 143-cal portion — also a meaningfully fatty food, just protein-forward
rather than fat-dominant):

- **Without any fat correction**: predicted FID for eggs ≈ 12 (reasonably close to the
  real 14).
- **With the walnut-calibrated fat penalty applied**: predicted FID for eggs collapses
  to ≈ 7 — now badly wrong in the *opposite* direction.

**Conclusion**: no single linear fat coefficient fits both a fat-dominant food
(walnuts) and a protein-forward-but-still-fatty food (eggs) at the same time. This
matches what the source paper itself found — fat was tested in their stepwise
regression and dropped as non-significant once carb and protein were already in the
model, presumably because fat's real relationship to insulin response depends on
macronutrient *proportions and interactions*, not a flat per-gram penalty.

**Decision made**: don't force a fudge factor that "works" for one anchor food and
breaks silently on the next. Instead:
1. Leave the published equation as-is (no fat term).
2. Add a **confidence flag** under the result: if fat is ≥55% of the food's calories,
   show a red "likely overestimate" warning; if 35–55%, show an amber "moderate
   confidence" note. Both point the user back toward tested values.
3. Build a better tool for exactly this failure mode — see Tab 3.

If this equation or its coefficients are ever revisited, **re-run this validation**
(walnuts + eggs, at minimum) before shipping a change — it's cheap to check and it's
what caught the problem last time.

## Tab 3: "Closest match" (added after Tab 2's limitations were found; now the default tab)

Rather than estimate FII from a formula, this searches the `FOODS` array for the
foods whose macro *proportions* (% of calories from carb/protein/fat — not absolute
grams) are closest to the user's input, by Euclidean distance in that 3D percentage
space. It then scales the closest match's real tested FID to the user's portion size:

```
FID_estimate = user_calories × (matched_food.fid / matched_food.cal)
```

(This is algebraically the same as computing the matched food's implied FII and
re-applying the FID formula — `FII_ref = matched_food.fid × 239 / matched_food.cal`,
then `FID = user_calories × FII_ref / 239` — the `239`s cancel, hence the simplified
form above.)

Shows the top 3 matches with a similarity score, the matched food's tested FID/portion
for reference, and the scaled estimate — deliberately not just the single "best" match,
so the user can sanity-check whether the nearest neighbor actually makes food-type
sense (macro-percentage similarity alone can't tell a lentil dish from a slice of
white bread if their carb/protein/fat splits happen to align).

### Validation

Tested by removing walnuts from the candidate pool conceptually (i.e. feeding its own
macros back in as if it were an unlisted food: 4g carb, 4.6g protein, 19.6g fat,
196 cal) and confirming the top matches were almond butter, avocado, and olive/coconut
oil — all fat-dominant, low-carb/protein foods — with estimated FIDs in the 3–5 range,
consistent with the real tested value of 4. This is a much better result than Tab 2's
regression gave (≈18) for the same input, which is why this tab is now the default.

This validation was done with a throwaway Node script (extract the `FOODS` array from
the HTML, recompute `macroPct` and nearest-neighbor distance in isolation). Worth
turning into an actual repeatable test if the matching logic changes — see
`CLAUDE.md`'s "possible next steps."

### Known limitation

Matching is only as good as the ~95-food pool it draws from. A food with no
reasonable neighbor in the ledger (unusual fermented foods, very high-fiber foods,
etc.) won't have a good match available, and the similarity score is a rough
proportional-distance signal, not a guarantee of physiological similarity — fiber
content, food matrix, and processing all affect real insulin response in ways that
macro percentage alone doesn't capture. The in-app copy already says this; keep that
caveat if the UI is reworded. See below — this was investigated and the same failure
mode as the fat correction (Tab 2) turned up.

## Investigated but not implemented: a fiber correction

Fiber isn't digested into glucose, so a food's *total* carbohydrate can overstate its
real insulin-relevant load when a lot of that carb is fiber (legumes, bran, whole
grains). This raised the question of whether Tab 2's regression (or Tab 3's matching)
should account for it. It was tested the same way the fat-penalty idea was tested —
by checking the current formula against real tested foods already in the ledger —
**before** touching any equation.

### Validation: naturally higher-fiber foods already in `FOODS`

Using Tab 2's formula (`FID = carb + 0.4×protein + 10.4×cal/239`) against each food's
own tested FID:

| Food | Tested FID | Predicted FID | Off by |
|---|---:|---:|---:|
| Navy Beans | 22 | 64.4 | +193% |
| Wholemeal Pasta | 20 | 46.2 | +131% |
| All-Bran Original | 19 | 37.1 | +95% |
| Grain Bread | 11 | 16.6 | +51% |
| All-Bran Wheat Flakes | 29 | 34.7 | +20% |
| Sultana Bran | 56 | 51.6 | −8% |
| Soy & Linseed Bread | 22 | 19.2 | −13% |
| Wholemeal Bread | 28 | 22.1 | −21% |
| Baked Beans | 33 | 20.9 | −37% |

Navy Beans and All-Bran Original are dramatically overestimated (nearly 2–3x too
high) — a plausible fiber-blindness signature, and a clean anchor case the same way
walnuts was for fat.

**But Baked Beans breaks a simple correction the same way eggs broke the fat
penalty.** Navy Beans and Baked Beans are both legume-based and plausibly similar in
fiber content, yet Navy Beans is overestimated by 193% and Baked Beans is
*underestimated* by 37% — opposite directions. The working explanation: Baked Beans
carries a sugary tomato sauce (added carbohydrate that *does* drive a real insulin
response), which offsets or exceeds whatever fiber-blunting effect is present, while
Navy Beans here is plain cooked beans with nothing masking the fiber effect. A
coefficient calibrated to fix Navy Beans would very likely make Baked Beans worse.

### Decision made

Same conclusion as the fat case: **don't add a fiber coefficient to the regression.**
No single linear correction can fix a legume that's fiber-heavy-and-plain (Navy Beans)
and a legume that's fiber-heavy-but-sauced (Baked Beans) at the same time, and there's
no way to tell those two apart from macros alone. Documented here instead so the next
person doesn't have to re-derive this.

### Caveat on this validation

This table uses food *category* (legume, bran, whole grain) as a fiber proxy, not
actual fiber grams — `FOODS` doesn't track fiber at all currently. If this is ever
revisited seriously, the real prerequisite is adding an estimated fiber field to the
ledger (same caveat as the existing carb/protein/fat estimates: reasonable
USDA-typical values, not lab-precise) and re-running a validation like this one across
several sauced-vs-plain pairs, not just Navy Beans/Baked Beans, before trusting any
correction.

### Update: confirmed directly from Bell's PhD thesis (primary source, not just this app's own validation)

The full thesis behind the Bell booklet — Kirstine Bell, *Clinical Application of the
Food Insulin Index to Diabetes Mellitus* (PhD thesis, University of Sydney, 2014;
`Bell_KJ_thesis_11.pdf`) — was obtained after the analysis above and settles the fiber
question with actual data, not just this app's own heuristic:

> "Fibre was the only nutrient assessed that showed virtually no association with the
> FII (r = 0.08, p = 0.361)" — Chapter 3, across the full combined dataset of 700
> individual FII observations.

Chapter 3 also ran a "kitchen-sink" stepwise regression including every nutrient
(carbohydrate, fat, protein, sugar, fibre) simultaneously:

```
FII = 16.2 + 1.0 Carbohydrate – 0.2 Fat + 0.3 Protein – 0.1 Sugar – 0.4 Fibre   (Equation 4, r² = 50%)
```

Despite fibre appearing in that equation with a nonzero coefficient (−0.4), the thesis
states explicitly that **carbohydrate was the only statistically significant variable**
in it — fat, protein, sugar, and fibre's coefficients did not reach significance. When
the model was re-run keeping only significant predictors, it dropped to:

```
FII = 10.4 + 1.0 Carbohydrate + 0.4 Protein   (Equation 5, r² = 49%)
```

**This is, verbatim, the equation this app already implements in Tab 2** — confirming
it's not an arbitrary published-paper pick but the thesis's own final, statistically-
refined model, already netting out fat, sugar, *and* fibre as noise once carbohydrate
and protein are accounted for. This supersedes the category-based validation above as
the authoritative reason not to add a fiber term: it was tested at source, with the
real underlying data (700 observations, not 9 category-guessed foods), and explicitly
rejected.

One nuance worth keeping in mind if this is ever revisited: a near-zero *univariate*
correlation (fibre alone vs. FII) doesn't strictly prove fibre has zero *conditional*
effect once other nutrients are controlled for — Equation 4 did include a nonzero fibre
coefficient, it just wasn't individually significant, which can happen when predictors
are correlated with each other (fibre correlates with carbohydrate). The thesis's own
conclusion, not just this app's re-derivation of it, is what makes this a closed
question rather than an open one: "a precise FII cannot be generated on the
macronutrient composition alone."
