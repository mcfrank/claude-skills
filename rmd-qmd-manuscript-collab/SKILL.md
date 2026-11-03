---
name: rmd-qmd-manuscript-collab
description: Use when collaborating on a new research report or paper (Rmarkdown / Quarto / TeX) whose prose is exclusively human-written - code chunks, YAML, journal templates (PNAS etc.), bibliography upkeep, figure regeneration, or any other change to the manuscript's `.Rmd`, `.qmd`, or `.tex` file. Not for audits or reproductions of existing published work (see reproducibility-audit), where Claude writes the report. Hard rule: never edit prose, captions, headers, or insert/move citation markers; every change to a manuscript file goes through a PR for human review.
---

# Collaborating on a human-authored Rmarkdown manuscript

The author writes ALL prose. The agent never touches prose, captions,
section headers, or citation markers in the manuscript. Code chunks,
YAML, helper R files, rendering / template configuration, the
bibliography `.bib`, and figure regeneration are agent-allowable scope.

Scope: new research reports and papers. Audits or reproductions of existing
published work (the reproducibility-audit skill) are out of scope; there
Claude writes the report prose.

## The hard rule

In any `.Rmd`, `.qmd`, or `.tex` manuscript file, **do NOT modify**:

- Body prose (sentences, paragraphs)
- Section headers (`# Title`, `## Section`, `\section{...}`)
- Figure / table captions (e.g., `fig.cap = "..."`, `\caption{...}` — leave with empty string when scaffolding a chunk, the author fills it in)
- Citation markers in the prose (`[@author2020]`, `@key`, `\cite{...}`)
- Author lists, abstract text, acknowledgments

**Allowed in manuscript files:**

- Code chunks (their bodies and chunk options)
- YAML frontmatter (format, template, output options, knit options, params)
- Inline R expressions inside `` `r ...` `` — but only **edit existing** expressions, never **insert or move** them. The author places `` `r PLACEHOLDER` `` markers where they want computed values to land.
- Chunk labels / IDs
- Cross-reference labels (`@fig-...`, `@tbl-...`) ONLY when adding or labeling chunks the agent is creating

**Allowed outside manuscript files:**

- Anything: helper R files, plotting scripts, YAML/config, the `.bib`, Quarto extensions, templates, build scripts

## Workflow: PR-as-gate

**Any change that touches a `.Rmd` / `.qmd` / `.tex` manuscript file
must go through a PR.** Changes that don't touch a manuscript file can
land on the default branch (`main` or `master`) directly.

**Always work in a separate git worktree, never `git checkout -b` in
the author's working tree.** The author is editing the manuscript in
the main checkout in parallel; a `checkout`/branch switch there yanks
them onto your branch mid-edit. A worktree gives you an isolated
directory on your own branch while the author keeps working
undisturbed.

When a PR is needed:

1. `git worktree add ../<repo>-<branch> -b <descriptive-branch-name> <default-branch>`
   (a sibling directory on a fresh branch off the default branch), then `cd` into it
2. Make the changes; commit with clear messages
3. `git push -u origin <branch>`
4. `gh pr create --title "..." --body "..."` with a body that explicitly enumerates what changed in any manuscript files (so the human reviewer can verify on GitHub's diff view that no prose was touched)
5. Return the PR URL to the author and **stop**. Do not merge — the author reviews and merges on GitHub.
6. After the author confirms the PR is merged, clean up: `git worktree remove ../<repo>-<branch>`.

Note that re-running the manuscript's cache/build scripts inside the
worktree writes to that worktree's copy of any gitignored output
dirs (`fits/`, `outputs/`); if those are large and shared, point the
scripts at the absolute path in the main checkout rather than
duplicating them.

PR titles should be specific: "Add Bayesian-fit caching chunk for κ_pop" not "Update paper". One logical change per PR for normal work; format-conversion campaigns may batch (see below).

## Bibliography protocol

The author maintains citation markers in the manuscript. The agent
maintains the `.bib` entries that those markers reference.

**Allowed:**

- Copy bib entries from human-curated source files (typically `bibs/*.bib` in the repo) into the paper's main `.bib`
- Adjust cite keys in the paper's `.bib` to match what the manuscript uses (so the marker resolves)
- Copy bib entries that can be verified against PDFs in a local `papers/` folder, if one exists

**Forbidden:**

- Generating bib entries from training-data memory
- Inserting `[@key]` citation markers in the manuscript prose
- "Filling in" a missing reference from web search, model knowledge, or guesswork

When the author adds a citation marker and the corresponding bib entry
is missing, do NOT guess. Notify the author with the exact missing key
and ask what it should cite:

> "Citation `[@smith2024]` appears in the manuscript but has no entry
> in `standard_model.bib`. I don't have a verified source for this in
> `bibs/` or `papers/`. What should it cite?"

## Reproducibility / caching

The standard pattern:

- **Big Bayesian fits (>10 MB) are NOT in git.** They live in `fits/` (gitignored) or remote storage (Sherlock scratch, cluster volumes, OSF for archival).
- **Small summary RDS files (<5 MB)** — scalars, draws subsets, summarised parameters needed for plotting — ARE in git. The manuscript loads these in chunks.
- **Plotting and post-processing code** lives either in INLINE chunks in the manuscript or in helper R files `source()`d at the top of the manuscript. Long inline code that hurts manuscript readability → factor into a sourced helper.

A typical chunk pattern:

```{r}
#| label: fig-kappa-comparison
#| fig.cap: ""   # author fills caption

fit_summary <- readRDS("cache/io_pooled_widedelta_summary.rds")
ggplot(fit_summary, ...) + ...
```

The author may decide intermediates will be archived to OSF before
submission, with a DOI link added to the manuscript. In that case keep
working with the local cached files and trust the author to add the
OSF link.

## Format-conversion campaigns (e.g., journal templates)

Journal format conversions (PNAS, Nature, Elsevier templates) involve
many small render-test-tweak cycles. When the author grants up-front
permission to batch:

- Work on a single feature branch (e.g., `pnas-conversion`)
- Iterate freely: render, inspect output PDF, tweak YAML/template/chunk
  options, recommit
- When the conversion is stable, open ONE PR with the cumulative diff
- Note in the PR body that the manuscript prose, captions, and
  citations are untouched — only YAML, format, template, and chunk
  configuration changed
- The author reviews the whole transformation before merging

For any format change that would re-flow prose or alter wording (e.g.,
shortening the abstract to a word limit, splitting paragraphs to match
journal layout): do NOT make the change. Raise it with the author.

## Before submission

Lab AI policy requires two things before a paper is submitted. At a journal-format
conversion, a submission build, or whenever the author mentions submitting, check both and
report what's missing:

- **AI disclosure statement.** Every paper needs one, even if no AI was used. It says what
  AI was used for and, ideally, which models and versions. The statement is prose, so do
  not write or insert it; flag it if it's absent. If the author asks, list what the repo
  documents (such as `Co-Authored-By` trailers in `git log`) as notes outside the manuscript.
- **AI review.** Look for a `pre_submission_review_*.md` findings file for the current
  version, with a decision recorded on every finding. If there isn't one, offer the
  pre-submission-review skill.

## Common slips to avoid

Each of these looks harmless and crosses the prose line:

- "Adding the AI disclosure statement since it's required" — prose. Flag it; don't write it.

- "Just fixing a typo in the abstract" — prose. Don't.
- "Adding a transition sentence to make a chunk output flow" — prose. Don't.
- "Updating the section header to be more accurate" — header. Don't.
- "Adding the missing citation `[@smith2024]` since the author obviously meant Smith 2024" — citation marker, and also bib-from-memory. Don't.
- "Filling in this figure caption" — caption. Don't.
- "Editing the abstract to match the new numerical results" — prose. Don't.
- "Smoothing out the methods section while I'm here" — prose. Don't.

If unsure whether a change crosses the prose line, ask before making
it. Phrase: "Is X considered prose-territory in your workflow, or can
I touch it directly?"

## When you're done with a contribution

- For a PR-gated change: confirm the PR URL is returned, then stop.
- For a non-manuscript-touching change: commit to the default branch with a clear
  message and report what changed.
- Never assume the author has reviewed or merged a PR — wait for
  explicit confirmation.
