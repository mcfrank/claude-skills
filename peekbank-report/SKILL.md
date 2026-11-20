---
name: peekbank-report
description: >
  Spin up (or resume) a Quarto analysis report over Peekbank, the open database of
  looking-while-listening eye-tracking data, in R with peekbankr. Use when a session starts a new
  Peekbank analysis repo or picks up an existing one: environment setup in a fresh cloud
  container, the repo scaffold (knitr Quarto book, render.sh, caution banner, HANDOFF.md), pulling
  and caching a pinned release, the Peekbank schema and its traps, the standard accuracy pipeline
  (Frank et al. 2025), mixed-effects and brms modelling across datasets, and handling unpublished
  collaborator data in a public repo. Pairs with quarto-book (book mechanics) and redivis (API and
  token details).
---

# Peekbank analysis reports

A Peekbank report is a Quarto **book** of R notebooks. The first chapter pulls a pinned release with
`peekbankr` and caches it. Middle chapters derive a trial-level analysis file, which is committed so
that the modelling chapters render without a token. Last come the models and plots. A
`HANDOFF.md` carries decisions and state between sessions.

Reference implementation: `mcfrank/peekbank-wl` (novel-word learning: accuracy by number of
teaching exposures × age, 2026). Copy its `notebooks/_quarto.yml`, `render.sh`, `_caution.qmd`,
`.gitignore` and `HANDOFF.md` structure rather than starting from nothing.

Read `quarto-book` for book mechanics (freeze, publishing, data-gating tiers). That skill is written
for a Python kernel. The R/knitr differences are below.

## 1. Fresh cloud container (about 10 minutes)

The cloud image has no R or Quarto. Run this once per container, or put it in the environment's
setup script:

```bash
apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -y -qq r-base-core r-base-dev \
  libcurl4-openssl-dev libssl-dev libxml2-dev libfontconfig1-dev libharfbuzz-dev libfribidi-dev \
  libfreetype6-dev libpng-dev libtiff5-dev libjpeg-dev pandoc

# Binary CRAN packages from Posit Package Manager; the user agent is what makes p3m serve binaries
cat > ~/.Rprofile <<'EOF'
options(repos = c(CRAN = "https://p3m.dev/cran/__linux__/noble/latest"),
        HTTPUserAgent = sprintf("R/%s R (%s)", getRversion(), paste(getRversion(), R.version["platform"], R.version["arch"], R.version["os"])),
        Ncpus = 4)
EOF
Rscript -e 'install.packages(c("tidyverse","remotes","here","knitr","rmarkdown","lme4","lmerTest","broom.mixed","viridis","jsonlite"))'
# Optional, for Bayesian models (binaries, about 2 minutes):
Rscript -e 'install.packages(c("brms","rstan","posterior","tidybayes"))'

# peekbankr: remotes::install_github() fails in cloud sessions, because the GitHub API is scoped
# to the session's repos. An anonymous git clone works, so install from a local clone:
git clone --depth 1 https://github.com/peekbank/peekbankr.git /tmp/peekbankr
Rscript -e 'remotes::install_local("/tmp/peekbankr", upgrade="never", dependencies=TRUE)'

curl -sSL -o /tmp/quarto.deb https://github.com/quarto-dev/quarto-cli/releases/download/v1.6.42/quarto-1.6.42-linux-amd64.deb
dpkg -i /tmp/quarto.deb
```

Check the token before anything else:

```bash
Rscript -e 'nchar(Sys.getenv("REDIVIS_API_TOKEN")) > 0'
```

The token needs the `data.metadata` scope, and should be an environment secret, never in the repo.
Only the data-pull chapter needs it.

## 2. Repo scaffold

```
HANDOFF.md            # goal, decisions (don't re-litigate), status, gotchas, next steps
README.md             # layout, requirements, render commands
metadata/             # hand-coded design CSVs (one row per dataset × condition, with confidence + notes)
data/derived/         # small trial-level CSV(s): COMMITTED
cache/                # peekbankr pulls, model fits, unpublished data: GITIGNORED (keep cache/README.md)
notes/                # design summaries from papers or correspondence (not raw lab documents)
notebooks/
  _quarto.yml  index.qmd  _caution.qmd  render.sh
  01-data.qmd  02-….qmd  …
  _freeze/            # COMMITTED (except chapters over unpublished data)
```

**`_quarto.yml`** sets `project: type: book`. Omit `jupyter:`; knitr is picked up from R chunks.
Also set:

```yaml
execute: { freeze: auto, warning: false, message: false }
format: { html: { theme: cosmo, toc: true, toc-depth: 3, code-fold: show, df-print: paged } }
```

**`render.sh`** for R. There is no venv. The locale line matters in containers, which default to
POSIX: without it, en dashes and other non-ASCII characters in figures print as `<U+2013>`.

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export LANG="${LANG:-C.UTF-8}"
quarto render "$@"
```

Render one chapter per call (`./render.sh 03-trials.qmd`) and check the log for `Output created:`.

**Every chapter's setup chunk** calls `here::i_am()`:

```r
library(tidyverse); library(here)
i_am("notebooks/03-trials.qmd")   # _quarto.yml would otherwise make notebooks/ the here() root
```

Without it, recent `here` stops at `notebooks/` because it finds `_quarto.yml` there, and every
`here("data", …)` path breaks.

**`_caution.qmd`** holds an "AI-generated, not for publication" callout and a "preliminary"
callout. Include it after each chapter title with `{{< include _caution.qmd >}}` (see quarto-book).

**`.gitignore`** at minimum:

```
notebooks/_book/
notebooks/.quarto/
notebooks/*_files/
notebooks/*.rmarkdown        # knitr intermediate, present only while a render runs
cache/*
!cache/README.md
.Renviron
.secrets
```

## 3. Pulling data (chapter 01 pattern)

Pin the release, pull only what the question needs, and cache to one RDS:

```r
library(peekbankr)
RELEASE <- "2026.1"                       # Redivis version v1.4; record it in HANDOFF decisions
CACHE <- here("cache", "01_peekbank.rds")

if (!file.exists(CACHE)) {
  pb <- peekbank(RELEASE)
  meta <- list(datasets = pb$datasets, subjects = pb$subjects,
               administrations = pb$administrations, trials = pb$trials,
               trial_types = pb$trial_types, stimuli = pb$stimuli)
  keep <- meta$stimuli |> filter(stimulus_novelty == "novel") |> distinct(dataset_name) |> pull()
  aoi <- pb$subset$aoi_timepoints(dataset_name %in% !!keep)   # note !! for local vectors
  saveRDS(c(meta, list(aoi_timepoints = aoi, release = RELEASE)), CACHE)
}
d <- readRDS(CACHE); stopifnot(d$release == RELEASE)
```

Keep the cache out of git (it is large and regenerable). Delete it to force a fresh pull.

About the client:

- **`pb$subset$<table>(…)`** filters server-side for timepoint tables (only SQL-translatable
  predicates) and in R for small tables. Predicates may use columns from linked tables, e.g.
  `pb$subset$aoi_timepoints(age >= 24, vanilla_trial, !excluded)`.
- **`t_norm` can't be filtered in `subset$aoi_timepoints()`**, because the table is transferred
  run-length encoded. Filter after fetching, or pass `rle = FALSE`, which is slow.
- **`pb$db$<table>`** gives dbplyr handles for server-side summaries, run with `collect()`.
- **Anonymous REST** for a quick look without a token, small tables only:
  `https://redivis.com/api/v1/tables/datapages.peekbank:a3v0:v1_4.<table>/rows?maxResults=…`.
  Results over 100 MB (any timepoint table) and queries need auth.

## 4. Schema essentials and traps

Tables in release 2026.1, with the columns you will actually use:

| table | key columns |
|---|---|
| datasets (44) | dataset_id, dataset_name, shortcite, cite |
| subjects | subject_id, sex, native_language, lab_subject_id, dataset_name |
| administrations | administration_id, subject_id, age (months), coding_method, tracker, dataset_name |
| trials | trial_id, trial_order, trial_type_id, excluded, exclusion_reason, dataset_name |
| trial_types | trial_type_id, full_phrase, condition, target_id, distractor_id, vanilla_trial, point_of_disambiguation, dataset_name |
| stimuli | stimulus_id, stimulus_novelty (familiar/novel), english_stimulus_label, original_stimulus_label, dataset_name |
| aoi_timepoints | administration_id, trial_id, aoi (target/distractor/other/missing), t_norm (ms from point of disambiguation, 25 ms steps) |

Traps found in practice:

- **`trials` has no `administration_id`.** The trial-to-child link is only in the timepoint
  tables. Recover it with `aoi |> distinct(administration_id, trial_id)`.
- **Trials can be missing from Peekbank entirely.** Gaps in `trial_order` within an
  administration usually mean a trial was dropped at import for lack of gaze data. Anything that
  counts earlier trials (exposures, position in session) undercounts. In peekbank-wl's chapter 02, a
  gap is filled from donor administrations with an identical trial list (exact match on every
  shared position, at least 80% overlap, all donors agree). Trailing gaps are never filled.
- **Same-age repeat administrations** (e.g. visit A and visit B at one age) have no session
  order field. `administration_id` order matched content in every case checked
  (fernald_marchman_2012). Verify for your datasets.
- **`condition` and `full_phrase` carry design information** that the stimulus columns don't. For
  example: incidental labels inside a phrase ("the baby on the deebo"), or a novel word whose
  target is a familiar object. Some datasets have generic phrases instead of the real sentences
  (adams_marchman_2018), so this can't be parsed for every dataset.
- **Novelty coding errors:** fernald_totlot ME trials code the distractor as "novel" because the
  import set it from the target label. Mispronunciation datasets (swingley_aslin_2002,
  moore_bergelson_2022_verb) use "novel" for mispronounced familiar words. Always check what
  `stimulus_novelty == "novel"` actually covers before trusting it.
- **Some teaching isn't in the data at all:** learning videos, habituation phases, un-imported
  trials. Code it by hand from the papers into a `metadata/external_teaching.csv` with a source
  column.
- **Design spreadsheets and dissertations from labs** are the ground truth when papers are vague.
  Summarize what you used in `notes/`, and don't commit the lab's own documents.

When you find an import bug, record it in HANDOFF as a candidate issue on
`peekbank/peekbank-data-import`. Import scripts live there under `data/<dataset_name>/import.R`.
Reading an import script is often the fastest way to learn what a column means for that dataset.

## 5. Standard measures (Frank et al. 2025, eLife)

The best-practice reference is *Continuous developmental changes in word recognition…* (Frank et
al., 2025, eLife). Its code is in `peekbank/peekbank-development` (`1_tidy_data.qmd`).

```r
acc <- d$aoi_timepoints |>
  filter(t_norm > 200, t_norm <= 4000) |>                     # long window; short = ≤ 2000
  mutate(correct = case_when(aoi == "target" ~ 1, aoi == "distractor" ~ 0)) |>
  summarise(long_window_accuracy = mean(correct, na.rm = TRUE),
            long_window_prop_data = mean(!is.na(correct)),     # denominator = samples recorded
            .by = c(administration_id, trial_id)) |>
  mutate(long_window_accuracy = if_else(long_window_prop_data >= .5 &
                                          !is.nan(long_window_accuracy),
                                        long_window_accuracy, NA))
```

Then drop trials with `excluded == TRUE` and trials with NA accuracy. Report an exclusion table by
dataset. Decide explicitly, and record in HANDOFF, whether to apply a minimum number of trials per
child (peekbank-wl uses none), and whether to keep bilingual children.

Write the trial-level file (IDs, dataset, age, condition codes, predictors, accuracy) to
`data/derived/*.csv` and commit it. Peekbank data are public, and this file lets everything
downstream render without a token or the large cache.

## 6. Modelling across datasets

- **Age:** use `log(age / 24)`, so the intercept and lower-order terms are at 24 months.
  Count predictors: `log(n + 1)`.
- **Baseline lmer:**
  ```r
  acc ~ x * log_age + (1 | dataset_name) + (1 | subject_id) + (1 | target_label)
  ```
- **Between-dataset confounding is the main threat.** When the predictor varies mostly *between*
  datasets, the pooled slope mixes in procedure, stimulus and lab differences. Two checks:
  - Fit a within-dataset version: dataset-mean-centre the predictor, or use dataset fixed effects.
  - Fit a brms model with **dataset random slopes**:
    `(1 + x * log_age | dataset_name)`. lmer usually fails to converge with these when data are
    sparse; brms with regularizing priors handles them.
  In peekbank-wl, a within-dataset lmer interaction with p = .047 shrank to a credible interval
  spanning zero once dataset slopes were allowed. Report both.
- **brms settings that worked** (Gaussian on proportion scale):
  - Priors:
    - `normal(0.6, 0.2)` intercept
    - `normal(0, 0.2)` b
    - `exponential(10)` sd
    - `lkj(2)` cor
    - `exponential(5)` sigma
  - Sampler: `adapt_delta = .99`, 4 chains × 3000 iterations (1000 warmup), `seed`.
  - `file = here("cache", "<chapter>_<name>")`, `file_refit = "on_change"`.
  - At `adapt_delta = .95`, there were a few divergences. At .99, there were none.
  - Report max R-hat, min ESS and the divergence count in a diagnostics chunk.
- **Timing confounds** in within-session designs: trial position, test number, and trials since
  teaching are often collinear with exposure count. Add them as controls and say plainly when
  they can't be separated.
- **Plots:**
  - Show model predictions (lines with credible or confidence bands) over the predictor range
    observed near each age, with observed child means as unconnected points. Don't connect
    descriptive means across sparse cells.
  - Drop cells with fewer than about 5 children.
  - Also useful:
    - the predictor's slope as a function of age, with dataset-specific slopes overlaid
      (filled points for datasets that vary the predictor, open for those that don't);
    - per-dataset facets of predicted curves.

## 7. Long renders in cloud sessions

A brms fit with dataset slopes on about 6k trials (800 children) takes about 20 minutes on 4 cores. A chapter
with two or three fits exceeds the default 30-minute background-task limit and gets killed.
Mitigations:

- Run long renders in the background with a long timeout (up to 2 hours).
- Cache fits with `file =` so a re-render after a kill reuses the finished ones.
- Fit one model per chapter where possible.
- Delete the cached `.rds` when the model or data change. `file_refit = "on_change"` catches
  formula, data and prior changes, but not changes upstream of the data frame you pass.
- Freeze keys on chapter source, not on data. After editing `metadata/*.csv` or regenerating
  `data/derived/`, delete `notebooks/_freeze/<chapter>` for every downstream chapter and re-render
  in order.

## 8. Unpublished or collaborator data in a public repo

Peekbank analysis repos are usually **public**. When a lab shares a dataset that isn't in Peekbank
(raw iChart files, design documents, dissertations):

- Put the raw files and your notes on them in gitignored `cache/<study>/`.
- Give the study its own chapter that is gated on the file's existence:
  ```r
  RAW <- here("cache", "study", "raw.txt"); HAVE_DATA <- file.exists(RAW)
  knitr::opts_chunk$set(eval = HAVE_DATA)
  ```
  Open the chapter with a callout saying the data are unpublished and local-only.
- Gitignore that chapter's `_freeze/<chapter>/` and any derived files. Keep its numbers out of
  committed files, HANDOFF.md included. Describe the chapter, not its results.
- Commit nothing from it until the owners clear sharing. A Peekbank import is a separate,
  later step.
- Fernald-lab iChart format:
  - One row per trial.
  - 33 ms frame columns, with frame `F0` at word onset.
  - Codes: `1` target, `0` distractor, `.` and `-` away or off, blank = no frame (not missing data).
  - A `PSO?` column marks lab prescreening, treated like Peekbank `excluded`.
  - Check column positions with a `stopifnot()`.

## 9. HANDOFF.md discipline

Sessions are ephemeral, so HANDOFF.md is the project's memory. Keep these sections current, and
commit and push after each chunk of work:

- **Goal**, and the release and paper the analysis follows.
- **Decisions made with the PI (do not re-litigate):** numbered. A new session reads these first
  and does not reopen them.
- **What exists:** chapters, metadata files, and what each role or code means.
- **Environment setup:** the block from section 1, adjusted for the project.
- **Gotchas learned:** dataset-specific traps like those in section 4.
- **Source-paper coding status:** which datasets are coded from which source, with confidence,
  and a hit list of what is still missing. Name who is being asked.
- **Dated status entries:** results with numbers (public data only) and what changed.
- **Next steps:** numbered, with completed items struck through.

Also commit the questions you'd send to data owners as paste-ready text. The PI may be unable to
open HANDOFF from a phone, so give the text in chat too.
