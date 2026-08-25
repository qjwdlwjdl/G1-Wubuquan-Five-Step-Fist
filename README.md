# G1 Five-Step Chain Fist — SuperSONIC Challenge Submission

## Project Overview

We taught a Unitree G1 humanoid a **Five-Step Chain Fist** — a 12.98-second
continuous Chinese wushu routine: five stance types (bow, horse, drop,
crossed-leg, empty) chained with straight punches and palm strikes, two
direction changes, and 3.06 m of forward travel — generated with
**NVIDIA Kimodo** text-to-motion (G1-skeleton variant, via Ultimate Bots
Studio) and *learned* by fine-tuning the **SONIC** whole-body control
foundation model. The stock model cannot perform the routine (it survives
only 5.4 s before falling); after fine-tuning, the robot executes the full
12.98 s routine without interruption.

## The Move

649 frames @ 50 fps = **12.98 s**; 5 stance types in sequence; peak joint
speed **45 rad/s** (fast straight punches); pelvis drops to **0.38 m** in the
lowest stance; the routine travels **3.06 m** forward while turning in both
directions.

## Why It's Hard

- **Long horizon, large root motion**: 13 s of continuous stance transitions
  with 3 m of travel — root tracking over a long horizon is the dominant
  challenge.
- **Wide dynamic range**: deep stances (0.38 m pelvis) combined with fast
  straight punches (45 rad/s) require precise ankle/hip coordination across
  very different pose regimes.
- **Physical plausibility**: full-body coordination under changing support
  stances requires physics/RL — the motion is not played back, it is
  executed by the policy under Isaac Sim physics.

## How We Did It

1. **Data**: generated the 649-frame, 50 fps, 29-DOF wushu sequence with
   NVIDIA Kimodo (G1-skeleton variant via Ultimate Bots Studio), converted it
   to SONIC motion_lib format (`convert_soma_csv_to_motion_lib.py`, fps 50),
   and validated: no NaN/Inf, no floor penetration, no teleporting.
2. **Fine-tuning (SONIC)**, starting from the official release checkpoint:
   - **V1**: 4000 iterations on the single motion (2048 envs).
     Result: mpjpe_g 179.4 mm — root drift remained high (172.0 mm), so plain
     fine-tuning alone was not enough for the long-horizon root motion.
   - **V1.1 (final)**: 1500 more iterations with `tracking_anchor_pos`
     weight 1.0 (std 0.2) and actor lr 1e-5 (root-focused refinement).
     Result: **mpjpe_g 128.2 mm, mpjpe_l 46.8 mm, root drift 118.6 mm** —
     down 24%/26% vs the stock baseline — and **all 32 eval envs now survive
     the full 649-step (12.98 s) motion** (the stock baseline dies at
     step 272).
3. **Evaluation**: official SONIC evaluation pipeline (32 envs,
   tracking-error termination `manager_env/terminations=tracking/eval`),
   dump of per-env trajectories (14 tracked bodies) in `eval/`.
4. **ONNX export** of the fine-tuned policy (`onnx/model_step_001500_*`,
   matching V1.1) for deployment.

## Results

| Metric | Stock SONIC (before) | Wubuquan V1 | **Fine-tuned V1.1 (after)** |
|---|---|---|---|
| mpjpe_g | 168.2 mm | 179.4 mm | **128.2 mm** |
| mpjpe_l | 42.9 mm | 48.4 mm | **46.8 mm** |
| root drift | 159.7 mm | 172.0 mm | **118.6 mm** |
| Survival (of 649 steps) | 272 (42%) | 648 (100%) | **648 (100%)** |

*Alternate candidate (backup):* a 5.14 s in-place martial-arts clip
(`action_clip_martial_fixed`), V1: mpjpe_g **103.8 mm** — ONNX
`onnx/model_step_003000_*` and eval data included too.

## Demo Videos

- Full-body (Isaac Sim, follow camera, failure-cut): this repository's
  release **`videos-v1`** — `wbq_before.mp4` / `wbq_after.mp4`.
- The "after" clip runs the complete 12.98 s routine; the "before" clip
  stops where the stock policy fails (4.56 s).

## Releases (this repository)

- `videos-v1` — before / after full-body videos (see above)
- `v1-checkpoints` — V1, V1.1 and base checkpoints
  (`wubuquan_v1.pt`, `wubuquan_step4000.pt`, `wubuquan_v11.pt`,
  `martial_clip_v1.pt`, `sonic_release_base.pt`)
- `run-config-v11` — exact `config.yaml` / `meta.yaml` of the V1.1 run
  (reproducibility; used together with `v1-checkpoints` and the commands in
  `config/training_config.md`)

## Repository Contents

- `README.md` — this write-up
- `config/` — training config & commands, dataset / ONNX cards
- `docs/` — `TUNING-NOTES.md`: full tuning campaign (V1 / V1.1 / two
  rejected experiments) and why V1.1 was chosen as final
- `eval/` — official eval trajectories (npz) and metrics
- `data/` — motion data (SONIC motion_lib pkl + raw Kimodo CSVs)
- `onnx/` — exported fine-tuned policy (`model_step_001500_*` final,
  `model_step_003000_*` backup clip)

## Credit

Motion Data by Bones Studio
