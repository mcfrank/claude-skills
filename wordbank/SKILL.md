---
name: wordbank
description: >
  Orientation to the Wordbank project (langcog / Mike Frank): what it is
  scientifically (CDI parent-report vocabulary data, ~121k administrations, 50
  languages), the data model and its semantics, the new infrastructure (Redivis
  dataset datapages.wordbank, the Quarto/Observable site wordbank-datapage on
  GitHub Pages, wordbankr 2.0), the legacy stack being decommissioned
  (Django/MySQL on AWS, Shiny apps), where every repo lives on this machine, how
  to access and release data, and the people and open threads. Load whenever a
  session touches Wordbank, wordbankr, the CDI datapage site, the Redivis wordbank
  dataset, or the old wordbank.stanford.edu stack. Pairs with cdi-vocabulary-data
  (measurement/IRT), shiny-to-datapage (the migration playbook and its traps),
  redivis (client/API mechanics) and cran-safe-release (package release gates).
---

# Wordbank

Wordbank is an open database of children's vocabulary development, built on
the MacArthur-Bates Communicative Development Inventories (CDIs) — parent-report
checklists of the words a child produces (and, on infant forms, understands).
Researchers worldwide contribute their CDI administrations; Wordbank archives
them in a common schema and provides interactive analyses, downloads, and an R
package. Everything is open (CC-BY; a few datasets CC-BY-NC, flagged per
dataset).

Cite: Frank, Braginsky, Yurovsky & Marchman (2017). Wordbank: An open
repository for developmental vocabulary data. *JCL 44*(3).
doi:10.1017/S0305000916000209. Book-length synthesis: Frank et al. (2021),
*Variability and Consistency in Early Language Learning: The Wordbank Project*,
MIT Press (wordbank-book.stanford.edu; code in langcog/wordbank-book).

For instrument psychometrics, IRT modelling, and the standard artifact
objections, see the `cdi-vocabulary-data` skill — this one is about the
project and its infrastructure.

## Scientific shape of the data

- **Forms.** Words & Gestures (WG: ~8–18 mo, comprehension + production) and
  Words & Sentences (WS: ~16–30 mo, production only), plus short forms and
  language-specific variants. An **instrument** = language × form (108 as of
  v3.0). Never pool raw counts across forms — item sets differ.
- **Scale (v3.0, released 2026-09-20):** 120,878 administrations, ~104k
  children, 50 languages, 108 instruments, 180 contributed datasets, 63.9M
  item responses (v2.0 was 105,290 / 42 / 89 / 146 / 55.9M). Largest
  instruments: English (British) and English (American) WS (~7.5k admins ×
  ~680 items); many languages have a few hundred children from one lab.
  Sparse instruments (e.g. Spanish (Peruvian) WS, 105 admins, single dataset)
  produce jagged plots — that is the data, not a bug; the site now labels it.
- **Mostly cross-sectional.** Longitudinal subsets are few and precious; child
  IDs do not always link across forms. See cdi-vocabulary-data for the traps.
- **Uni-lemmas** are cross-linguistic conceptual glosses linking items across
  languages (maintained in langcog/update_unilemmas; policy doc there). They
  power the cross-linguistic analyses and `get_crossling_*`.
- **Norms.** The site's "norms" are empirical quantiles of the convenience
  sample (LOESS-smoothed, cells <10 children hidden) — descriptive, not
  clinical. Clinical percentile scoring uses the official norming tables
  (English: Marchman et al. 2023 manual, GAMLSS, census-weighted; Mexican
  Spanish: Jackson-Maldonado et al. 2003), ported into the site's CDI Scoring
  page. The old GAMLSS norm curves were deliberately dropped from the site
  (Mike's lab now favours a Rasch view, theta(t) = a + b·log(age); parametric
  norms are a future item).

## Data model (Redivis `datapages.wordbank`, current = v3.0)

Dataset: https://stanford.redivis.com/datasets/627v-9ewzpdvz0 (the old
redivis.com/datapages/... URL is dead). Qualified reference `wordbank:627v`.
Versions are immutable, citable snapshots; **v1.1 is known-bad (doubled rows
from a Redivis merge-strategy trap) — never use it**. v1.5 masked
fully-mirrored WS comprehension per dataset and validated exposure
proportions; v2.0 normalized the schema and typed columns; v3.0 (Sept 2026)
added 37 instruments / 34 datasets, renamed 17 instruments (form names
respaced: `Form 2A`, `WG Short`, `CDI 2`, `Oxford Short`; `German` →
`German (German)`), imported birth variables, restored WS comprehension
where genuinely measured, relabelled race `Other` → `Other/Mixed`.

| table | grain | notes |
|---|---|---|
| `instruments` | language × form | `age_min`/`age_max` define the normed range |
| `datasets` | contributed dataset | carries `license` (CC-BY vs CC-BY-NC), citation |
| `administrations` | child × test (`data_id`) | age (months), production, comprehension, `is_norming`, `in_age_range`, demographics |
| `children` | child | sex, birth order, caregiver education, ethnicity, birth info |
| `language_exposures`, `health_conditions` | unnested from administrations | exposures validated to [0,100] |
| `items` | item × instrument | `item_kind` (word / gesture / …), `item_definition`, `category`, `lexical_category`, `uni_lemma` |
| `item_responses` | administration × item (long, 63.9M) | see semantics below |
| `item_summaries` | instrument × item × age × measure → proportion | powers trajectories |
| `uni_lemma_summaries` | language × uni_lemma × age × measure | powers cross-linguistic |
| `vocab_summaries` | instrument × measure × age → quantiles | powers norms |
| `aoa` | item-level age-of-acquisition estimates (`fit_aoa`, word items only) | powers networks |
| `item_embeddings` | Gemini `gemini-embedding-001` vectors for 30,635 word definitions (JSON string on Redivis) | powers networks |

**Response semantics (do not flatten):** `value` `""` = asked-and-negative,
`NA` = missing; `produces`/`understands` are `NA` where undefined (non-word
items; `understands` on WS). The legacy wordbankr API depended on these and
the ETL was once caught flattening them — the characterization tests exist to
catch exactly this.

**Known data issues:** the French (French) Tsuji produces/understands
reversal (langcog/wordbank#348) is FIXED in v3.0 but present in v2.0 and
earlier (`data_id >= 390958`; exclude or flip if pinning old versions). Open
in v3.0: 45 Dimitrova administrations reference child_ids missing from
`children` (no demographics; langcog/wordbank-import issue). Alvin Tan
(alvinwmtan) owns import-side data issues; Mika owns wordbankr/CRAN.
Diagnostic pattern worth reusing: per-dataset production-vs-age correlation
and easiest-word production rates.

**Adding data** now goes through Alvin Tan's `langcog/wordbank-import`
pipeline (raw CSVs + `datasets.csv` manifest → Redivis NEXT directly; the
MySQL route via `etl/extract_wordbank.R` is legacy). The derived tables are
then rebuilt FROM Redivis in wordbank-datapage: `fetch_redivis_tables.R next
--all` → `fetch_item_responses.R next` (~1 h) → `compute_derived.R`,
`compute_aoa.R`, `compute_embeddings.R` → `upload_redivis.R --tables ...`
→ audit → release → `write_site_slices.R` → commit slices → push. Old flow:
`etl/upload_redivis.R` → new Redivis version (set
`upload_merge_strategy = "replace"` on every table before upload; the
create-time setting silently doesn't stick). Release via the API (token has
release scope) or the Redivis UI. **Uploads within one unreleased version
APPEND** — delete a table's existing uploads before re-uploading it
(`upload_redivis.R --tables` does, and refuses files that halve the rows or
blank a populated column). Pre-release audit checklist: row counts + one
upload per table, key uniqueness incl. (data_id, item_id), orphans,
categorical labels vs wordbankr's factor levels (mismatches silently become
NA), derived tables recomputed, production == sum(produces), and **no
administrations with zero answered word items** (the sum check passes
trivially when both are 0 — v3.0 shipped Elison's 1,629 empty WG forms this
way, and since the site counts missing as "doesn't produce", every
English (American) WG trajectory dipped at 17–18 months; wordbank-import
issue). When a curve does something odd, decompose it by dataset first.
Totals-based checks cannot catch **item misalignment**: in v3.4 Elison's
totals were plausible but answers were attached to the wrong items (top
"words" at 17 mo: star, yes, kitty, tiger). The detector is a per-item
correlation of a new dataset's production proportions (same instrument, same
age band) against an established dataset: real samples correlate ~0.9+
(even Edgin's Down syndrome sample: 0.94), Elison was 0.01. Also note the
v3.x import codes "asked, not known" as `value = NA` + `produces = FALSE`
(legacy used `""`) and turns missing answers into FALSE, so "all-NA value"
does NOT mean missing any more (langcog/wordbank-import#9). Derived tables
are NOT recomputed by Alvin's release pipeline — they must be rebuilt and
uploaded into the NEXT version before release (released versions are
immutable).

## Infrastructure — new (live)

- **Site:** https://langcog.github.io/wordbank-datapage — Quarto website with
  the `datapage` extension, Observable JS + Plot, deployed by GitHub Actions
  (`.github/workflows/publish.yml`, "Render and publish", ~6 min; a runner
  image update can make it ~20 min while the R package cache rebuilds).
  **Push to `main` = deploy.** wordbank.stanford.edu now 307-redirects here
  (Stanford-side redirect, path+query preserved with a harmless `//`);
  `analyses.qmd` maps legacy `/analyses?name=<app>` deep links to the new
  pages; `/publications` and `/stats` are aliases.
- **Pages:** home (stats + the language "snail" — the original D3 v3
  `layout.pack`, loaded directly because v3's UMD won't `require()`), Vocabulary
  Norms, Item Trajectories, Cross-Linguistic Trajectories, Semantic Networks,
  CDI Scoring, Data (unified by-child / by-word / child-by-word downloads with
  shared filters; long or wide), Contributors (built from the `datasets`
  table), Blog (static snapshot of the old Blogger posts), About (team +
  data-driven publications from `resources/publications.json`), FAQ.
- **Data tiers in the browser:** render-time embeds (instruments, stats) /
  lazily fetched per-instrument slices in `slices/` (admins CSV, responses
  parquet queried with DuckDB-WASM, uni-lemma and network JSON) / everything
  else on Redivis. Response parquets are committed to git (13 MB); other slices
  are regenerated by `etl/write_site_slices.R` in CI from Redivis.
- **Secrets:** `wordbank-datapage/.secrets` holds `REDIVIS_API_TOKEN` and
  `GEMINI_API_KEY` (never print values; repo secrets are set by Mike). Load
  with `set -a; source .secrets; set +a`.
- **Local dev:** `~/Projects/wordbank/wordbank-datapage`; render with
  `RENV_CONFIG_AUTOLOADER_ENABLED=FALSE quarto render <page>.qmd` (renv's
  autoloader breaks Rscript in these repos); preview server config
  `.claude/launch.json` (name `site`, python http.server on `_site`, port
  8731). Render-time R reads `data/*.parquet` if present, else Redivis.
- **Style truth:** the old site was Bootswatch **Spacelab** (Open Sans, hero
  63px), not Cosmo. Primary blue is `#1f6fd0` (nudged from `#2780e3` for WCAG
  AA both ways). Accessibility issue wordbank #190 is closed.

## Infrastructure — legacy (being decommissioned)

- Django 3.1 + MySQL (AWS Elastic Beanstalk + RDS) at the old
  wordbank.stanford.edu, Shiny apps (langcog/wordbank-shiny: admin_data,
  instrument_data, item_data, item_trajectories, networks, scoring,
  uni_lemmas, vocab_norms) embedded by iframe from a hard-coded EC2 IP.
  Repo langcog/wordbank (local checkout on branch `wordbank2`; default branch
  `master`; README now points to the new stack). The MySQL DB remains the
  source of truth for *ingest* until the import pipeline is replaced.
- **Decommission plan (Mike, Aug 2026):** Shiny box soon after the URL
  cutover (done); EB/RDS on a longer clock — the archived wordbankr 1.x on
  CRAN still points at RDS, so ~6 months after the 2.0 CRAN release is the
  natural horizon. Users needing 1.x semantics: `remotes::install_version()`.

## wordbankr (R package)

- **2.0** reads Redivis directly; same public API as 1.x
  (`get_instruments`, `get_datasets`, `get_administration_data`,
  `get_item_data`, `get_instrument_data`, `get_crossling_items/data`,
  `summarise_items`, `fit_aoa`, `fit_vocab_quantiles`) plus `get_aoa`,
  `get_embeddings`, `wb_dataset`. **Every data function takes
  `version = "current"`** (e.g. `version = "v2.0"`) and stamps the resolved
  version in a `dataset_version` column; the old
  `options(wordbankr.dataset_version=)` is gone. Deliberate API deltas are in
  NEWS (Mika signed off, issue #51).
- Install: `remotes::install_github("langcog/wordbankr")` (main is 2.0.0);
  the Redivis client comes from the langcog R-universe:
  `install.packages("redivis", repos = c("https://langcog.r-universe.dev", "https://cloud.r-project.org"))`.
- **CRAN status:** 1.0.3 was archived 2024-04-04 for internet-access policy
  violations. 2.0.0 (main fast-forwarded, old main tagged `v1.0.3.9001-mysql`) **has
  been submitted to CRAN** (confirmed 2026-10-06); check the CRAN package
  page or incoming queue for acceptance before assuming either way. Mika
  (maintainer, mika.br@gmail.com) must click CRAN's confirmation email for
  any resubmission; submit with `devtools::submit_cran()` from
  `~/Projects/wordbank/wordbankr-redivis`. After acceptance: tag v2.0.0,
  close #44/#46/#47/#52, announce. Some data-import cleanup also waits on
  Mika.
- CI has three workflows: `check.yml` (the real gates: cran-simulation +
  network canary, on push to main/redivis + nightly) plus the legacy
  `R-CMD-check.yaml` (5-OS matrix) and `pkgdown.yaml`, which trigger only on
  `main` — all three must pin `github::redivis/redivis-r` in
  `extra-packages` (pak can't resolve the Suggests dep otherwise), and any
  job without credentials must set `NOT_CRAN: "false"` on the check step
  (r-lib's default `true` re-enables the vignette's network chunks, which
  then retry for hours and die); pkgdown gets `REDIVIS_API_TOKEN` so the
  site vignette renders with real output.
- Safety design (see `cran-safe-release`): zero network in CRAN checks
  (skip_on_cran helper, NOT_CRAN-gated vignette, `\dontrun` examples),
  graceful failure via `wb_try`, characterization tests against fixtures from
  the MySQL-era package (55 tests), CI = cran-simulation + nightly network
  canary (needs `REDIVIS_API_TOKEN` repo secret).
- Local checkouts: `~/Projects/wordbank/wordbankr-redivis` (worktree, branch
  `redivis` — the working branch; main mirrors it) and
  `~/Projects/wordbank/wordbankr` (Mike's checkout on `bilingual`; don't
  touch). Run R with `RENV_CONFIG_AUTOLOADER_ENABLED=FALSE`; `quarto pandoc`
  lives at `/Applications/quarto/bin/tools/aarch64` when a tool wants pandoc.

## Repo map (`~/Projects/wordbank/`)

`wordbank` (legacy Django), `wordbank-datapage` (the site), `wordbank-shiny`
(legacy apps; `apps/scoring` holds the norming tables the site ported),
`wordbankr-redivis` / `wordbankr` (package), `wordbank-book`, `wordbank-paper`
(the 2017 JCL paper), and analysis repos behind the old "reports":
`aoa_prediction`, `vocab-comp`, `crosslinguistic`, `demo-vocab`, `momed_vocab`,
`vocab_correct`. Migration playbook with every trap:
`~/Projects/shiny-to-observable-advice.md` (also the `shiny-to-datapage` skill).

## People and channels

Mike Frank (PI, mcfrank@stanford.edu), Mika Braginsky (wordbankr maintainer,
site co-architect), Virginia Marchman (CDI norms; MB-CDI Advisory Board),
Alvin Tan (handling the restricted-dataset approach, wordbank #325), Dan
Yurovsky. Public contact: wordbank-contact@stanford.edu. Issues: langcog/wordbank
(legacy + data), langcog/wordbank-datapage (site), langcog/wordbankr (#47 is
the 2.0 tracking issue).

## Working conventions that have bitten sessions

- Parallel Claude sessions have worked in the same datapage checkout: stage
  files selectively (`git add <file>`), never `git add -A`; pull before edits.
- OJS: object literals need parens (`({...})`); any cell syntax error kills the
  whole page silently; query strings break OJS resource resolution; Quarto
  hoists the first `<h1>` even inside raw HTML blocks; the datapage extension
  forces `mainfont` (override in `_quarto.yml`).
- Browser-pane testing: Observable's runtime schedules on
  `requestAnimationFrame`, so a hidden preview pane freezes every OJS page —
  front the tab before judging anything; synthetic events don't drive
  `viewof` inputs; trust screenshots over DOM queries.
- Git worktrees: `.git` is a *file*, which `R CMD build` doesn't exclude —
  `.Rbuildignore` needs `^\.git$`.
- Redivis: releases take minutes to propagate for big tables; version tags
  print as `v1_4` but resolve as `"v1.4"`; `gh run watch` exhausts the API
  rate limit — poll every few minutes instead.
