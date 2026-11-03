# claude-skills

Mike Frank's Claude Code skills. This repo is checked out directly as
`~/.claude/skills`, so every skill here is available in every local Claude Code
session. Frontmatter `description` fields are matched against your request to
decide when to auto-load them.

## Using these in a cloud session

Add this to the cloud environment's setup script:

```bash
git clone --depth 1 https://github.com/mcfrank/claude-skills ~/.claude/skills
```

Or copy a single skill into a repo's `.claude/skills/<name>/` to scope it to that
project.

## What's here

### Stan / Bayesian model fitting

- **`gcp-large-stan`**: fit big Stan models on GCP n2d-standard-* VMs (sizing,
  SMT-aware threading, compile-flag pitfalls, disk, detached launches).
- **`stan-mixing-cold-start`**: diagnose and recover from non-convergent Stan
  fits; in family-build workflows the usual cause is warm-starting from a
  degenerate source posterior.
- **`psis-loo-chunked`**: PSIS-LOO on very large log_lik matrices without
  segfaults (chunking is exact).

### Remote compute

- **`remote-vm-gotchas`**: SSH and long-job footguns (scp into a running
  script, pkill self-matching, detached launches, `/tmp` filling up).

### Research workflow and evaluation

- **`adversarial-protocol-review`**: "what could manufacture this result?"
  review of experiments, evals, and analyses.
- **`pre-submission-review`**: required AI review of a lab paper before submission;
  read-only on the manuscript, with a findings file for the author to decide on.
- **`reproducibility-audit`**: computational reproducibility audit of a
  published paper (OSF fetch, prereg check, Quarto report template, error
  checklist, robustness toolkit).
- **`rmd-qmd-manuscript-collab`**: collaborating on human-written manuscripts
  (code, YAML, templates, bib) without touching prose.
- **`quarto-book`**: building and publishing a Quarto book of analysis notebooks.

### Data infrastructure and R packages

- **`redivis`**: Redivis datasets, uploads, tokens, clients, releases.
- **`shiny-to-datapage`**: migrating a Shiny/Django data site to Quarto +
  Observable over Redivis.
- **`cran-safe-release`**: keeping internet-dependent R packages on CRAN.

### Lab projects and datasets

- **`wordbank`**: the Wordbank project, its new Redivis/Quarto stack, and wordbankr.
- **`cdi-vocabulary-data`**: CDI measurement, Wordbank access, IRT, vocabulary growth.
- **`babyview-dataset`**: the BabyView head-camera corpus, its pipeline,
  releases, and storage tiers.

## Local-only skills

A few skills hold internal or laptop-only details (program admin, course
internals, cluster auth), so `.gitignore` keeps them out of this repo. Published
skills can likewise keep sensitive specifics in a gitignored `private.md` next
to their `SKILL.md`.

## Project-local overrides

A project's `.claude/skills/<name>/SKILL.md` takes precedence over a global
`~/.claude/skills/<name>/SKILL.md` of the same name. Use project-local for
specifics (VM names, repo paths, variant lists) and global for transferable
lessons.

## Adding new skills

Create `~/.claude/skills/<name>/SKILL.md` (a bare `.md` at the root is NOT
loaded) with YAML frontmatter:

```markdown
---
name: my-skill-name
description: One sentence saying what it covers and when to use it.
---

# Body content in Markdown
```

The `description` is the auto-match trigger. Write it as the answer to "when
would I use this?". Then commit and push; if the skill holds anything that
shouldn't be public, add it to `.gitignore` first.
