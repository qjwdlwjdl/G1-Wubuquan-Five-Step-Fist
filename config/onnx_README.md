---
license: apache-2.0
pipeline_tag: reinforcement-learning
tags:
- robotics
- humanoid
- sonic
- g1
---

# G1 Five-Step Chain Fist — Fine-tuned SONIC Policy (ONNX)

Fine-tuned SONIC whole-body control policy for the SuperSONIC Challenge
(Martial Arts track). Teaches a Unitree G1 humanoid a 12.98-second
Five-Step Chain Fist wushu routine.

## Files (final model — V1.1)
- `model_step_001500_g1.onnx` — **main policy** (G1 robot)
- `model_step_001500_encoder.onnx` / `model_step_001500_decoder.onnx` — motion token encoder/decoder
- `model_step_001500_teleop.onnx` — teleoperation variant
- `model_step_001500_smpl.onnx` — SMPL variant

## Results (vs stock SONIC)
| Metric | Stock | Fine-tuned (V1.1) |
|---|---|---|
| mpjpe_g | 168.2 mm | **128.2 mm** |
| mpjpe_l | 42.9 mm | **46.8 mm** |
| root drift | 159.7 mm | **118.6 mm** |
| Survival (649 steps) | 272 | **648** |

## Training
Fine-tuned from the official GEAR-SONIC release checkpoint: 4000 iters
(V1) + 1500 iters root-focused refinement (tracking_anchor_pos weight 1.0,
std 0.2, actor lr 1e-5), single L40S. See the GitHub repo
`config/training_config.md` for reproducible commands.

## Backup policy
`model_step_003000_*` (on the GitHub repo) — the alternate 5.14 s in-place
martial-arts clip fine-tune (V1, mpjpe_g 103.8 mm).

## Credit
Motion Data by Bones Studio
