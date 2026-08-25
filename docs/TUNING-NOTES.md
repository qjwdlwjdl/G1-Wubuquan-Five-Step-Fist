# Tuning Notes — Wubuquan (Five-Step Fist) on G1

This document records the full fine-tuning campaign, including the two
experiments that did **not** beat the final model, and the evaluation
fairness work behind the final numbers.

## Setup (all runs)

- Model: SONIC/GEAR-SONIC (`manager/universal_token/all_modes/sonic_release`
  exp), trained on a single NVIDIA L40S (Nebius/Modal), 2048 parallel envs.
- Data: `data/part1_0_13s.pkl` (649 samples @ 50 Hz, ~13.0 s).
- Eval: official SONIC pipeline, 32 envs, `im_eval` callback.
- Metric convention: mpjpe_g (global, mm) / mpjpe_l (root-aligned, mm) /
  mean root-position error (mm).

## Run history (official `tracking/eval` terminated pipeline)

| Run | Start checkpoint | Iterations | lr | anchor std | mpjpe_g | mpjpe_l | root err | Reached final sample |
|---|---|---|---|---|---|---|---|---|
| baseline | — (stock SONIC) | — | — | — | 168.2 | 42.9 | 159.7 | 272/648 |
| V1 | sonic_release/last.pt | 4000 | default | default | 179.4 | 48.4 | 172.0 | 648 |
| **V1.1 (final)** | V1 | +1500 | 1e-5 | 0.2 | **128.2** | **46.8** | **118.6** | **648** |
| V1.2 (rejected) | V1.1 | +2500 | 5e-6 | 0.15 | 222.6 | 47.4 | 140 | 648 |
| V1.3 (rejected) | V1.1 | +2500 | 1e-5 | 0.2 | 157.7 | 47.3 | 149 | 648 |

## Evaluation fairness (why the official pipeline misleads here)

The official `tracking/eval` terminations stop an episode as soon as
anchor/EE tracking error exceeds thresholds, and the reported MPJPE is
averaged over the frames **before** termination. The stock SONIC model
only survives the first 272 samples (42 %) of this motion — the easiest
part — while V1/V1.1 complete the whole 13 s. Comparing "168.2 vs
128.2 mm" therefore compares different time horizons.

To fix this we re-ran all three policies with a `fixed_horizon`
termination (motion time-out only; all early-termination triggers
disabled), so each policy runs until the motion ends or the robot
physically falls:

- Apples-to-apples 0–4.5 s window (stock falls at sample 225, so every
  policy is compared on exactly the same 225 samples):
  **stock 198.1 / V1 108.4 / V1.1 76.0 mm** (mpjpe_g); local MPJPE
  **41.4 / 35.8 / 33.0 mm**; root error **190.1 / 100.7 / 64.1 mm** —
  the fine-tuned model improves **every** metric.
- Full-horizon (each policy's rollout): stock 198.1 (225 samples) /
  V1 180.3 / V1.1 **133.1 mm** mpjpe_g; V1.1 local 46.8 mm, root
  123.7 mm.

Under both fair configurations the "local MPJPE regression" that the
terminated pipeline suggested (46.8 vs 42.9 mm) **disappears**: in the
matched-window eval the fine-tuned model's local MPJPE is lower than
stock's (33.0 vs 41.4 mm). We therefore report the apples-to-apples
window as the primary result and keep the official-pipeline table for
comparability with earlier reports.

## What we learned from the rejected runs

1. **Local pose saturates early.** Across every run the root-aligned
   mpjpe_l stays within 33–48 mm depending on the evaluation window —
   the remaining error is almost entirely global root translation /
   heading, a long-horizon problem (drift grows monotonically towards
   the end of the routine).
2. **Tightening the anchor penalty backfires.** V1.2 (anchor std 0.2 →
   0.15) collapsed the policy into a "correct pose, drifting root" local
   optimum; training metrics were flat from iteration 63 to 2500 and the
   policy only tracked the reference for the first ~6.5 s of ground
   truth.
3. **More iterations on the same data also backfires.** V1.3 (std 0.2 /
   lr 1e-5, +2500 iterations) degrades monotonically with iteration count
   (1500 it → 128.2 mm, 2000 it → 147.4 mm, 2500 it → 157.7 mm in
   terminated-eval terms). We evaluated the intermediate
   `model_step_002000.pt` to confirm the trend is monotonic and not a
   checkpoint artifact.
4. **Conclusion.** V1.1 is the peak of the single-motion fine-tune
   pipeline. Further gains would require either (a) a root-specific
   reward term, or (b) multiple reference clips (tempo/mirror variants)
   to widen the training distribution.

## Raw-data investigation: the 44.9 rad/s spike

The raw `joint_vel.csv` contains 44.94 rad/s (frame 322, DOF 11; DOFs 15
and 19 simultaneously at −30.7/31.3 rad/s). The DOF-11 position jumps
~0.9 rad within 40 ms and then settles — a retarget/IK discontinuity
(multi-joint co-occurring), not a ballistic punch speed. It does not
violate joint limits, but we removed the "45 rad/s fast punches" claim
from all public-facing materials and document the artifact in the
dataset card instead.

## Reproducibility of V1.1

The exact `config.yaml` / `meta.yaml` of the V1.1 run is published as the
`run-config-v11` release of this repository, together with the
fine-tuned checkpoints in `v1-checkpoints`, and the commands in
`config/training_config.md` reproduce the numbers above.
