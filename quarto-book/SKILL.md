---
name: quarto-book
description: >
  Practical playbook for building and maintaining a Quarto **book** of analysis notebooks
  (.qmd) that render to a browsable HTML site and publish to Quarto Pub. Use when creating
  or organizing a Quarto book, wiring the Python/venv kernel, deciding what to commit vs
  gitignore, publishing, or setting chapter style/notation. Covers the freeze-cache traps,
  data-gating tiers (incl. human-subjects / unpublished data going to a PUBLIC url), and
  the publish safety gate.
---

# Quarto analysis books

A Quarto **book** turns a sequence of `.qmd` notebooks into one browsable HTML site with a
chapter sidebar + search, computed inline from your code. Ideal for a project's analysis
arc (validate → results → extensions). This is the operational playbook, gotchas first.

## Layout
```
notebooks/
  _quarto.yml     # book config: chapters list, format, freeze, kernel
  _publish.yml    # publish target (written by `quarto publish`) — COMMIT it
  index.qmd       # overview (unnumbered landing page)
  00-data.qmd …   # one chapter per file, zero-padded numeric prefix
  render.sh       # wrapper that points Quarto at the project venv
  _freeze/        # execution cache — usually committed (see Freeze)
  _book/          # rendered HTML — GITIGNORE
```
Chapters are ordered in `_quarto.yml` under `book.chapters:`. Adding a chapter = add the
file + one line there (+ keep the `index.qmd` chapter table in sync).

## The kernel wrapper — do this first
Quarto must execute cells with the project **venv** Python (the one with your deps +
ipykernel). A `kernel 'python3' not found` error means this is unset. `notebooks/render.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
export QUARTO_PYTHON="$(cd .. && pwd)/.venv/bin/python"
quarto render "$@"
```
and in `_quarto.yml`: `jupyter: python3`. Render one chapter `./render.sh 07-rsa.qmd`;
whole book `./render.sh`. **`render.sh` reliably renders ONE file per call** — loop if you
need several; a multi-file arg list can silently skip files (render each, verify each
"Output created").

## Freeze — the #1 trap
`_quarto.yml` sets `execute: { freeze: auto }` so unchanged chapters skip re-execution
(fast; also lets you render without every chapter's heavy deps installed). But:
- **Freeze keys on the chapter SOURCE, not on the data its cells read.** If a cell's code is
  unchanged but the file it loads changed (new results, regenerated CSV), freeze serves the
  **stale** cached output. Fix: `rm -rf notebooks/_freeze/<chapter>` then render to force
  re-execution.
- Editing prose re-triggers that chapter (source changed) — usually fine.
- **Commit `_freeze/`** so collaborators and the publish step reuse the cache (it stores
  computed outputs incl. figure PNGs).

## Data gating & gitignore — tiers
Notebooks depend on computed artifacts. Decide per artifact:
- **Commit** the *small* things a chapter loads to render: `summary.json`, `result.json`,
  small `*.npy`, seeds, and `_freeze/`. Keeps the book clone-renderable.
- **Gitignore** the large/regenerable: raw MCMC draws (`*.npz`), sampler `samples`, plots
  the notebook regenerates inline (`**/*.png`), big built datasets (`*.pkl`).
- **One consistent rule beats piecemeal per-dir rules**, e.g.
  ```
  *.npz
  experiments/**/*.png
  _book/
  .quarto/
  *_files/
  ```
- **Prefer JSON over pickle for a notebook's committed inputs.** A `results.json` the
  notebook reads is robust; a committed `.pkl` breaks on a library/Python bump and is
  opaque in diffs.

### Sensitive data going to a PUBLIC surface — read before publishing
Publishing pushes the book to a public URL. So:
- **Never commit raw human-subjects data** (transcripts, video frames, PII), even to a
  private repo. Commit only **aggregate/derived** numbers.
- Pattern for a chapter over sensitive data: a committed script computes an aggregate
  **`results.json`** (counts, stats, vocabulary — no rows, no PII); gitignore everything
  else in that data dir with `datadir/*` + `!datadir/results.json`; the notebook renders
  from the JSON alone.
- **Unpublished collaborator data / others' results** (a dissertation talk's condition
  means, human-subjects stats) need **explicit owner clearance before public publish**. A
  generic "publish" is *not* clearance for a newly-added sensitive chapter — surface it and
  confirm. Note the caveat in the README beside the publish command, and keep it current
  after each publish.

## Publishing
```bash
cd notebooks && quarto publish quarto-pub --no-render --no-prompt   # reuse the verified _book
```
- The target (id + URL) is recorded in **`_publish.yml`** after the first publish — commit
  it so re-publishes hit the same URL. A fresh clone with no `_publish.yml` mints a NEW url.
- `--no-render` publishes the `_book` you already rendered and eyeballed (safest). Default
  re-renders (freeze keeps it quick). Uploads are **incremental** — only changed pages go.
- **Auth:** first publish needs a browser login; afterward a stored token is reused. A
  headless shell with no token can't publish — have the user run it, or authorize once
  interactively. (Don't ask for tokens/codes.)
- A publish can be **gated by a safety check** (public surface + sensitive data). That's
  working as intended: confirm the owner cleared the material, then retry — don't route
  around the gate.

## Chapter style
- **Open each chapter with the setup cell, then the math/claim** — not preamble. A short
  `## The model` / `## The proposal` block with the equations up top anchors the chapter;
  each extension then states itself as a delta on the base model.
- **Consistent notation across chapters.** Fix symbols once (e.g. lexicon ℓ, intent I_s)
  and don't overload — if α already means one thing, don't reuse it for another (use λ).
- **Computed cells for your own claims; static markdown tables for hand-labeled or external
  numbers.** Label cells (`#| label:`) and caption figures (`#| fig-cap:`).
- **Generate figures inline from committed data** so they're always current, rather than
  committing PNGs the prose refers to.
- Sensible `format.html`: `code-fold: show`, `toc: true`, `toc-depth: 3`,
  `df-print: paged`, `theme: cosmo`; `execute: { warning: false, message: false }`.

## Per-chapter disclaimer / caution boxes
When a book is AI-generated, uses preliminary data, or is otherwise not-for-publication, put a
standing banner at the **top of every chapter** — don't rely on one note in the index.
- Write the boxes once in a partial named with a **leading underscore** (e.g. `book/_caution.qmd`)
  so Quarto skips it as a standalone render and it stays out of `_quarto.yml`:
  ```
  ::: {.callout-important}
  ## AI-generated — not for publication
  The code and prose in this book were written by an AI agent (MODEL, e.g. Claude Opus 5.5)
  under human direction; not line-by-line verified; must not be published or cited as-is.
  :::
  ::: {.callout-warning}
  ## Preliminary & unvetted analysis
  Exploratory analyses of preliminary work; numbers not independently validated; check before use.
  :::
  ```
- Pull it into each chapter with a shortcode on the line **after** the `# Title {#sec-…}`:
  `{{< include _caution.qmd >}}`. Use the *shortcode* form — `include-before-body` inserts raw
  HTML and will **not** render markdown callouts.
- `callout-important` (red) and `callout-warning` (orange) read as the strongest banners;
  `callout-caution`/`callout-note` are softer. Give each a `## Title`.
- Adding it to N chapters is a one-line script that inserts the include after the first `^# ` line
  (skip files that already have it, so it's idempotent on re-runs).

## Gotchas grab-bag
- `kernel not found` → `QUARTO_PYTHON` unset (the wrapper).
- Stale figure after new data → wipe that chapter's `_freeze/`.
- Raw `$$…$$` in the output HTML is normal (MathJax renders client-side); only a render-time
  LaTeX error signals bad math.
- macOS `ls --time-style` / GNU flags fail → use `stat -f`.
- Verify a render by grepping the log for `Output created:` per file, not by trusting one
  multi-file call.
