---
name: psis-loo-chunked
description: Compute PSIS-LOO on Stan fits with very large log_lik matrices (N >~ 2M observations). Use whenever the vanilla loo() call segfaults inside psis_apply on a big posterior, or proactively when about to run LOO on N > 1.5M obs. PSIS-LOO is per-observation independent, so chunking is exact (not an approximation).
---

# Chunked PSIS-LOO for very large datasets

## Why vanilla `loo()` segfaults at scale

The `loo` R package's `psis_apply` function builds intermediate `lw_list` copies of the log-weights matrix. At N ≈ 2M obs × 2000 draws (thinned), the log_lik matrix is ~35 GB; `psis_apply` peak memory pushes past ~70 GB. On the 128-vCPU GCP VM used for these fits it segfaulted with "memory not mapped" addresses, so treat this as a scale limit of `psis_apply`, not only a RAM shortfall.

(Observed on `n2d-standard-128`, May 2026, Norwegian Wordbank fit at N=2.18M.)

## Chunking is exact, not approximate

PSIS-LOO computes, **per observation `i` independently**:

- `log_weights_i = -log_lik_i`
- Fits a generalized Pareto to the top tail (PSIS smoothing)
- `elpd_loo_i = log(mean(exp(log_lik_i + smoothed_log_weights_i)))`
- `pareto_k_i = GPD shape param`

Nothing crosses observation boundaries. Therefore:

- `total elpd_loo = sum(elpd_loo_i)` (exact)
- `SE(elpd_loo) = sqrt(N · var(elpd_loo_i))` (exact)
- `pareto_k_i` is per-obs and stable under chunking

Validation: on synthetic data, chunked output matches vanilla `loo()` to machine precision (max abs diff `0.000e+00` on elpd, pareto_k, mcse).

## Implementation recipe

```r
suppressPackageStartupMessages({
  library(posterior)
  library(loo)
})

chunked_loo <- function(ll, chunk_obs = 250000L) {
  N <- ncol(ll)
  S <- nrow(ll)

  elpd_pw  <- numeric(N)
  p_pw     <- numeric(N)
  mcse_pw  <- numeric(N)
  k_pw     <- numeric(N)
  n_eff_pw <- numeric(N)

  n_chunks <- ceiling(N / chunk_obs)
  for (ci in seq_len(n_chunks)) {
    a <- (ci - 1L) * chunk_obs + 1L
    b <- min(ci * chunk_obs, N)

    ll_chunk <- ll[, a:b, drop = FALSE]
    r_eff_chunk <- rep(1.0, ncol(ll_chunk))

    loo_chunk <- loo(ll_chunk, r_eff = r_eff_chunk, cores = 1L,
                     save_psis = FALSE)   # save_psis is the big memory hog

    pw <- loo_chunk$pointwise
    elpd_pw[a:b]  <- pw[, "elpd_loo"]
    p_pw[a:b]     <- pw[, "p_loo"]
    mcse_pw[a:b]  <- pw[, "mcse_elpd_loo"]
    k_pw[a:b]     <- loo_chunk$diagnostics$pareto_k
    n_eff_pw[a:b] <- loo_chunk$diagnostics$n_eff

    rm(ll_chunk, loo_chunk, r_eff_chunk)
    gc(verbose = FALSE)
  }

  # Aggregate point estimates and SEs across observations
  elpd_loo <- sum(elpd_pw)
  elpd_se  <- sqrt(N * stats::var(elpd_pw))
  p_loo    <- sum(p_pw)
  p_se     <- sqrt(N * stats::var(p_pw))
  looic    <- -2 * elpd_loo
  looic_se <- 2 * elpd_se

  estimates <- matrix(
    c(elpd_loo, elpd_se, p_loo, p_se, looic, looic_se),
    nrow = 3, byrow = TRUE,
    dimnames = list(c("elpd_loo", "p_loo", "looic"),
                    c("Estimate", "SE"))
  )

  pointwise <- cbind(
    elpd_loo      = elpd_pw,
    mcse_elpd_loo = mcse_pw,
    p_loo         = p_pw,
    looic         = -2 * elpd_pw
  )

  out <- list(
    estimates   = estimates,
    pointwise   = pointwise,
    diagnostics = list(pareto_k = k_pw, n_eff = n_eff_pw),
    elpd_loo    = elpd_loo,
    se_elpd_loo = elpd_se,
    p_loo       = p_loo,
    looic       = looic,
    N = N, S = S, method = "psis_chunked"
  )
  class(out) <- c("psis_loo", "loo")
  out
}
```

Full implementation including the per-tag driver:
`standard_model_2/sherlock/extract_loo_thinned.R`

## Chunk size

`chunk_obs = 250000` works well at 2000 draws (≈4 GB per chunk). Tune:

- Reduce if you have <16 GB working memory free (try 100K)
- Increase if you have lots of RAM and want fewer chunks (try 500K)

Each chunk is independent, so chunk size doesn't affect the result, just speed × memory tradeoff.

## Pre-thin draws first

Before chunking, thin the draws if you have more than ~2000 of them — PSIS-LOO is robust to thinning down to ~1000 well-mixed draws. Halves memory:

```r
N_THIN <- 2000L
ll_full <- fit$draws("log_lik", format = "draws_matrix")
D <- nrow(ll_full)
if (D > N_THIN) {
  keep <- as.integer(seq(1, D, length.out = N_THIN))
  ll <- ll_full[keep, , drop = FALSE]
  rm(ll_full); gc()
} else {
  ll <- ll_full; rm(ll_full); gc()
}
```

## Always pass `save_psis = FALSE`

Each `loo()` call defaults to saving the full PSIS object. At N=2M that's ~100 GB. Pass `save_psis = FALSE` to skip it. Downstream consumers (e.g., `loo_compare`) can recompute if needed.

## What success looks like

```
log_lik shape: 4000 draws x 2180667 obs (69.78 GB)
Thinned to 2000 draws (34.89 GB)
Computing PSIS-LOO in chunks of 250000 obs ...
  chunk 1/9: obs 1..250000 (250000 obs)
  ...
  chunk 9/9: obs 2000001..2180667 (180667 obs)
Wrote LOO (elpd = -654836.6 +- 770.1, pareto_k > 0.7: 97168 / 2180667 obs).
```

High fraction of pareto_k > 0.7 (e.g., 4.5%) is a **model-misspecification signal**, not a numerical problem with chunking. The fit isn't great; LOO is correctly telling you.
