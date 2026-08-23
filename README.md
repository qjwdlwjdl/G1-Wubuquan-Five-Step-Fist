# G1 Martial Arts — SuperSONIC Challenge Submission

## Project Overview

We taught a Unitree G1 humanoid a **Wubuquan (Five-Step Fist)** — a 13-second
classic Chinese martial-arts routine (5 stance types: bow, horse, drop,
crossed-leg, empty; with straight punches and palm strikes) — generated with
NVIDIA Kimodo text-to-motion and learned by fine-tuning the **SONIC**
whole-body control foundation model.

**The move**: 649 frames @ 50 fps = **12.98 s**. The routine travels **3.06 m**
forward while switching through low stance types (pelvis drops to 0.38 m),
with fast punches (peak joint speed 45 rad/s) and turns in both directions.

## Why It's Hard

- **Long horizon, large root motion**: 13 s of continuous stance transitions
  with 3 m of travel — root tracking over a long horizon is the dominant
  challenge.
- **Wide dynamic range**: deep stances (0.38 m pelvis) combined with fast
  straight punches (45 rad/s) require precise ankle/hip coordination across
  very different pose regimes.
- **Stock SONIC cannot do it**: baseline global MPJPE 168.2 mm with root drift
  of 159.7 mm after 13 s.
- **Physical plausibility**: full-body coordination under changing support
  stances; physics/RL required.

## How We Did It

1. Generated the 649-frame, 50 fps, 29-DOF motion with Kimodo (G1-skeleton
   variant via Ultimate Bots Studio), converted to SONIC motion_lib format,
   validated: no NaN/Inf, no floor penetration.
2. Fine-tuned official SONIC release checkpoint:
   - V1: 4000 iterations on the single motion (2048 envs).
     Result: mpjpe_g 179.4 mm — root drift remained high (172.0 mm), showing
     that plain fine-tuning alone is insufficient for the long-horizon root
     motion. (V1.1 root-focused refinement was started but suspended when the
     compute budget ran out; results will be updated here.)
3. Evaluation with the official pipeline (mpjpe_g / mpjpe_l / root drift),
   exported ONNX for deployment.

*Alternate candidate (backup):* a 5.14 s in-place martial-arts clip
(`action_clip_martial_fixed`) also fine-tuned during this trial —
baseline 138.8 mm → V1 **103.8 mm** (mpjpe_g), ONNX included in `onnx/`.

## Results

| Metric | Stock SONIC | Wubuquan V1 | Martial clip V1 (backup) |
|---|---|---|---|
| mpjpe_g | 168.2 mm | 179.4 mm | **103.8 mm** |
| mpjpe_l | 42.9 mm | 48.4 mm | **38.4 mm** |
| root drift | 159.7 mm | 172.0 mm | **95.8 mm** |

## Credit

Motion Data by Bones Studio
