# LLaMA-Factory (vendored for STEP)

This directory contains a **vendored copy of [LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory)**,
which is the training backbone used by **STEP: Spatial Thinking and Egocentric Pointing for
Embodied Instruction Following**.

We use LLaMA-Factory to fine-tune Qwen2.5-VL on the STEP-CoT dataset. Its training algorithms
and architecture are unchanged. This vendored snapshot contains project-specific adjustments
to examples, dataset registration, documentation, and runtime defaults. All STEP-specific
artifacts (training YAMLs, data-registration entries, and sample data) live under
[`../scripts/`](../scripts/) and [`../data/`](../data/), so that this folder stays easy to
compare with upstream.

The vendored package reports version `0.9.4.dev0`. When refreshing this directory, record
the exact upstream commit so future releases can be compared and upgraded deterministically.

> The upstream README is retained as [`README.upstream.md`](README.upstream.md)
> (and [`README_zh.upstream.md`](README_zh.upstream.md) for Chinese), with removed local
> demo links redirected to the upstream dataset. Please refer to it for the full list of
> supported models, PEFT methods, and installation options.

## What is (and isn't) in this copy

Kept from upstream:
- `src/` — LLaMA-Factory Python package (`llamafactory`), CLI entry-points, `train.py`, `api.py`, `webui.py`.
- `examples/` — upstream training / inference / merge / deepspeed / megatron examples.
- `data/dataset_info.json` — upstream dataset registry. Supported local demo entries now
  resolve to the public `llamafactory/demo_data` dataset on demand; unsupported local-only
  multimedia demo entries are omitted.
- `docker/`, `scripts/`, `tests/`, `requirements.txt`, `setup.py`, `pyproject.toml`, etc.

Removed / not vendored:
- STEP-specific launch shell scripts, checkpoint dumps (`saves/`, `export/`), experiment logs,
  and previous STEP dataset registrations in `data/dataset_info.json`.
- Upstream local demo corpora and demo multimedia, which are not required for STEP training.

## Installation

```bash
cd LLaMA-Factory
pip install -e ".[torch,metrics]"
```

See [`README.upstream.md`](README.upstream.md) for hardware requirements and alternative
install profiles (`deepspeed`, `bitsandbytes`, `vllm`, `awq`, etc.).

## Training STEP with this copy

1. Prepare your STEP-CoT training JSON in ShareGPT format (see
   [`../data/`](../data/) for three concrete examples).
2. Use the checked-in project-level [`../data/dataset_info.json`](../data/dataset_info.json).
   If your full corpus uses a different filename, update its `step_cot_train` entry:
   ```json
   "step_cot_train": {
     "file_name": "step_cot_train.json",
     "formatting": "sharegpt",
     "columns": {"messages": "messages", "images": "images"},
     "tags": {
       "role_tag": "role", "content_tag": "content",
       "user_tag": "user", "assistant_tag": "assistant"
     }
   }
   ```
   and set `dataset: step_cot_train` in the training YAML.
3. Launch training with the STEP recipe:
   ```bash
   llamafactory-cli train ../scripts/qwen2_5vl_full_sft_STEP.yaml
   ```

Refer to the project root [`../README.md`](../README.md) for the full STEP pipeline.

## Credits & License

LLaMA-Factory is authored by [hiyouga](https://github.com/hiyouga) and contributors and released
under the Apache-2.0 license — see [`LICENSE`](LICENSE).
