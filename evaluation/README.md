# STEP · Evaluation

STEP is evaluated on two embodied benchmarks:

1. **ALFRED** — Household instruction following (train / valid-seen / valid-unseen /
   tests-seen / tests-unseen splits) with metrics *Success Rate (SR)*, *Goal-Condition
   Success (GC)*, and their path-length-weighted variants (*PLWSR*, *PLWGC*).
2. **AI2-THOR-Nav** — Cross-task generalization to point-goal / object-goal navigation
   inside the same simulator.

Both stacks are executed inside the AI2-THOR simulator, so evaluating STEP requires the
official ALFRED codebase and its simulator dependencies.

> [!IMPORTANT]
> This release documents the evaluation protocol but does not yet include the executable
> STEP-to-ALFRED adapter. Exact end-to-end reproduction therefore also requires the adapter,
> checkpoint, and episode configuration that will be published in a follow-up release.

## Upstream evaluator: use ALFRED as-is

We do **not** fork or vendor the evaluator — running numbers should be reproducible against
the community-standard implementation. Please clone the official repository:

- **ALFRED**: <https://github.com/askforalfred/alfred>

  ```bash
  git clone https://github.com/askforalfred/alfred.git
  cd alfred
  # Follow the upstream INSTALL / README.md for AI2-THOR + data preparation.
  ```

  Relevant entry points inside ALFRED:
  - `scripts/check_thor.py` — validate the simulator install.
  - `models/eval/eval_seq2seq.py` — the reference task evaluator used to report
    SR / GC / PLWSR / PLWGC on `valid_seen` and `valid_unseen`.
  - `data/` — expert trajectories that STEP-CoT distills into Chain-of-Thought supervision.

- **AI2-THOR-Nav**: use AI2-THOR's built-in navigation tasks
  (<https://github.com/allenai/ai2thor>) or any compatible point-/object-goal benchmark
  (e.g. RoboTHOR) with the same episode splits as reported in the paper.

## Bridging STEP to the ALFRED evaluator

Because STEP outputs *point-level* waypoints instead of atomic actions, a thin adapter is
required between the STEP policy and ALFRED's `THORConnector` action interface. The adapter is
a straightforward mapping and is described in the paper — a reference implementation will be
added to this folder in a follow-up commit. In the meantime, the recipe is:

1. Serve the STEP checkpoint (e.g. with `vllm` or `llamafactory-cli chat`).
2. At every step, feed:
   - the current egocentric RGB with Set-of-Mark labels on reachable regions
     (Grounded-SAM + SoM, see Sec. 3.2 of the paper), and
   - the running BEV map produced by the Environment-Filtering module.
3. Parse the returned `<answer>` — if it is a marker index, teleport / navigate to the
   corresponding point; if it is `MoveAhead` / `RotateLeft` / `RotateRight`, forward it
   directly to `THORConnector`.
4. Feed the resulting frame back into step 2 until the episode terminates.

## Reproducing the reported numbers

Please refer to the paper's Section 4 for hyper-parameters (temperature = 0, top-p = 1,
max output tokens = 512, max steps = 1000 for ALFRED). The reported runs use ALFRED's
default `valid_seen` / `valid_unseen` episode order. The exact adapter implementation and
run configuration will be released here so those claims can be independently verified.

## Citation

If you use ALFRED or AI2-THOR, please cite the corresponding papers alongside STEP:

```bibtex
@inproceedings{shridhar2020alfred,
  title={ALFRED: A Benchmark for Interpreting Grounded Instructions for Everyday Tasks},
  author={Shridhar, Mohit and Thomason, Jesse and Gordon, Daniel and Bisk, Yonatan and
          Han, Winson and Mottaghi, Roozbeh and Zettlemoyer, Luke and Fox, Dieter},
  booktitle={CVPR},
  year={2020}
}

@article{kolve2017ai2thor,
  title={AI2-THOR: An Interactive 3D Environment for Visual AI},
  author={Kolve, Eric and Mottaghi, Roozbeh and Han, Winson and VanderBilt, Eli and
          Weihs, Luca and Herrasti, Alvaro and Gordon, Daniel and Zhu, Yuke and
          Gupta, Abhinav and Farhadi, Ali},
  journal={arXiv preprint arXiv:1712.05474},
  year={2017}
}
```
