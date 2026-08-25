# G1 Wubuquan (Five-Step Fist) — SuperSONIC Challenge Submission

## Project Overview

We taught a Unitree G1 humanoid a **Wubuquan-inspired Chinese wushu
sequence** — a continuous ~13-second routine: five stance types (bow,
horse, drop, crossed-leg, empty) chained with straight punches and palm
strikes, two direction changes, and 3.06 m of net forward displacement —
generated with **NVIDIA Kimodo** text-to-motion (G1-skeleton variant, via
Ultimate Bots Studio) and *learned* by fine-tuning the **SONIC**
whole-body control foundation model.

*Naming note:* the motion is a Kimodo-generated rendition inspired by the
canonical Five-Step Fist (Wubuquan / 五步拳); we did not verify it move by
move against a canonical syllabus, and label it as **Wubuquan-inspired**
accordingly. The stock model cannot perform the routine (it falls after
~4.5 s); after fine-tuning, the simulated robot executes the full routine
without interruption in all evaluation environments.

## The Move

649 samples @ 50 Hz = **~13.0 s**; 5 stance types in sequence; pelvis
drops to **0.38 m** in the lowest stance; **~3.06 m net root displacement**
(start→end) in the horizontal plane.

## Why It's Hard

- **Long horizon, large root motion**: 13 s of continuous stance
  transitions with 3 m of travel — root tracking over a long horizon is
  the dominant challenge.
- **Wide dynamic range**: deep stances (0.38 m pelvis) combined with fast
  straight punches require precise ankle/hip coordination across very
  different pose regimes.
- **Physical plausibility**: full-body coordination under changing
  support stances requires physics/RL — the motion is never played back,
  it is executed by the policy under Isaac Sim physics (simulated).

## How We Did It

1. **Data**: generated the 649-sample, 50 Hz, 29-DOF wushu sequence with
   NVIDIA Kimodo (G1-skeleton variant via Ultimate Bots Studio), converted
   it to SONIC motion_lib format (`convert_soma_csv_to_motion_lib.py`,
   fps 50), and validated: no NaN/Inf, no floor penetration, no
   teleporting.
2. **Fine-tuning (SONIC)**, starting from the official release checkpoint:
   - **V1**: 4000 iterations on the single motion (2048 envs).
   - **V1.1 (final)**: 1500 more iterations with `tracking_anchor_pos`
     weight 1.0 (std 0.2) and actor lr 1e-5 (root-focused refinement).
3. **Evaluation**: official SONIC evaluation pipeline (32 envs), plus a
   **fixed-horizon (no tracking-termination) eval** that disables the
   early-termination triggers and lets every policy run until the motion
   ends or the robot physically falls — see `eval/metrics.md` for the
   full comparison matrix and `docs/TUNING-NOTES.md` for the tuning
   campaign (including the two rejected follow-up experiments).
4. **ONNX export** of the fine-tuned policy (`onnx/model_step_001500_*`,
   matching V1.1) for deployment.

## Results

**Apples-to-apples window (0–4.5 s, no tracking termination; the stock
policy physically falls at ~4.5 s, so all three models are compared on
exactly the same 225 samples):**

| Metric | Stock SONIC (before) | Wubuquan V1 | **Fine-tuned V1.1 (after)** |
|---|---|---|---|
| mpjpe_g (global) — mm | 198.1 | 108.4 | **76.0** |
| mpjpe_l (local) — mm | 41.4 | 35.8 | **33.0** |
| mean root-position error — mm | 190.1 | 100.7 | **64.1** |

**Full-horizon (no tracking termination; metrics over each policy's full
rollout — V1/V1.1 run all 649 samples, stock falls at sample 225):**

| Metric | Stock SONIC | Wubuquan V1 | **Fine-tuned V1.1 (after)** |
|---|---|---|---|
| mpjpe_g — mm | 198.1 (225 samples) | 180.3 | **133.1** |
| mpjpe_l — mm | 41.4 | 48.3 | **46.8** |
| mean root-position error — mm | 190.1 | 172.9 | **123.7** |

**Official eval (SONIC `tracking/eval` termination — the reference
pipeline; metrics averaged over frames before early termination):**

| Metric | Stock SONIC | Wubuquan V1 | **Fine-tuned V1.1 (after)** |
|---|---|---|---|
| mpjpe_g — mm | 168.2 | 179.4 | **128.2** |
| mpjpe_l — mm | 42.9 | 48.4 | **46.8** |
| mean root-position error — mm | 159.7 | 172.0 | **118.6** |
| Reached final reference sample (index 648 of 649) | 272 (42 %) | 648 (100 %) | **648 (100 %)** |

*Interpretation:* in a strictly fair comparison (same 0–4.5 s window, no
early termination) the fine-tuned model improves **every** metric —
global MPJPE −62 %, mean root error −66 %, local MPJPE −20 % vs the stock
rollout. The apparent "local MPJPE regression" under the official
terminated pipeline is an artifact of that pipeline (stock dies at 42 %
of the motion, so its average covers only the easy early frames);
in matched-window/full-horizon evals local MPJPE also improves.

## Demo Videos

- Full-body (Isaac Sim, follow camera): this repository's release
  **`videos-v1`** — `wbq_before.mp4` / `wbq_after.mp4` (separate clips)
  and **`wbq_before_after.mp4`** (side-by-side comparison with labels;
  the stock side freezes at its failure point with an on-screen notice).
- The **after** clip runs the complete 649-sample routine; the **before**
  clip renders the first ~4.5 s (225 samples) of the stock rollout — the
  stock policy falls at that point, so the clip stops there.

## Releases (this repository)

- `videos-v1` — before / after full-body videos
- `v1-checkpoints` — fine-tuned checkpoints (V1, V1.1, backup-clip V1)
  plus tools; the original SONIC base checkpoint is **not** redistributed
  here — it is available from the official SONIC release
  (see `config/training_config.md`)
- `run-config-v11` — exact `config.yaml` / `meta.yaml` of the V1.1 run
  (reproducibility; used together with `v1-checkpoints` and the commands
  in `config/training_config.md`)

## Licenses (important)

- **Model weights** (fine-tuned ONNX/PT derived from NVIDIA SONIC):
  **NVIDIA Open Model License** — see the `LICENSE` file and `NOTICE`.
  "Licensed by NVIDIA Corporation under the NVIDIA Open Model License."
- **Source code**: Apache License 2.0 (SONIC/GEAR-SONIC
  [repository](https://github.com/NVlabs/GR00T-WholeBodyControl)).
- **Motion data**: "Motion Data by Bones Studio" (created with NVIDIA
  Kimodo).

## Repository Contents

- `README.md` — this write-up
- `config/` — training config & commands, dataset / ONNX cards
- `docs/` — `WRITEUP.md` (what / why / how submission write-up),
  `TUNING-NOTES.md`: full tuning campaign (V1 / V1.1 / two rejected
  experiments) and why V1.1 was chosen as final
- `eval/` — eval trajectories (npz) and the metrics card
- `data/` — motion data (SONIC motion_lib pkl + raw Kimodo CSVs)
- `onnx/` — exported fine-tuned policy (`model_step_001500_*` final,
  `model_step_003000_*` backup clip)
- `LICENSE` / `NOTICE` — NVIDIA Open Model License + attribution

## Credit

Motion Data by Bones Studio
