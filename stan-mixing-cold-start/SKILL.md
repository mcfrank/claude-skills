---
name: stan-mixing-cold-start
description: Diagnose and fix Stan fits that completed sampling but didn't converge (high Rhat, low n_eff, treedepth saturation). The dominant cause in hierarchical model "family build" workflows is warm-starting a variant from a source posterior that pinned a parameter the target variant frees. Use whenever a fit comes back with Rhat > 1.1, n_eff < 100, large between-chain time spread, or "100% of transitions hit max treedepth" warnings.
---

# Stan mixing failures: the cold-start / boundary-parking pattern

## Symptoms

A Stan fit that sampled all the way through but came back with:

- **Rhat 2.5–3.0** on key parameters (target: <1.05)
- **n_eff ≈ 5** (target: >>100)
- **100% of transitions hit max treedepth=10**
- **All chains had E-BFMI < 0.3** (energy diagnostic failure)
- **Large between-chain wall-clock spread** during sampling — fastest chain finishes 25–50% earlier than slowest (e.g., 10 h vs 14 h)

These usually co-occur. If you see one, the others are likely true.

## Most common cause: warm-start from a degenerate prior fit

Hierarchical "family build" workflows fit a sequence of nested variants — each one adds a parameter (a variance component, a slope, etc.) the previous didn't have. Warm-starting variant N+1 from variant N's posterior means initializing the newly-freed parameter at the value variant N pinned it to.

**Concrete pattern** (encountered in `standard_model_2`, May 2026):

`demo_pure` pins σ_α = 0. `demo_alpha` frees σ_α but is initialized from `demo_pure`'s posterior, where σ_α ≈ 0. All chains then start at the σ_α = 0 boundary. The geometry there is highly correlated (σ_α and σ_r trade off ambiguously near zero), and the sampler can't escape — every transition saturates max_treedepth trying. Result: Rhat ~2.7, n_eff ~5, full treedepth saturation.

## The fix is the init, not the sampler config

This is **not** fixed by:

- **Bumping `max_treedepth`** (chains still can't move; they spend more time per trajectory — wall-clock goes 6–8× slower)
- **Bumping `adapt_delta`** (smaller step, more leapfrog steps, same stuck region)
- **More warmup iters** (warmup is doing the right thing; the init is wrong)

The fix is to **cold-start** the variant (drop `init = ...`, let cmdstanr draw initial values from the prior).

## Policy for nested family builds

For each variant N+1, ask: does the source variant N pin (force to a specific value) any parameter that N+1 frees?

- If **yes**: cold-start N+1. The source posterior is degenerate on those parameters; warm-init parks chains in a corner.
- If **no**: warm-start is fine. Source posterior is in the typical set, init savings are real (~10–15% warmup reduction on big fits).

Concretely in a typical incrementally-built family:

| Source | Target | Pin in source? | Init from source? |
|---|---|---|---|
| `pure` (pins σ_α=0) | `+α` (frees σ_α) | yes | **cold-start** |
| `+α` (pins κ=1) | `+κ` (frees κ) | yes | **cold-start** |
| `+κ` (pins σ_ζ=0) | `+σ_ζ` (frees σ_ζ) | yes | **cold-start** |
| `+σ_ζ` (well-mixed, all params free) | `+σ_s` (adds onset) | no | warm-start OK |

## Don't reach for adapt_delta / max_treedepth defensively

When recovering from a mixing failure, **don't bump `adapt_delta` to 0.97 or `max_treedepth` to 12 just in case**. They:

- Cost ~6–8× per-iteration wall-clock on healthy chains
- Don't fix the underlying boundary problem (which comes from init, not from sampler config)
- Make it harder to tell whether the fix actually worked, because everything is slow

Default knobs (`adapt_delta=0.95`, `max_treedepth=10`) are fine. Only bump after a cold-start fit ALSO fails to converge, and you've ruled out other causes.

## Spread between chain finish times is the early warning

If chain 1 finishes sampling at 10 h and chain 3 is still at 70% sampling 13 h in, you're likely already in the boundary-parking regime. Don't wait for the final summary — check Rhat from a partial chain readback, and if it's bad, abort and switch to cold-start.

```
ps -eo etime,cmd | grep -i stan      # remaining chain elapsed times
grep "Iteration" <log> | tail -10    # iter progression per chain
```

## Recovering a stalled family

If the family driver crashed (e.g., bash `set -e` killed it after a bad fit), the standard recovery is:

1. Delete the bad fit's `.rds` (so the driver doesn't think it's done): `rm fits/<bad-tag>.rds`
2. Re-run the family driver. A well-written family driver should have a `skip-if-exists` check on `fits/<tag>.rds` so it picks up from the failed variant.

If the family driver doesn't have skip-if-exists, add one — it makes recovery from any kind of mid-family failure trivial. Reference pattern in `standard_model_2/gcp/run_family.sh`.

## A note on Rhat and admin density

With sparse longitudinal data (e.g., median 3 admins/child), per-child slope parameters become weakly identified. The population-level "intercept-and-slope" decomposition shifts across data densities even when the aggregate (e.g., `κ_i`) distribution is the same. Don't read too much into a population-mean term shift between datasets with very different per-unit measurement density — read the aggregate.
