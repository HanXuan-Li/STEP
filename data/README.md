# STEP · Data Samples

This folder ships **three concrete training samples** produced by the STEP-CoT data engine.
They are meant as a reference for the exact JSON schema, prompt style, and image layout that
STEP expects — *not* as a substitute for the full 400k-sample training corpus, which will be
released separately.

```
data/
├── README.md
├── DATA_FORMAT.md                   # full spec of the STEP-CoT sharegpt schema
├── dataset_info.json                # portable LLaMA-Factory dataset registry
├── samples.json                     # 3 records in LLaMA-Factory sharegpt format
└── images/                          # RGB views + BEV maps referenced by samples.json
    ├── 01_topological_map.png
    ├── 02_pose_rgb.png
    ├── 02_pose_map.png
    ├── 03_navigation_rgb.png
    └── 03_navigation_map.png
```

## Schema

Each entry follows LLaMA-Factory's `sharegpt` format (see
[`DATA_FORMAT.md`](DATA_FORMAT.md) for the full spec):

```json
{
  "messages": [
    {"role": "user",      "content": "... <image> ... <image> ..."},
    {"role": "assistant", "content": "..."}
  ],
  "images": ["images/<file_1>.png", "images/<file_2>.png"]
}
```

The number of `<image>` placeholders in the user turn matches `len(images)`, and images appear
in the same order in which they are referenced. Assistant answers are wrapped in
`<thinking>...</thinking>` and `<answer>...</answer>` tags when Chain-of-Thought supervision is
enabled.

## The three samples

| # | Task family | Perception input | Answer style |
|---|-------------|------------------|--------------|
| 01 | **Topological Reasoning** (primitive skill) | BEV map with agent arrow | Objects at *right-back* of the agent, listed inside `<answer>` |
| 02 | **Pose Estimation** (primitive skill)      | Egocentric RGB + BEV map | Orientation angle in `[0, 360°]` inside `<answer>` |
| 03 | **Point-Level Navigation** (main task)     | Egocentric RGB with Set-of-Mark labels + BEV map | `<thinking>` chain followed by a marker ID such as `<answer>20</answer>` or an atomic action (`MoveAhead`, `RotateLeft`, `RotateRight`) |

Samples 01 and 02 correspond to two of the *primitive-skill* subsets produced by STEP-CoT to
boost spatial awareness (Topological Reasoning, Pose Estimation, Relative Orientation, Semantic
Inquiry, …). Sample 03 illustrates the *main* Point-Level Planning task where the agent must
choose the next waypoint given an egocentric view marked with reachable candidates.

## Using the samples

The checked-in [`dataset_info.json`](dataset_info.json) already registers these records
under `step_samples`. For a quick data-loading smoke test, copy the training YAML and set
`dataset: step_samples`; keep `dataset_dir: ../data` and `media_dir: ../data` unchanged.

For full training, place the released corpus at `data/step_cot_train.json` (ignored by
Git by default) or update the `step_cot_train` entry to the filename you use.

## The full STEP-CoT dataset

The three samples here are a small window into the STEP-CoT corpus. The full dataset
(≈400k multimodal Chain-of-Thought samples covering the ALFRED / AI2-THOR domain) is being
prepared for public release. Please watch this repository — release links and download
instructions will be added to the top-level [`README`](../README.md) as soon as they are
available.
