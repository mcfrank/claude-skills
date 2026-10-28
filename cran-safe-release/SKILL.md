---
name: cran-safe-release
description: Defensive engineering checklist for building and releasing R packages that stay on CRAN — especially packages that access internet resources (databases, APIs, hosted datasets), which historically get archived when flaky network fails R CMD check. Use when preparing a CRAN submission or release, hardening an R package against check failures, setting up R package CI, handling a non-CRAN dependency, or debugging "package was archived from CRAN" / intermittent R CMD check failures.
---

# CRAN-safe R package releases

Worked examples of everything below: langcog/wordbankr (2.0.0, accepted
Sept 2026; issues #47–52 document the process), langcog/childesr (0.3.0,
accepted Sept 2026), langcog/metalabr (1.0.0, Sept 2026 — its human review
produced "The cookbook review round" section below). Root principle:
**CRAN machines run your examples, tests, and vignette builds — any
network access on their machines is a time bomb.** A package gets archived
not for being wrong but for failing intermittently. Engineer so that a
CRAN check involves ZERO network, and user-facing network failures never
error.

## The three gates (all are needed; each catches what the others miss)

1. **Functions fail gracefully** — CRAN policy requires packages using
   internet resources to fail with an informative message, not an error.
   Wrap every network call: retry transient failures with exponential
   backoff, then `message(...)` + return `invisible(NULL)`. Every caller
   checks for NULL and returns early
   (`if (is.null(x)) return(invisible(NULL))`).
2. **Vignettes**: gate evaluation with
   `knitr::opts_chunk$set(eval = identical(Sys.getenv("NOT_CRAN"), "true"))`
   in the setup chunk. Trap: if the data source is PUBLIC, a tokenless
   CRAN machine will happily download real data — absence of credentials
   is NOT protection.
3. **Examples**: reviewers now enforce the CRAN cookbook line — `\dontrun{}`
   is ONLY for examples that truly cannot execute (missing software, missing
   API keys / interactive OAuth); anything runnable belongs unwrapped
   (< 5 s) or in `\donttest{}`. `--as-cran` executes donttest (since R 4.0),
   so donttest examples MUST be fast and fail gracefully — with gate 1 in
   place that is safe (worst case: a message + NULL, never an ERROR).
   For credential-gated examples, don't rely on `\dontrun{}` plus a
   "requires an API token / interactive OAuth login" justification:
   reviewers accept it inconsistently (childesr passed with 12 `\dontrun`;
   metalabr was asked to switch to `\donttest`; wordbankr had the same
   justification rejected). Put a credential guard at the network
   chokepoint instead (item 2 of "Cookbook review: further items"
   below) so those examples are safe as `\donttest{}`. Keep `\dontrun{}`
   only for an example that can't finish within check time, with a comment
   in the example saying why. Design for the strict reviewer. (Bundling a tiny sample
   dataset lets some examples run for real with zero network — nice bonus.)

Tests: put `skip_on_cran()` + a credential gate
(`skip_if(Sys.getenv("MY_TOKEN") == "")`) in one helper
(`skip_if_no_backend()`) used by every network test. Keep some pure tests
(fixtures + computation-only functions) so CRAN still exercises real code.

## Characterization tests for backend rewrites

When changing where data comes from: generate golden fixtures by running
the OLD package across its full argument matrix
(`data-raw/make_fixtures.R` → `tests/testthat/fixtures/*.rds`; store
"shape" summaries for unbounded pulls), then require the new backend to
reproduce them exactly (align columns, sort rows, compare nested
list-columns after sorting). This catches silent semantic changes no
spot-check finds (NA vs "" codings, factor levels, join order, column
order).

## CI (the early-warning system that replaces CRAN surprise emails)

Two jobs in `.github/workflows/check.yml`:

- **cran-simulation**: R CMD check with NO credentials AND
  `env: NOT_CRAN: "false"` on the check step — r-lib's `check-r-package`
  sets NOT_CRAN=true by default, which silently re-enables gated
  vignettes. `error-on: '"warning"'`. Runs on every push/PR: you see what
  CRAN would see, before CRAN does.
- **network-tests**: full test suite with real credentials from repo
  secrets, on push + nightly cron. Breakage arrives as a red run the next
  morning, not an archival notice weeks later.

Hardening: `LIBARROW_BINARY: "true"` + `NOT_CRAN: "true"` env on
`setup-r-dependencies` steps (runner image updates invalidate the package
cache and arrow then compiles from source for 1–2 h; this makes it ~5
min); `use-public-rspm: true`; a `concurrency` group with
`cancel-in-progress: true`. GitHub-only deps don't resolve from the
DESCRIPTION Remotes field via pak on runners — pin them explicitly in
`extra-packages` as `github::owner/repo`. Poll runs at ≥5-min intervals
(`gh run watch` long-polling exhausts the API rate limit).

## Non-CRAN dependencies

A hard Imports on a non-CRAN package fails the check outright. The
CRAN-accepted pattern:

1. Move it to **Suggests** with a runtime guard at the single chokepoint:
   `if (!requireNamespace("pkg", quietly = TRUE)) stop("install with: ...", call. = FALSE)`
   printing the exact install command.
2. **`Additional_repositories:`** in DESCRIPTION pointing at an
   R-universe: create a repo named `<org>.r-universe.dev` containing a
   packages.json listing the GitHub package — R-universe auto-builds
   binaries with zero effort from the upstream maintainer.
3. **Remove `Remotes:`** before submission (CRAN rejects it).

User experience:
`install.packages("pkg", repos = c("https://<org>.r-universe.dev", "https://cloud.r-project.org"))`.

**The binary-bake trap**: R evaluates top-level package code at *build*
time and serializes the results into binaries. If a non-CRAN dep computes
paths/env at top level (e.g. `file.path(Sys.getenv("HOME"), ".redivis")`),
every r-universe/CRAN *binary* ships with the build runner's values baked
in (`/Users/runner/...`) — OAuth succeeds but tokens never cache, every
call re-prompts. Source installs never reproduce it; only binary-installing
users hit it. Audit any dep you point `Additional_repositories:` at for
top-level `Sys.getenv`/`path.expand` before recommending binary installs.
redivis ≥ 0.12.12 computes paths lazily (verify with
`deparse(redivis:::get_redivis_dir)` showing a function, not a baked path).
For an affected dep: document `type = "source"` or env-var token auth
(bypasses the file cache); optionally add a CRAN-safe `.onAttach` check in
YOUR package that detects the foreign-HOME path and prints the fix (read
the dep's namespace with `get(..., asNamespace())`, never mutate it); and
report upstream — the fix is computing paths in `.onLoad`.

## Release checklist

- `roxygen2::roxygenize()` — stale Rd files for deleted functions are a
  check ERROR; deprecated exports get `@keywords internal` +
  `.Deprecated()`
- NEWS.md documents every deliberate behavior change
- `urlchecker::url_check()` (dead URLs are a rejection reason)
- R CMD check clean locally, on CI cran-simulation, win-builder, and rhub
- cran-comments.md: note the graceful-failure design and why any remaining
  examples are dontrun; if previously archived, say what changed
- If the data source is versioned, pin test fixtures to a version
  (`options(pkg.dataset_version = ...)`) — but beware:
  `pkgload::load_all()` sources testthat helpers, so a pin there leaks
  into interactive sessions
- Local trap: renv-enabled repos hijack `Rscript` library paths — run
  scripts with `RENV_CONFIG_AUTOLOADER_ENABLED=FALSE`

## Migration/parallel-operation pattern (changing backends)

Old version stays on CRAN untouched against the old backend; new major
version installable from GitHub for early adopters; the CRAN release of
the new version IS the flip; decommission the old backend ~6 months later
(the old CRAN version then fails gracefully with a pointer). Users who
need the old behavior: `remotes::install_version()`.

## Release pitfalls

**Workflows you forgot exist.** Fast-forwarding a long-lived branch onto
`main` woke legacy workflows whose triggers were `branches: [main]` only —
they had never run against the new code. Audit `.github/workflows/*` for
main-only triggers before the merge, and apply to *every* workflow
(pkgdown included): the GitHub-dep pin (`github::owner/repo` in
`extra-packages`), and `env: NOT_CRAN: "false"` on any check step without
credentials. r-lib's `check-r-package` defaults to `NOT_CRAN=true`; with a
gated vignette and no token that means every data chunk hits the network,
retries with backoff, and the matrix fails after *four hours* with a
confusing `object 'x' not found`. pkgdown is the exception: give its build
step `NOT_CRAN=true` **plus** the token so the site vignette shows real
output. Also `_pkgdown.yml` must list every exported function in the
reference index or pkgdown fails ("page render error" email from GitHub).

**Test graceful failure by rendering the vignette with a bad token.**
`NOT_CRAN=true REDIVIS_API_TOKEN=garbage Rscript -e 'rmarkdown::render("vignettes/x.Rmd")'`
is the single most useful pre-submission test: it found one exported
function that errored instead of returning `NULL` (a CRAN internet-policy
violation the suite missed) and a vignette whose "graceful" chain was
broken (`if (!is.null(a)) b <- ...` leaves `b` undefined; write
`b <- if (!is.null(a)) ...` so every step is NULL-safe). Render all three
modes: `NOT_CRAN=false` (CRAN), bad token, real token.

**check-standard template.** `usethis::use_github_action("check-standard")`
is the 5-OS matrix; the current template differs from older copies by
`actions/checkout@v6`, `permissions: read-all`, and
`build_args: 'c("--no-manual","--compact-vignettes=gs+qpdf")'`. Re-apply
your env/pin modifications after refreshing it. Cross-platform matrix +
`check.yml`'s cran-simulation/network-canary pair is the right set.

**`devtools::check(remote = TRUE)`** reproduces CRAN's incoming-feasibility
NOTE locally, including the archival `X-CRAN-Comment` (use *its* date in
cran-comments.md, not memory — ours was off by months) and the line
`Availability using Additional_repositories: <pkg> yes`, which is the
proof that the r-universe route resolves. `manual = TRUE` needs pdflatex.
Tools that want pandoc (urlchecker, check) find quarto's bundled one via
`RSTUDIO_PANDOC=/Applications/quarto/bin/tools/<arch>`.

**Reverse dependencies:** `tools::package_dependencies("pkg",
available.packages(), reverse = TRUE, which = "all")`. A Suggests-only
revdep with no code usage (grep its tarball) is not a blocker; say so in
cran-comments.

**`usethis::use_release_issue("x.y.z")`** creates the canonical checklist as
a GitHub issue (needs `GITHUB_PAT=$(gh auth token)` non-interactively).
Items people skip: `Authors@R` needs a `cph` (copyright holder) role;
`use_cran_comments()`; README install instructions.

**Git worktrees:** `.git` is a *file* there and `R CMD build` only excludes
the directory form — add `^\.git$` to `.Rbuildignore` or you get a NOTE.
Appending to `.Rbuildignore` without a trailing newline mangles the last
pattern; rewrite the file.

**Submission mechanics.** The portal closes for CRAN-team vacations (e.g.
Aug 5–19, 2026); devtools then 404s on
`xmpalantir.wu.ac.at/cransubmit/index2.php` and prints a stale page
comment — check the banner at `/cransubmit/` before debugging. Non-
interactive submission: `devtools:::upload_cran(as.package("."),
devtools::build("."))`. The confirmation email goes to the DESCRIPTION
maintainer, as do win-builder results — coordinate with them before
firing. Local `--as-cran` in ~30 s with `NOT_CRAN=false` is the fast
loop; the 5-OS matrix is the slow one.

**Data-backed packages, new data releases.** Keep `TEST_VERSION` as a single
constant in a testthat helper and smoke-test an unreleased data version
with `version = "next"` before flipping `current`: schema/type diffs per
table, row-count deltas, orphan keys, coding of categorical columns (a
label→code change silently NA's out `factor(levels = ...)` recoding
downstream), and whether derived tables were actually recomputed.

## CRAN human (cookbook) review

A new package now gets a human reviewer who greps the source against the
**CRAN cookbook** (<https://contributor.r-project.org/cran-cookbook/>) and
cites its sections back at you. Reply came ~11 days after the confirmation
click. Pre-audit for all three before ever submitting:

```
grep -rn "dontrun\|GlobalEnv\|globalenv\|<<-\|set.seed" R/
```

1. **`\dontrun` policing** (cookbook general_issues.html#structuring-of-
   examples): "unwrap if executable in < 5 sec, or replace with \donttest".
   See the examples gate above. Converting the tokenless functions
   to `\donttest` + keeping credential-gated ones in `\dontrun` with an
   explicit "requires API token / interactive OAuth" justification
   satisfied the request for metalabr — but the same justification was
   REJECTED for wordbankr two weeks later ("please put functions which
   download data in \donttest"). Don't rely on it. The robust answer is
   the credential guard in "Cookbook review: further items" below: it makes
   `\donttest` safe even though redivis-r launches an OAuth flow for
   public datasets (the REST API serves them tokenless; the R client's
   table download still 401s anonymously as of 0.12.14).
2. **No writes to `.GlobalEnv`** (code_issues.html#writing-to-the-
   .globalenv) — includes `load(..., envir = .GlobalEnv)` legacy-loader
   patterns, not just `<<-`. Fix: `load()` into a private
   `new.env(parent = emptyenv())` and RETURN the data like every other
   accessor (tibble + version message + a `<pkg>_release` attribute — the
   childesr/peekbankr idiom). Don't add an `envir=` compatibility argument
   for the old loading behavior unless a *living* consumer needs it: audit
   who actually depends on the side effect first (for metalabr the answer
   was nobody — dead apps, static site, broken 0.x install base), and a
   maintainer will rightly flag the hatch as "an anti-pattern and then a
   fix". Pre-acceptance review is the cheapest moment to fix API shape;
   after acceptance it's a breaking change.
3. **No hardcoded `set.seed()` inside functions** (code_issues.html#setting-
   a-specific-seed). For deterministic data pipelines (seeded imputation
   that released data depend on): make the seed a **documented argument
   with the historical default** threaded from the exported entry point,
   and save/restore `.Random.seed` around the seeded section (restoring
   the global `.Random.seed` is the sanctioned exception to rule 2):

   ```r
   if (!is.null(seed)) {
     old <- if (exists(".Random.seed", globalenv(), inherits = FALSE))
       get(".Random.seed", globalenv()) else NULL
     on.exit(if (is.null(old)) {
       if (exists(".Random.seed", globalenv(), inherits = FALSE))
         rm(".Random.seed", envir = globalenv())
     } else assign(".Random.seed", old, envir = globalenv()), add = TRUE)
     set.seed(seed)
   }
   ```

   Verify three things: characterization fixtures still byte-identical
   (default seed = old behavior), and `set.seed(42); runif(3)` gives the
   same answer with a full pipeline call in between (stream untouched).

**Resubmission mechanics**: keep the same version number (never published);
add a "## Resubmission" section at the TOP of cran-comments.md quoting each
review point and the change made; rerun the full battery (fixtures,
3-mode vignette renders, --as-cran, CI) before uploading again — a
resubmission that trips a new issue restarts the human-review clock.


## Cookbook review: further items

Same reviewer script, different verdicts — design for the strict one.

1. **`\value` on every Rd page**, including a `@keywords internal` page
   that carries exported aliases via `@rdname` (`check_db_args.Rd` was
   flagged because `connect_to_wordbank()` documents there). For
   deprecated shims say what each alias returns.
2. **`\donttest` examples RUN under `--as-cran`, on machines with no
   credentials and no browser.** OAuth-style clients (redivis-r) respond
   to "no token" by printing a device-login URL and polling forever: a
   check timeout plus a write to `~/.redivis`. Guard at your single
   network chokepoint, *before* the client is touched:

   ```r
   wb_ready <- function() {
     if (!requireNamespace("redivis", quietly = TRUE)) { message(<install cmd>); return(FALSE) }
     if (!interactive() && !credentials_available()) { message(<how to auth>); return(FALSE) }
     TRUE
   }
   ```
   `credentials_available()` mirrors the client's own check (env token,
   notebook env var, its cached-credentials file). Interactive users keep
   the browser flow; scripts, CI and CRAN get a fast graceful `NULL`.
   Add a no-network unit test (unset the env vars, `HOME` → tempdir, and
   clear the package's in-session cache first — cached tables are
   legitimately served without re-auth, which otherwise makes the test
   order-dependent). Accept the message regex for both the "no
   credentials" and the "package not installed" messages.
3. **Verify three CRAN scenarios locally** with `R CMD check --as-cran`
   on the built tarball (plain `R CMD check` works when `devtools::check`
   refuses over an unaccepted Xcode licence), reading
   `<pkg>.Rcheck/<pkg>-Ex.timings` each time:
   - credentials present: examples really run. One took **352 s** (it
     queried every instrument) — that stays `\dontrun` with a comment in
     the example saying why; total examples must stay well under 10 min.
   - `env -u <TOKEN> HOME=<empty dir> R_BROWSER=false`: every example ~0 s,
     `find $HOME -type f` empty.
   - Suggests dep **absent**: build a library holding the full dependency
     closure minus the dep (`tools::package_dependencies(..., recursive =
     TRUE)`, `install.packages(type = "binary", dependencies = FALSE)`),
     then `R_LIBS=<lib> R_LIBS_USER=<lib> R_LIBS_SITE=<lib>
     _R_CHECK_FORCE_SUGGESTS_=false R CMD check --as-cran`. CRAN does not
     install from `Additional_repositories`, so this IS their machine.
     Our CI had never run it; mirror it as a second cran-simulation job
     (no `github::` pin, `_R_CHECK_FORCE_SUGGESTS_: "false"`).
4. Cookbook items that were clean but are one grep away from a rejection:
   `cat()`/`print()` for status (use `message()`), `T`/`F`, `set.seed()`
   in functions, unrestored `options()`/`par()`, writes outside
   `tempdir()`, > 2 cores, installing packages in code/examples. And in
   DESCRIPTION: Title Case, single quotes around software/API names
   ('Wordbank', 'Redivis'), acronyms spelled out, a reference as
   `Authors (year) <doi:...>` (no space after `doi:`), a `cph` role.
