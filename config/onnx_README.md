---
license: nvidia-open-model-license
pipeline_tag: reinforcement-learning
tags:
- robotics
- humanoid
- sonic
- g1
---

# G1 Wubuquan (Five-Step Fist) — Fine-tuned SONIC Policy (ONNX)

Fine-tuned SONIC whole-body control policy for the SuperSONIC Challenge
(Martial Arts track). Teaches a Unitree G1 humanoid a ~13.0-second
Kimodo-generated Wubuquan-inspired martial-arts routine.

> **License:** these weights are derivative models of NVIDIA's SONIC
> model and are licensed under the **NVIDIA Open Model License**
> (see the `LICENSE` / `NOTICE` files in the GitHub repository).
> "Licensed by NVIDIA Corporation under the NVIDIA Open Model License."

## Files (final model — V1.1)
- `model_step_001500_g1.onnx` — **main policy** (G1 robot)
- `model_step_001500_encoder.onnx` / `model_step_001500_decoder.onnx` — motion token encoder/decoder
- `model_step_001500_teleop.onnx` — teleoperation variant
- `model_step_001500_smpl.onnx` — SMPL variant

## Results (fair, apples-to-apples: same 0–4.5 s window, no tracking termination)

| Metric | Stock SONIC | Fine-tuned (V1.1) |
|---|---|---|
| mpjpe_g | 198.1 mm | **76.0 mm** (−62 %) |
| mpjpe_l | 41.4 mm | **33.0 mm** (−20 %) |
| mean root-position error | 190.1 mm | **64.1 mm** (−66 %) |

Full-horizon (same eval config): V1.1 **133.1 mm** mpjpe_g / 46.8 mm
mpjpe_l / 123.7 mm root error over all 649 samples. The complete metric
matrix (matched-window, full-horizon, and the official terminated
`tracking/eval` pipeline) is in the GitHub repository's `eval/metrics.md`.

## Training
Fine-tuned from the official GEAR-SONIC release checkpoint: 4000 iters
(V1) + 1500 iters root-focused refinement (tracking_anchor_pos weight 1.0,
std 0.2, actor lr 1e-5), single L40S. The exact V1.1 run config
(`run-config-v11` release) and reproducible commands are in the GitHub
repository `config/training_config.md`; the tuning campaign (including
two rejected follow-up experiments) is documented in
`docs/TUNING-NOTES.md`.

## Backup policy
`model_step_003000_*` (in the GitHub repository and the HF companion)
— the alternate 5.14 s in-place martial-arts clip fine-tune (V1,
mpjpe_g 103.8 mm).

## Github companion repository
https://github.com/qjwdlwjdl/G1-Wubuquan-Five-Step-Fist — README, config
cards, eval trajectories, raw motion data, LICENSE/NOTICE.
