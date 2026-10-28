---
name: babyview-dataset
description: >
  Orientation to the BabyView dataset — what it is, its processing pipeline and file
  structure, the release versions, and where every piece lives across GCP (canonical raw
  video), the ccn2 cluster (working copy of frames/audio/transcripts/detections/pose/
  embeddings), Oak (archival), and Airtable (the subject/video registry). Use whenever a
  task involves locating or understanding BabyView data — frames, transcripts, pose,
  embeddings, the video↔subject crosswalk, or which release/volume holds what. Pairs with
  the train-on-ccn2 skill (cluster mechanics) and notes/STORAGE.md in vlm-headcam.
---

# BabyView dataset

## What it is + why it exists

BabyView is a large **longitudinal head-mounted-camera corpus of young children's egocentric
experience** (Long et al. 2025) — infants/toddlers wear a lightweight chest/head GoPro-style rig
at home, capturing the visual + auditory stream a child actually receives. It is built to study
early visual and **language** development from *naturalistic* input, and as training data for
developmentally-plausible vision and vision-language models. It is the successor/scale-up to SAYCam.

**Human-subjects data — the hard rule.** Frames show children's faces and home interiors. Raw
frames/video/audio are **restricted** and must **never** reach a public URL, repo, or external
service. Only aggregate/derived numbers and face-free plots leave the cluster. (In vlm-headcam this
is enforced by `book/publish-public.sh`, which strips frames and aborts on a safety gate.)

### The vlm-headcam sub-project (the work this skill grew out of)
Learn **word↔object mappings** from BabyView video + ASR transcripts with a frozen-encoder
two-tower, and diagnose *why* naive CLIP-style training on this stream fails. Headline: the
bottleneck is the **referential learning signal** (which moments have a word aligned to the visible
object), not the vision encoder or the architecture. See the book
(`mcfrank.quarto.pub/learning-words-from-a-childs-eye-view`) and the `project_vlm_headcam` memory.

## Processing pipeline (raw video → derived artifacts)

Each **recording** (a session, keyed by a `unique_video_id` in the registry) flows through:

```
raw video (GCS) ──▶ compressed video
     │
     ├─▶ extracted_frames_1fps/   1 frame / second (the workhorse for vision models)
     ├─▶ mp3/                     extracted audio
     │      └─▶ ASR transcripts (parsed, time-aligned) ── outputs/
     ├─▶ object detections (CDI-noun YOLOE)             ── outputs/
     ├─▶ pose  (2025.2: YOLO12x + mmpose rtmw-x;         ── outputs/pose_1fps*
     │          COCO-WholeBody 133-kpt, per-person bbox+limbs)
     └─▶ image embeddings (precomputed, several encoders) ── image_embeddings/
```

Alignment layers computed on top of the frame×transcript pairs:
- **CLIP** filter — `full_clip_results.csv` (per-pair CLIP similarity; the original weak filter).
- **Gemini-2.5-Flash** (Vertex AI) — per-pair alignment 0–100 + referent word over the full
  ~1.14M-pair corpus → `scored/gemini_full.parquet`. The current gold alignment signal.

## Releases

| Release | Status | Notes |
|---|---|---|
| **2025.1** | superseded | earlier packaging; the loose **"868-hour chunk"** is the pre-reorg 2025.1 |
| **2025.2** | **current working release** | everything below defaults to this |
| **2026.1** | incoming | ≥2× larger; Alvin (`tanawm`) cycling GCP → ccn → Oak. `/ccn2a/dataset/babyview/2026.1/` is partial while this is in progress — list the directory before assuming frames exist (as of 2026-08-21 it had transcripts and one child's `mp3/`, no frames). Same `video_id` naming as 2025.2. |

## Where things live (four tiers)

### GCP — canonical raw + compressed video
Google Cloud Storage is the **source of truth** for raw and compressed video (confirm the exact
bucket before scripting against it). The Airtable registry stores the per-recording GCS paths
(`superseded_gcp_name_feb25`). A *separate* GCP project holds the Vertex service account used for
Gemini annotation. Project names and the creds location are in `private.md` in this skill's
folder (local only; if it's missing, ask Mike).

### ccn2 cluster — working copy of the processed release
Shared NFS; this is where models actually read frames/embeddings. (Cluster mechanics — SSH
flakiness, GPUs, envs — are in the **train-on-ccn2** skill.) Two roots split raw-ish vs derived:

- **`/ccn2a/dataset/babyview/2025.2/`** — the release:
  - `extracted_frames_1fps/` — 1 fps frames
  - `mp3/` — audio
  - `outputs/` — transcripts, detections, `pose_1fps*`, and **`scored/`** (our Gemini parquet)
  - `image_embeddings/babyview_877k/` — Khai's precomputed embeddings for the **877,802** topline
    frames, in 5 encoders (DINOv2, DINOv3-B/L, V-JEPA2, ZWM) + 4×4 grid variants
  - `eval_konkle_mcfrank/`, `eval_levante_mcfrank/` — eval-set embeddings
- **`/ccn2/dataset/babyview/`** — derived features + older artifacts:
  - `2025.2/outputs/pose_1fps_bbox_limbs.csv` (6.5M rows, per-person body-part bboxes)
  - `full_clip_results.csv` (CLIP alignment)
  - `outputs_20250312/pose/4M_frames_old/` — the older feb-2025 133-kpt pose pull (pickles + csv)

`/ccn2` and `/ccn2a` are shared NFS, usually ~98% full — check `df` before
big writes. `/data`, `/data2` are node-local NVMe (our scratch/caches live in `/data2/mcfrank/`).

### Oak — archival / cold storage (SRCC)
Group space `/oak/stanford/groups/mcfrank/`. A **10 TB Oak allocation** was purchased in 2026 (~$500/yr)
to hold 2026.1 videos **and** intermediates; Alvin is migrating. Steady-state plan: small derived
caches stay on ccn2, the bulk (videos + regenerable intermediates) lives on Oak. Transfer via the DTN
`dtn.oak.stanford.edu` (`oak-dtn` host alias on ccn2; key-based, **Duo-free**; transfer-only, no
shell). Archive-then-verify-then-free — never blind-delete. Full Oak playbook is in **train-on-ccn2**.
The vlm-headcam archive lives at `…/mcfrank/vlm-headcam-archive/` (see `notes/STORAGE.md` in the repo).

### Airtable — the subject / video registry
The authoritative record of recordings and subjects: per-recording IDs, subject **age** and
**language**, capture metadata, and GCS paths. Exported to CSV for offline joins:
`metadata/videos.csv` (**gitignored — human-subjects; local only**). Crosswalk used to join
processing outputs back to a child/session:

```
unique_video_id (Airtable rec-id)  ↔  superseded_gcp_name_feb25 (GCS path)  ↔  pose dir {gcp}_processed
```

(Exact Airtable base/table names not recorded here — confirm with the lab if you need live API access
rather than the exported CSV.)

## Quick "where is X?" table

| Want | Location |
|---|---|
| 1 fps frames | `ccn2:/ccn2a/dataset/babyview/2025.2/extracted_frames_1fps/` |
| audio | `…/2025.2/mp3/` |
| transcripts | `…/2025.2/outputs/` (parsed ASR) |
| object detections | `…/2025.2/outputs/` (CDI YOLOE) |
| pose (current) | `ccn2:/ccn2/dataset/babyview/2025.2/outputs/pose_1fps_bbox_limbs.csv` (+ per-frame pkls in `…/pose_1fps/`) |
| precomputed embeddings | `…/2025.2/image_embeddings/babyview_877k/` |
| CLIP alignment | `ccn2:/ccn2/dataset/babyview/full_clip_results.csv` |
| Gemini alignment | `…/2025.2/outputs/scored/gemini_full.parquet` |
| raw/compressed video | GCS (project in `private.md`), paths in the Airtable registry |
| subject/video metadata | Airtable → `metadata/videos.csv` (gitignored) |
| cold archive / 2026.1 | Oak `/oak/stanford/groups/mcfrank/` |

## Caveats
- ccn2 paths live-verified 2026-08-21 (frames/pose/2026.1 dirs); expect 2026.1 to reshuffle as the
  Oak migration lands. Pose pipeline + runbooks: `~/Projects/bv-annotations` (see
  `docs/handoff-pose-2026.1.md`).
- GCS bucket name and Airtable base id are **not** firmly recorded here — verify before automating.
