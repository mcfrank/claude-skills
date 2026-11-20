---
name: quarto-book
description: >
  Practical playbook for building and maintaining a Quarto **book** of analysis notebooks
  (.qmd) that render to a browsable HTML site and publish to Quarto Pub. Use when creating
  or organizing a Quarto book, wiring the Python/venv kernel, deciding what to commit vs
  gitignore, publishing, or setting chapter style/notation. Covers the freeze-cache traps,
  data-gating tiers (incl. human-subjects / unpublished data going to a PUBLIC url), the
  publish safety gate, and building or publishing from a cloud session.
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
if [ -x "../.venv/bin/python" ]; then
  export QUARTO_PYTHON="$(cd .. && pwd)/.venv/bin/python"
else
  export QUARTO_PYTHON="$(command -v python3)"   # cloud containers usually have no venv
fi
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
- **Name the model and its version in the banner** (e.g. "Claude Opus 5.5"), not just "an AI
  agent" or "Claude Code". Readers need to know which model produced the text.
  - Don't guess it from memory or from a product name. In a cloud session, read it from the
    session's own metadata (the `get_session` tool: report the model the session is set to and
    the one that last served a turn). Locally, use the model shown by `/status` or `/model`.
  - If more than one model or session contributed, list each. When a later session makes
    substantial edits, add its model to the banner in the same commit.
  - If a session is not permitted to write model identifiers into committed files, leave an
    obvious placeholder (`MODEL AND VERSION: fill in`), and tell the user so they can fill it
    in. Never drop the field silently.
- Pull it into each chapter with a shortcode on the line **after** the `# Title {#sec-…}`:
  `{{< include _caution.qmd >}}`. Use the *shortcode* form — `include-before-body` inserts raw
  HTML and will **not** render markdown callouts.
- `callout-important` (red) and `callout-warning` (orange) read as the strongest banners;
  `callout-caution`/`callout-note` are softer. Give each a `## Title`.
- Adding it to N chapters is a one-line script that inserts the include after the first `^# ` line
  (skip files that already have it, so it's idempotent on re-runs).

## Cloud sessions (Claude Code on the web)

A cloud container starts empty and is thrown away, and it has no browser. Everything above
still applies; these are the differences.

**Setup**
- Quarto is not preinstalled. Install the Linux tarball from the `quarto-dev/quarto-cli`
  GitHub releases (unpack to `/opt/quarto`, symlink `bin/quarto` onto the PATH), and
  `pip install jupyter` plus the project's requirements. There is usually no `.venv`, which is
  why `render.sh` above falls back to the system Python. Put the install in the environment's
  setup script if the book will be worked on again.
- Commit a `requirements.txt` so the next session can rebuild the kernel.

**Nothing survives unless it is pushed**
- Commit `_freeze/`, `_publish.yml` and any `results.json` the chapters read, and push, before
  the session ends. An unpushed render is lost.
- Data pulled during the session (API tokens in the environment, downloaded tables) will not be
  there next time. Chapters that read gitignored data cannot be re-rendered from a fresh
  clone. Use the pattern from the data-gating section: a committed script writes a small
  aggregate JSON, the JSON is committed, and the chapter renders from it alone.

**Publishing**
- There is no browser login. Ask for the Quarto Pub token through the session's secure
  credential request for the host `quartopub.com` (bearer token). Never ask the user to paste
  it into chat. The proxy then adds the token to requests, and no environment variable holds
  it, so give the CLI a placeholder so it will run:
  `QUARTO_PUB_AUTH_TOKEN=proxy-injected quarto publish quarto-pub --no-render --no-prompt`.
  Check it works first: `curl https://quartopub.com/api/v1/account` should return 200.
- **A first publish cannot be done with `--no-prompt`:** it needs `_publish.yml`, and creating a
  site is an interactive step. Either have the user do the first publish locally and commit
  `_publish.yml`, or create the site with one call and write the file yourself:
  ```bash
  curl -sS -X POST https://quartopub.com/api/v1/sites -H "Accept: application/json" \
    --data-urlencode "type=site" --data-urlencode "title=BOOK TITLE" --data-urlencode "slug=short-name"
  # -> {"id": "...", "url": "..."}; check the slug is free first: HEAD /api/v1/slugs/short-name is 404
  ```
  ```yaml
  # _publish.yml
  - source: project
    quarto-pub:
      - id: <id from the response>
        url: 'https://<username>.quarto.pub/short-name/'
  ```
  Creating a site makes a public address. Do it only when the user has asked to publish.
- **The CLI's own requests may fail behind the proxy** ("connection closed before message
  completed", "Broken pipe"), typically on image uploads and status checks, and retrying does
  not help. `publish_curl.py` in this skill's folder makes the same API calls with curl. Copy
  it next to `_quarto.yml` and run `python3 publish_curl.py` after rendering. It uploads only
  the files the server is missing.
- **A new site returns 404 for a minute or two** after the deploy reports ready. Poll the
  public address; do not publish again.
- **Verify on the public address,** not just from the exit code: fetch the index, one chapter,
  one figure PNG and one stylesheet linked from the page.
- Afterwards, a local session can publish in the normal way; `_publish.yml` keeps the address.
- The publish safety gate applies as it does locally. A request to publish the book is not
  clearance for a chapter added later that draws on sensitive or unpublished data.

## Gotchas grab-bag
- `kernel not found` → `QUARTO_PYTHON` unset (the wrapper).
- Stale figure after new data → wipe that chapter's `_freeze/`.
- Raw `$$…$$` in the output HTML is normal (MathJax renders client-side); only a render-time
  LaTeX error signals bad math.
- macOS `ls --time-style` / GNU flags fail → use `stat -f`.
- Verify a render by grepping the log for `Output created:` per file, not by trusting one
  multi-file call.
