# STEP · Training

The STEP model is fine-tuned from **Qwen2.5-VL-7B-Instruct** using
[LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) as the training backbone. We ship a
lightly-pruned, vendored copy under [`LLaMA-Factory/`](LLaMA-Factory/) so that this repository
contains the exact training stack used by this release.

```
STEP/
├── LLaMA-Factory/            # vendored training backbone (upstream + apache-2.0 license)
│   └── README.md             # tells you what was kept / removed
├── data/
│   ├── dataset_info.json     # portable dataset registry
│   └── samples.json          # three multimodal format examples
└── scripts/
    └── qwen2_5vl_full_sft_STEP.yaml    # STEP full-SFT recipe (Qwen2.5-VL-7B)
```

## 1. Setup

```bash
# Create a fresh environment (Python >= 3.10)
conda create -n step python=3.10 -y
conda activate step

# Install LLaMA-Factory in editable mode
cd LLaMA-Factory
pip install -e ".[torch,metrics,deepspeed]"
```

You will additionally need:
- CUDA 11.8+ / PyTorch 2.1+ (bf16-capable GPUs recommended, e.g. 8× A100-80G or H800).
- The base checkpoint [`Qwen/Qwen2.5-VL-7B-Instruct`](https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct)
  (downloaded automatically by 🤗 Transformers on first use, or point
  `model_name_or_path` in the YAML to a local path).

## 2. Data

STEP is trained on the **STEP-CoT** dataset (see the top-level [`README`](README.md) and
[`data/`](data/) for the schema). Each record follows LLaMA-Factory's `sharegpt` format:

```json
{
  "messages": [
    {"role": "user", "content": "... <image> ... <image> ..."},
    {"role": "assistant", "content": "<thinking>...</thinking><answer>...</answer>"}
  ],
  "images": ["path/to/rgb.png", "path/to/bev_map.png"]
}
```

The project-level [`data/dataset_info.json`](data/dataset_info.json) already contains
two entries:

- `step_samples` points to the three checked-in records in `data/samples.json`.
- `step_cot_train` points to `data/step_cot_train.json`, the expected location of the
  full corpus. This large local file is ignored by Git.

If you use a different filename, update the portable registry rather than adding an
absolute path:

```jsonc
{
  "step_cot_train": {
    "file_name": "step_cot_train.json",
    "formatting": "sharegpt",
    "columns": {"messages": "messages", "images": "images"},
    "tags": {
      "role_tag": "role", "content_tag": "content",
      "user_tag": "user", "assistant_tag": "assistant"
    }
  }
}
```

## 3. Launch training

The recipe [`scripts/qwen2_5vl_full_sft_STEP.yaml`](scripts/qwen2_5vl_full_sft_STEP.yaml)
captures the full-parameter SFT setup reported in the paper (Qwen2.5-VL-7B, global
batch size 128 with 8 processes, learning rate 1e-5, cosine schedule, 2 epochs,
ZeRO-3, an unfrozen vision tower, and a trainable projector).

From `LLaMA-Factory/`:

```bash
# Ensure ../data/step_cot_train.json exists (or update the registry), then:
llamafactory-cli train ../scripts/qwen2_5vl_full_sft_STEP.yaml
```

Or with `torchrun` for multi-node runs — see LLaMA-Factory's
[`README.upstream.md`](LLaMA-Factory/README.upstream.md) for distributed launch patterns.

## 4. Notes on the recipe

- `freeze_vision_tower: false`, `freeze_multi_modal_projector: false` — both the vision tower
  and multimodal projector remain trainable so visual and BEV-map features can be jointly
  aligned with the language model.
- `image_max_pixels: 313600` — matches the 560×560 map crop used by STEP-CoT.
- `template: qwen2_vl` — the Qwen2.5-VL chat template shipped with LLaMA-Factory.
- `preprocessing_num_workers: 128` matches the reported run; reduce it on machines with
  fewer CPU cores.
- After training, use `llamafactory-cli export` (or the `examples/merge_lora/` recipe) to obtain
  a merged, deployable checkpoint.
