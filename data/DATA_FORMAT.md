# STEP-CoT · Data Format

Every STEP-CoT sample follows LLaMA-Factory's `sharegpt` format.

## Top-level schema

```json
{
  "messages": [
    {"role": "user",      "content": "<user prompt with <image> placeholders>"},
    {"role": "assistant", "content": "<optional <thinking>...</thinking><answer>...</answer>>"}
  ],
  "images": ["path/to/img_1.png", "path/to/img_2.png"]
}
```

- The number of `<image>` tokens inside the user turn **must** match `len(images)`.
- Image order in `images` matches their appearance in the prompt (RGB view usually comes
  first, followed by the BEV map).
- Keep paths inside `images` relative to the repository's `data/` directory. The supplied
  training recipe sets `media_dir: ../data`, which makes entries such as
  `images/example.png` portable across machines.
- Do not commit machine-specific absolute paths, credentials, access tokens, or
  unsanitized user data. Put local datasets in ignored files and use the checked-in
  `dataset_info.json` only as a portable registry.

## Task families

STEP-CoT covers three broad families. All of them share the top-level schema above; only
the prompt template and the answer style differ.

### 1. Point-Level Navigation (main task)

The agent receives:
- **Current RGB Viewpoint with Markers**: an egocentric frame where each reachable
  neighbouring region is labelled with a numeric marker via Grounded-SAM + Set-of-Mark.
- **Environmental Feature Map**: a BEV map centred on the agent, with a red arrow
  showing heading and light-gray / white regions denoting explored / unexplored areas.
- **Task Objective**: a short natural-language goal such as `"microwave"`.
- **Current Orientation**: an integer angle in `[0, 360°)`.

The assistant answers with:

```
<thinking>...why point X is the right waypoint given the objective and the map...</thinking>
<answer>X</answer>                # marker ID, or "RotateLeft" / "RotateRight" / "MoveAhead"
```

### 2. Task Breakdown

The agent receives a natural-language ALFRED instruction; the assistant outputs a
sequential decomposition (`1. go to <recep>. 2. pick up <obj>. …`) that matches the
expert plan. No image is required for pure text-only breakdown records, but multimodal
grounded breakdowns are also supported.

### 3. Primitive Skills (spatial-awareness boost)

| Sub-task | Input | Expected answer |
|----------|-------|-----------------|
| Topological Reasoning | BEV map + orientation + region query (e.g. *right-back*) | `<answer>Objects: A, B, C.</answer>` |
| Pose Estimation       | Egocentric RGB + BEV map | `<answer>{angle in [0, 360°]}</answer>` |
| Relative Orientation  | Two BEV maps / two views | `<answer>{delta angle}</answer>` |
| Semantic Inquiry      | BEV map + object query   | `<answer>{location description}</answer>` |

Dataset entries live in the project-level [`dataset_info.json`](dataset_info.json).
Families may be combined in `step_cot_train` or registered separately and mixed with
LLaMA-Factory's comma-separated `dataset:` syntax.

## Prompt conventions

- **Orientation legend** is repeated at the top of every prompt that references the BEV
  map: `0° = right, 90° = up, 180° = left, 270° = down`.
- **Map legend** clarifies the semantics of colour: `light gray = explored`,
  `white = unexplored`, `red arrow / red dot = agent pose`.
- **Section separators** are semicolons (`;`) so prompts can be trivially split and
  templated.

## Concrete examples

See [`samples.json`](samples.json) — it contains one record from each of
Topological Reasoning, Pose Estimation, and Point-Level Navigation.
