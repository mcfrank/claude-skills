---
name: shiny-to-datapage
description: Playbook for migrating a Shiny/Django data site to a static Quarto + Observable site backed by a versioned Redivis dataset (the "datapage" pattern). Use whenever a session involves converting, porting, or rebuilding a lab data site (shiny app, database-backed site) onto Redivis/Quarto/OJS, building lazy-loading in-browser visualizations over Redivis data, staging a legacy MySQL database (including multi-version history) to Redivis, or re-importing a dataset from an auth-walled upstream source. Covers architecture tiers (incl. hash-sharded DuckDB for high-cardinality lookups), OJS/Quarto gotchas, Redivis upload/release traps, legacy-DB staging at scale, upstream re-acquisition and parser validation, CI/deploy, preview-repo team QA, AI-derived tables, and companion R package migration (characterization tests, CRAN safety, the R binary-bake trap).
---

# Shiny → Redivis datapage migration

Read the full playbook FIRST — it encodes hard-won lessons from the
wordbank and childes-db migrations and will save hours on each of these:

**~/Projects/shiny-to-observable-advice.md**

Reference implementations (every pattern, working):
- https://github.com/langcog/wordbank-datapage (small-data patterns, AI
  tables; live: langcog.github.io/wordbank-datapage)
- langcog/childes-db-website branch `datapage` (big-data patterns:
  hash-sharded DuckDB frequency/bigram lookups, preview-repo QA) with
  langcog/childes-db `redivis/` + `pipeline/` (multi-version MySQL→Redivis
  staging; auth-walled upstream mirror + chatter parser + QA harness)
- Companion packages: langcog/wordbankr branch `redivis` (issues #47–52)
  and langcog/childesr branch `redivis` (characterization suite that
  caught NULL-in-IN and collation drift; binary-bake mitigation).

## Process that worked (follow this order)

1. **Survey** the old site/apps in parallel (what each page computes, where
   data lives, what model fitting exists — only true fitting blocks static
   migration; everything else runs in the browser).
2. **Extract → Redivis**: tidy tables shaped like the existing API's return
   values; keep source data verbatim (no NA flattening); validate derived
   numbers against the old app's own cached outputs before uploading.
   Set `upload_merge_strategy="replace"` via `tb$update()` EVERY upload or
   re-releases double the data.
3. **Site skeleton**: one working lazy-loading viz end-to-end first, then
   the rest. Tier every table (embed / lazy JSON slice / Redivis-only).
   Prefer plain JSON slices; DuckDB-WASM only for big raw tables needing
   arbitrary SQL splits.
4. **Static pages + styling**: replicate the original look (fetch its real
   rendered CSS/markup — declared stylesheets can be dead code; measure the
   live site). Plot/Table tabsets on every viz.
5. **CI**: fetch small tables from Redivis, rebuild slices from committed
   compact artifacts, publish to gh-pages (branch must pre-exist; publish
   step needs the same secrets; LIBARROW_BINARY=true).
6. **R package** (if any): characterization tests from the old package
   first — captured against the LIVE legacy backend before touching infra —
   then the lean Redivis backend against them.
7. **Team QA before cutover**: preview repo with Pages for the site
   (never in-place on a Jekyll-from-master prod), installable branch for
   the package, and a written issue-triage list. External users on binary
   installs hit bugs dev machines can't (see the binary-bake trap).
8. **If the data are also being updated**: stage all legacy versions to
   Redivis first (oldest→newest, one Redivis version each) so infra work
   never blocks on the new import; the re-import lands as just another
   version. Two-phase every release: pre-stage the draft, release by flag
   after sign-off.

Keep the playbook updated at the end of a migration session: append any
new traps encountered to ~/Projects/shiny-to-observable-advice.md.
