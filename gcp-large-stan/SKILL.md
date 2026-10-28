---
name: gcp-large-stan
description: Fit large Stan/cmdstanr models on GCP n2d-standard-* VMs. Use when launching, monitoring, or recovering Stan fits on a GCP VM — covers VM sizing, SMT-aware threading, the AMD EPYC compile-flag segfault trap, disk requirements for log_lik storage, and the launch / detach / monitor pattern.
---

# Stan model fitting on GCP n2d VMs

Concrete patterns for fitting big Stan models (N ≥ 1M observations) on
GCP `n2d-standard-*` virtual machines. Battle-tested on the
`standard_model_2` project (Mike Frank, May 2026) and the lessons here
mostly generalize to any Stan-on-GCP work.

## VM sizing

For N ≥ 1M observations and a moderately complex hierarchical model:

- **`n2d-standard-128`**: 64 physical AMD EPYC cores (128 vCPUs with SMT), 512 GB RAM
  - ~$5.40/hr on-demand (us-central1, Oct 2026)
  - Sweet spot for 4 chains × 16 threads/chain saturating the physical cores
  - Fits N ≈ 2M without memory pressure

- **`n2d-standard-32`**: 16 physical cores, 32 vCPUs, 128 GB RAM
  - Cheaper (~$1.35/hr); check peak memory for N > 1M (log_lik alone is ~35 GB at N ≈ 2M)
  - Use 4 chains × 4 threads/chain (don't oversubscribe SMT)

Create with a boot disk sized from the Disk table below (this example is for N ≤ 1.2M; use 400GB for 1.5M–2.5M):

```
gcloud compute instances create sm-fit-01 \
  --machine-type=n2d-standard-128 \
  --boot-disk-size=250GB \
  --boot-disk-type=pd-balanced \
  --image-family=ubuntu-2404-lts \
  --image-project=ubuntu-os-cloud \
  --zone=us-central1-a \
  --metadata-from-file=startup-script=setup_vm.sh
```

## Threading rule (the single most important config)

**`(chains × threads/chain) = physical cores`**. Not vCPUs.

`n2d-standard-128` has 128 vCPUs but only 64 physical cores (SMT 2×). Stan's HMC is FPU-bound, and SMT siblings share an FPU. If you set `chains × threads = 128`, you'll oversubscribe FPUs and run ~3× slower than the right config.

Configs that work:

| Machine | Physical cores | Config |
|---|---|---|
| n2d-standard-128 | 64 | 4 chains × 16 threads |
| n2d-standard-32 | 16 | 4 chains × 4 threads |
| n2d-standard-16 | 8 | 4 chains × 2 threads |

Pass via `STAN_THREADS_PER_CHAIN=N` env var, with cmdstanr `threads_per_chain = N`.

Verify by `ps -eo pcpu | head` while sampling — each chain should be at ~`N × 100%`. If you see 4 chains at ~3200% each on `n2d-standard-128` (32 threads/chain, filling every SMT sibling), you've oversubscribed.

## Compile-flag trap on AMD EPYC

Stan models on AMD EPYC at default cmdstan `-O3` will **segfault with "double free or corruption (out)"** if compiled with `-march=native -DNDEBUG`. This bit us hard on Stan 2.38 + AMD EPYC 7B13.

**`cmdstan/make/local` should contain ONLY**:
```
STAN_THREADS=true
```

No `-march=native`. No `-DNDEBUG`. No `CXXFLAGS+=-O3` (already on). Stan's default flags are fine.

If a previously-working fit suddenly starts segfaulting at startup or mid-sample, suspect contamination from a bad-flags compile. Recover by:
```
cd $CMDSTAN_PATH && make clean-all && make build
```
Then recompile each model.

## Disk requirement

Stan writes `log_lik` for every `(chain, iter, obs)` tuple to /tmp CSV during sampling. At N=2.18M obs, this is **~30 MB per iteration per chain** (measured, May 2026). A 4-chain × 1000-sampling-iter fit then writes:

- ~30 GB per chain to /tmp during sampling
- ~120 GB across all 4 chains
- Plus the .rds being assembled by `read_cmdstan_csv` (which uses /tmp too)
- Plus general OS / R working space

**Boot disk sizing rule:**

| N (observations) | Boot disk size | Notes |
|---|---|---|
| < 500K | 100 GB | default is fine |
| 500K – 1.2M | **250 GB** | tested with English Wordbank (N=1.1M); ~80 GB peak /tmp usage |
| 1.5M – 2.5M | **400 GB** | required for Norwegian (N=2.18M); 250 GB hit ceiling at ~iter 1800/2000 |
| > 2.5M | 500 GB | extrapolated; verify empirically |

Size the disk for the largest dataset you'll fit on this VM: if /tmp fills mid-sampling the fit dies and all sampling time is lost (15 h in one case).

Online-resize if a running fit is filling disk:
```
gcloud compute disks resize <disk-name> --size=<new-size>GB --zone=<zone>
# Then on the VM:
sudo growpart /dev/sda 1 && sudo resize2fs /dev/sda1
```

## Setup script pattern

A `setup_vm.sh` passed as `--metadata-from-file=startup-script=setup_vm.sh` should install in ~30 minutes:

1. Build essentials, R, libcurl/libssl headers
2. Posit Package Manager as `repos` in `/etc/R/Rprofile.site` (binary CRAN packages, fast install)
3. R packages: cmdstanr (from stan-dev R-universe), posterior, loo, dplyr, etc.
4. cmdstan to `/opt/cmdstan` via `cmdstanr::install_cmdstan()`
5. `make/local` with `STAN_THREADS=true` only
6. Chown `/opt/cmdstan` to default user so non-root compiles work
7. `/etc/profile.d/cmdstan.sh` exporting `CMDSTAN=/opt/cmdstan/cmdstan-<ver>`

Standard pattern in `standard_model_2/gcp/setup_vm.sh` for reference.

## Detached launch pattern

The four-piece incantation for fire-and-forget on a remote VM:

```
gcloud compute ssh <vm> --command="cd ~/path && \
  ENV_VAR=val nohup bash run_thing.sh > thing.log 2>&1 < /dev/null & disown"
```

All four pieces matter:

- `nohup` — survives SIGHUP when ssh disconnects
- `> log 2>&1` — captures all output (else SSH waits on it)
- `< /dev/null` — closes stdin (else SSH waits on input)
- `& disown` — backgrounds AND removes from bash's job table

Without `< /dev/null` and `& disown`, SSH can still hang or the process can die when ssh closes.

## Monitoring

```
# Are chains alive and saturating CPU?
ps -eo etime,pcpu,cmd --sort=-pcpu | grep log_irt | head

# Iter progression (Stan logs every refresh=200)
tail -20 thing.log | grep "Iteration"

# Disk pressure
df -h /tmp /
```

Healthy chain rate on `n2d-standard-128`: ~5–6 iter/min for typical models. <1 iter/min suggests treedepth saturation — see `stan-mixing-cold-start` skill.

## File handling between Mac and Linux VM

**Never rsync a compiled Stan model binary from Mac → Linux VM.** Mac arm64 binaries segfault on x86_64 Linux. Use selective `gcloud compute scp` of just the `.stan` source, then recompile remotely.

Always `rm` the binary + `.rds` cache after `scp`'ing a modified `.stan`:
```
rm -f model/stan/<name>           # the compiled binary
rm -f model/stan/<name>.rds       # cmdstanr's cache
```

## Cost notes

- `n2d-standard-128` ~$5.40/hr × 11 h/fit × 5 fits = ~$300 per language family run
- `n2d-standard-32` ~$1.35/hr × 14 h/fit × 5 fits = ~$95 per language family run
- Boot disk: ~$0.10/GB-month for pd-balanced (us-central1, Oct 2026)

Stop VMs (don't delete) between campaigns:
```
gcloud compute instances stop <vm> --zone=<zone>
```
Stopped VMs charge ~$0/hr for compute but ~$0.10/GB-month for the pd-balanced disk.
