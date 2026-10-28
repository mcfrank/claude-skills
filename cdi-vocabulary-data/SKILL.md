---
name: cdi-vocabulary-data
description: Working with MacArthur-Bates CDI vocabulary data — the instrument's measurement properties, Wordbank/wordbankr access, IRT modelling (Rasch vs 2PL) of CDI item responses, the efficiency/acceleration (xi/kappa) decomposition of vocabulary growth, and the artifact objections that get raised against any of it. Use when a task touches CDI or Wordbank data, vocabulary norms or percentiles, age of acquisition, item difficulty, CDI-based IRT or CAT, or longitudinal vocabulary trajectories.
---

# Working with CDI vocabulary data

The MacArthur-Bates Communicative Development Inventories are parent-report
checklists: a caregiver marks which of ~400–700 words their child produces (or
understands). Words & Gestures (WG, ~8–18 mo, comprehension + production) and
Words & Sentences (WS, ~16–30 mo, production) are the two English forms, with
adaptations in dozens of languages. **Every number you compute inherits the
instrument's properties**, and most disputes about vocabulary findings are really
disputes about those properties. Start here, not with the model.

## The literature you actually need

| what | source |
|---|---|
| Norms, reliability, validity, form construction | Marchman, Dale & Fenson (2021), *CDI User's Guide and Technical Manual*, 3rd ed., Brookes — the authority; cite this before any secondary source for psychometrics |
| The founding variability paper | Fenson et al. (1994), *Monographs of the SRCD* |
| Open data repository + R package | Frank, Braginsky, Yurovsky & Marchman (2017), *J. Child Language* 44 (Wordbank) |
| Book-length treatment of cross-linguistic variability | Frank, Braginsky, Yurovsky & Marchman (2021), *Variability and Consistency in Early Language Learning*, MIT Press |
| Cross-linguistic consistency in item difficulty/predictors | Braginsky, Yurovsky, Marchman & Frank (2019), *Open Mind* 3 |
| **Computerized adaptive testing (CAT)** | Kachergis, Marchman, Dale, Mankewitz & Frank (2022), *JSLHR* 65 — English and Mexican Spanish CATs; the reference for short-form/adaptive administration |
| Theoretical frame for accumulator models | Kachergis, Marchman & Frank (2022), *Curr. Dir. Psych. Sci.* 31 ("Toward a standard model") |
| Composition / lexical class effects | Bates et al. (1994) and Marchman & Bates (1994), *J. Child Language* 21 |
| Processing speed as a correlate | Marchman & Fernald (2008); Frank et al. (2026), *eLife* 14 |
| The "is the spurt an artifact of the scale?" argument | Mayor & Plunkett (2010), CogSci — "Vocabulary spurt: Are infants full of Zipf?" |

## Measurement properties that change your analysis

- **Ceiling and floor.** WS ceilings out well before 30 months for advanced
  children; WG floors near 8 months. Any growth model fitted across the full age
  range is fitting censored data at both ends. Check the proportion at 0 and at
  max before interpreting a slope.
- **The scale is ordinal, not interval.** A 100-word gain from 50→150 is not the
  same learning as 500→600. This is the single most common source of bogus
  findings — including much of the historical "vocabulary spurt" literature, which
  Mayor & Plunkett (2010) argue falls out of the Zipfian difficulty distribution
  rather than a change in the child. **If a claim depends on the shape of a
  raw-count trajectory, it is probably an artifact.** Model item responses, or
  work in a latent ability space, or at minimum check the claim survives a
  monotone rescaling.
- **Form differences are not child differences.** WG and WS have different item
  sets; the overlap is partial. Never pool raw counts across forms.
- **Parent report is a proxy** with known biases (education, birth order,
  recency). It correlates well with lab measures in aggregate; it is noisier per
  child. Treat single administrations accordingly.

## IRT on CDI items: use the Rasch model unless you need discrimination

The natural model is a Rasch/1PL: `P(produce word j) = logit^-1(theta_i - delta_j)`,
child ability minus item difficulty. The obvious upgrade is a 2PL adding a
per-item discrimination `lambda_j`.

**The 2PL fits better and changes nothing you care about.** Fitted on five
longitudinal CDI samples in three languages:

- 2PL wins LOO in every dataset, decisively (ΔELPD −1,400 to −4,400).
- Discrimination genuinely varies — SD of `log lambda` ≈ 0.24–0.34, a roughly
  2× spread from the 10th to 90th percentile — and is only weakly related to
  difficulty (r ≈ 0.05–0.22).
- But **age of acquisition is essentially identical**: r(1PL, 2PL) = 0.986–0.999,
  median absolute difference 0.13–1.47 months. Predicted trajectories move by
  ≤4 percentage points in four of five samples.

**The trap: the 2PL is much harder to sample.** In the same runs, R-hat was 1.05–1.07
for the three English samples but **1.83 for Norwegian** — badly unconverged — and
Japanese, the smallest sample, was the one where results actually shifted
(κ +28% vs +5–9% elsewhere, trajectory difference 0.15 vs 0.02–0.04). Larger item
sets and sparser data make discrimination weakly identified. **Always check R-hat
per dataset on a 2PL; do not report a pooled diagnostic.** If you only need AoA or
a growth parameter, the Rasch model gives the same answer and converges.

## The efficiency/acceleration decomposition

For growth modelling, a useful reparameterization of ability over time:

```
theta_i(t) = xi_i + kappa_i * log(t / a_0) + log H
```

- `xi_i` — **efficiency**: the child's level. Where the trajectory sits.
- `kappa_i` — **acceleration**: returns to experience. How much ability grows per
  log unit of accumulated input.
- `kappa = 1` is the meaningful null — the **pure accumulator**, in which learning
  is proportional to input and nothing about the learner changes. Fitted CDI data
  lands around 11–13.

Two things to know before using this:

**1. Cross-sectional data cannot separate `xi` from `kappa`.** One observation per
child makes "children differ in level" and "children differ in slope" generate
identical clouds. You need repeated administrations per child. This is not a power
problem; it is non-identifiability.

**2. Per-child `kappa_i` needs 3+ administrations.** With two, the per-child slope
has zero residual degrees of freedom and `sigma_b` is inflated by sparse data
masquerading as real variance. Use a 3+ filter for anything about individual
differences in slope; 2+ is fine for population-level parameters.

## The three artifact objections, and how they are answered

Anyone presenting CDI-derived growth parameters will get these. Each has a
concrete empirical answer; know which one you have run.

- **"It's the instrument."** If `kappa` only reflects how the CDI spaces its
  items, compressing the difficulty range should shrink it proportionally.
  Refitting on deliberately narrowed item sets (middle 50%, middle 25%, easy/hard
  halves) compresses difficulty SD up to 6.5× — a pure rescaling account predicts
  `kappa` falls to 1.6–9.1, i.e. most of the way to the null. Observed across 20
  narrowed fits: −3% to +29%, i.e. **it does not fall, and mostly rises**. The
  control matters as much as the test: 10 size-matched *random* subsets, which
  compress item count without compressing difficulty, move −3% to +4% and carry a
  ruler prediction of 10.4–13.5 — so the design separates "fewer items" from
  "narrower difficulty range". Run this before you need it.
- **"It's the estimator."** Fit the comparison system's own estimator to the child
  data and see whether the contrast survives. For the LM comparison this means a
  per-word 4-parameter logistic on the proportion of children *not yet* producing
  each word, with `kappa = 0.434/scale` — no IRT model anywhere. Child estimates
  come out robustly above the LM value (~1.2) but **attenuated** relative to the
  IRT fit, because pooling across children averages over heterogeneous acquisition
  curves. Correct for that with a recovery factor calibrated by simulation: generate
  data from the fitted model, run the identical pipeline, and take the ratio of
  recovered to true population `kappa`. Corrected estimates land close to the IRT
  values. Do not quote the raw pooled estimate as if it were comparable —
  the attenuation is the whole reason the correction exists.
- **"It's a power law, so the result is definitional."** Fractional return
  `gamma = alpha` is an identity for a power law; concede it. The content is not
  that returns are constant in the exponent but that the exponent is ~12 rather
  than ~1, and that a fitted `kappa = 1` model cannot reproduce the observed range
  of acquisition ages.

## Data access

- **`wordbankr`** — R client for Wordbank; `get_instrument_data()`,
  `get_administration_data()`, `get_item_data()`. Cite Frank et al. (2017).
- **CHILDES via `childesr`** for input-side frequency estimates. Word frequencies
  for CDI items usually come from parental speech in CHILDES, not from corpora of
  adult-directed language — the difference matters for AoA-vs-frequency analyses.
- **Longitudinal samples are rarer than they look.** Wordbank is mostly
  cross-sectional; the longitudinal subsets (Thal, Smith, Marchman in English,
  Simonsen's Norwegian, Hagihara's Japanese) are what support any slope claim.
  Some are unpublished datasets available through Wordbank and have **no citable
  publication** — label them like "Marchman (Wordbank)" rather than inventing a
  reference.
- **Child IDs do not always link across forms.** Verify that a longitudinal
  extraction actually joins WG to WS by child before trusting it; a silent failure
  here produces a dataset that looks longitudinal and is not, and it will show up
  as an inexplicable model anomaly rather than an error.

## Quality filtering

Longitudinal CDI series contain impossible transitions — parents re-taking a form
report large decreases, or implausible jumps. A defensible filter: exclude
administrations showing a decrease of more than 25% of productive vocabulary, or
an increase exceeding 40% of total vocabulary per month. On five samples this
removed ~1.9% of children, but was very unevenly distributed (16% in one dataset,
0.5% in another). **Report the per-dataset exclusion rate, not the pooled one** —
an uneven rate is itself a finding about data quality.
