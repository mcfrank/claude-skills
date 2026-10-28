---
name: redivis
description: >
  Practical playbook for any work with Redivis — creating/versioning datasets, tabular
  and raw-file (index table) uploads, org tokens and scopes, the python/R clients and
  REST quirks, releases, and linking/referencing datasets correctly. Use whenever a
  session touches Redivis (uploading data, building or extending a dataset, scripting
  against the API, debugging auth/upload/version errors, writing docs that link to
  datasets) and the task is NOT the shiny→datapage site migration — that pattern has its
  own skill (shiny-to-datapage), which pairs with this one.
---

# Redivis playbook

Battle-tested on: wordbank/childes-db/peekbank staging (2026) and the BabyView
IMU archive (Aug 2026; reference implementation with all patterns working:
`~/Projects/babyview-redivis` — manifest → download+scan → stage/verify/release).

## Mental model

Dataset → **versions** (immutable released snapshots + at most one `next`
draft) → tables (tabular, or **file index** for raw files) → uploads/files.
Any data change requires a new version; metadata (description, labels) can be
edited on released datasets. Version tags are **auto-assigned** minor bumps
(v1.0, v1.1, …) — you don't choose them, so keep your own source-release↔tag
map (JSON in the repo + restate in each version's release notes).

## Referencing & links (recurring gotcha)

- Every dataset/table has a short **reference code** (`babyview_imu:e7we`,
  `recordings:vzmc`). Name-only references work but break on rename and make
  the python client print warnings; use the qualified `name:code` form in code.
- **Dataset links in text/docs: name-based URLs to org datasets don't work.**
  Always build permalinks with the reference code, e.g.
  `https://redivis.com/datasets/datapages.babyview_imu:e7we` — not
  `…/datasets/datapages.babyview_imu`. Get the code from the API
  (`qualifiedReference`) or the dataset page URL.
- Version-pinned reference: `owner.name:code:v1_2` (tags print as `v1_4`,
  resolve as `"v1.4"`).

## Auth & scopes

- Token in env `REDIVIS_API_TOKEN` (keep in a gitignored `.secrets`, format
  `REDIVIS_API_TOKEN=…`). The python client silently prefers env over cached
  OAuth.
- Scopes bite in two places: `organization.read` to list/release org
  datasets, **`organization.write` to create datasets in an org** — a token
  that lists org datasets fine can still 403 `insufficient_scope` on create.
  Personal-workspace token UIs may not offer org scopes; mint from the org.
- Symptom pattern: 403 with `insufficient_scope` = wrong token scopes;
  400 `Cannot modify data on a released version` = you targeted a released
  version instead of the draft (see below).

## The draft-targeting rule (python)

Once ANY version is released, the default dataset reference resolves to the
released version and **rejects writes**. All staging operations must target
the draft explicitly:

```python
ds = redivis.organization("org").dataset("name:code")
ds.create_next_version(if_not_exists=True)   # no-ops if draft already open
draft = redivis.organization("org").dataset("name:code", version="next")
tb = draft.table("mytable")                  # write via THIS, not ds.table()
```

This bug is invisible on a brand-new dataset (everything is the draft until
the first release) and appears only when staging version 2.

## Tabular uploads

- `table.create(upload_merge_strategy="replace")` **does not take effect** —
  call `tb.update(upload_merge_strategy="replace")` every run, or a re-release
  silently DOUBLES the data. Uploads within one version COMBINE.
- With replace strategy, a table with NO new upload in a version silently
  keeps the previous version's rows — stage an explicit empty upload if a
  table should be empty.
- `tb.upload(name).create(path, type="delimited", replace_on_conflict=True,
  wait_for_finish=True, raise_on_fail=True)`. Redivis normalizes upload names
  — compare on alphanumerics when checking "already uploaded".
- Type inference from CSV is good: ISO dates → date, `true/false` → boolean
  (write lowercase), empty strings → NULL. Verify `numRows` and
  `list_variables()` after upload.
- Big data: compact to few large files first (thousands of small uploads =
  seconds of API overhead each); parquet needs `min_rows_per_group` set or
  you hit the 32,767-row-groups cap ("ordinal 32768 exceeds i16 max").

## Raw files (file index tables)

- `table.create(is_file_index=True)`, then `table.add_files(files=[{"path":…,
  "name":…, "size":…}, …])` (or `directory=`). Internally parallel (cpu×5)
  and batched; limits 100M files / 5 TB per file.
- Index rows: `file_id`, `file_name`, `size`, `added_at`, `md5_hash` —
  **md5_hash is base64**; convert (`base64.b64decode(v).hex()`) before
  comparing to hex md5s (GCS etc.). This enables true end-to-end checksum
  verification source → local → Redivis.
- Bulk `add_files` can die mid-run with 403s on signed temp-upload URLs
  ("Upload failed after N retries"). Design resumable: list draft index
  rows, skip present names, chunk uploads (~1000/call), rerun on failure.
- New versions carry forward existing files; `add_files` on the draft adds
  only the new ones.

## Reading / REST quirks (python client)

- Rows endpoint: `make_api_request(method="GET", path=f"{table.uri}/rows",
  query={"maxResults": N}, parse_response=False)` returns a
  `requests.Response` (use `.text`), NDJSON lines, typed values.
- **The /rows endpoint 500s on an empty file-index table** — check
  `table.get()` → `numRows` first, and only treat empty as empty when
  numRows==0; never swallow errors into "no files" (causes duplicate
  re-uploads or vacuous verification).
- Releases take minutes to propagate for large tables — a read right after
  `release()` can 404 at the previous version; retry.
- Pin versions (`dataset(name, version="v1.4")`) in anything reproducible.

## Release discipline

- **Two-phase always**: stage draft → verify draft against a manifest (row
  counts; for files: name set + sizes + md5s) → release only on PASS.
  Releases are the near-irreversible step (`unrelease()` exists for the
  latest version, but treat it as an escape hatch).
- `ds.release(release_notes=…)` — put the source-release name and coverage
  caveats in the notes.
- Dataset `description` is capped at **2,000 chars**; longer docs go in the
  UI Documentation section (no API in the python client) or the repo.
- Deletion of a whole dataset is possible while private; think before
  releasing public.

## R client / CI (brief — see shiny-to-datapage for depth)

Org-scoped token needed for API releases; `LIBARROW_BINARY=true` +
`NOT_CRAN=true` on CI or arrow compiles ~2 h; redivis-r isn't on CRAN (
Suggests + r-universe); beware the binary-bake trap (OAuth cache path baked
at build time into binaries). Full detail: `shiny-to-datapage` skill and
`~/Projects/shiny-to-observable-advice.md`.

Where the R client differs from the python one above (verified Aug 2026,
staging the `acceleration` fit archive):

- **Namespace form**: `redivis$organization("org")$dataset("name:code")`.
  The older `redivis::organization(...)` still works but warns.
- **Methods are invisible to `names()`** — that returns fields only, so an
  object looks empty. The classes are **R6** (verified redivis 0.12.12, Sep
  2026; the older `$methods()` recipe errors with "attempt to apply
  non-function"). Enumerate with
  `names(get("Table", envir = asNamespace("redivis"))$public_methods)`, and
  walk parents with `$get_inherit()` — many methods are inherited:
  `download_files()`, `list_files()` and `to_data_frame()` live on the parent
  `TabularReader`, not on `Table`. Signatures via
  `formals(gen$public_methods$download_files)`.
- **Downloading a file-index table:**
  `tb$download_files(path = dir)` writes every file flat into `dir` (names
  only, no subdirectories) — route files afterwards if your code expects a tree.
  Auth: `REDIVIS_API_TOKEN` if set (the client warns this is discouraged in
  interactive sessions), otherwise an OAuth device-login on first use.
- **Draft targeting, R form** of the python rule above:
  ```r
  ds$create_next_version()                       # errors if a draft is open
  draft <- redivis$organization("org")$dataset("name:code", version = "next")
  tb <- draft$table("mytable")                   # write via THIS after v1 exists
  ```
- **`list_files()` returns File refs carrying only `kind`, `id`, `uri` — no
  name**, so it cannot answer "is this file already uploaded". Read the index
  table itself instead: `tb$to_data_frame(max_results = 2e5)` gives
  `file_id, file_name, size, added_at, md5_hash`. That is also what makes a
  resumable uploader possible in R.
- **Verify on md5, not row count.** A matching `numRows` only proves the right
  NUMBER of files arrived. The index md5 is base64; in R,
  `paste(sprintf("%02x", as.integer(openssl::base64_decode(v))), collapse="")`
  converts it for comparison with `tools::md5sum()`.
- **`add_files()` prints curl noise while succeeding.** Broken-pipe and
  "HTTP2 framing layer" errors scroll past on a slow link and the call still
  returns normally with every file uploaded. Never read those as failure, and
  never read their absence as success — verify md5s either way. Chunking
  (~400 files) plus a retry loop plus the skip-what-is-present check above
  makes a multi-hundred-MB upload survive a flaky connection.

## Trap: re-uploading a table into an existing unreleased version appends

`upload_merge_strategy = "replace"` only supersedes the *previous released
version's* rows. Two uploads into the same unreleased ("next") version are
combined, so re-uploading a corrected table doubles it (wordbank NEXT,
2026-09-15: datasets went 180 → 360). To rebuild a table within a version:
`for (u in tb$list_uploads()) u$delete()` first, then upload. Verify with a
`SELECT COUNT(*)` on `` `org.dataset:ref:next.table` `` afterwards — the
table page's row count lags. Query references for an unreleased version use
the literal `next`; released ones use the tag with `_` (`v2_0`).
