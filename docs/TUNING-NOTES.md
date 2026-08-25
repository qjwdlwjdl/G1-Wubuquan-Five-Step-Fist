# Tuning Notes — Five-Step Chain Fist (G1)

This document records the full fine-tuning campaign for the Five-Step Chain
Fist policy, including the two experiments that did **not** beat the final
model, and why.

## Setup (all runs)

- Model: SONIC/GEAR-SONIC (`manager/universal_token/all_modes/sonic_release`
  exp), trained on a single NVIDIA L40S (Nebius/Modal), 2048 parallel envs.
- Data: `part1_0_13s.pkl` (12.98 s, 649 frames @ 50 fps).
- Eval: official SONIC pipeline, 32 envs, `im_eval` callback,
  `manager_env/terminations=tracking/eval`, same noise seeds.
- Metric convention: mpjpe_g (global, mm) / mpjpe_l (root-aligned, mm) /
  root drift (mm).

## Run history

| Run | Start checkpoint | Iterations | lr | anchor std | mpjpe_g | mpjpe_l | root drift | Survival 649 |
|---|---|---|---|---|---|---|---|---|
| basel | — (stock SONIC) | — | — | — | 168.2 | 42.9 | 159.7 | 272 |
| V1 | sonic_release/last.pt | 4000 | default | default | 179.4 | 48.4 | 172.0 | 648 |
| **V1.1 (final)** | V1 | +1500 | 1e-5 | 0.2 | **128.2** | **46.8** | **118.6** | **648** |
| V1.2 (rejected) | V1.1 | +2500 | 5e-6 | 0.15 | 222.6 | 47.4 | 140 | 648 |
| V1.3 (rejected) | V1.1 | +2500 | 1e-5 | 0.2 | 157.7 | 47.3 | 149 | 648 |

## What we learned

1. **Local pose saturates early.** Across every run the root-aligned
   mpjpe_l stays within 46.8–47.4 mm. All remaining error is global root
   translation/heading drift, and it is a long-horizon problem: drift grows
   monotonically from ~0.02 s bins to the final 3 s of the routine
   (0.30 m+ bins when degraded).
2. **Tightening the anchor penalty backfires.** V1.2 reduced `anchor std`
   0.2 → 0.15. The policy collapsed into a "correct pose, drifting root"
   local optimum: training metrics were flat from iteration 63 to 2500 and
   the policy only tracked the reference for the first ~6.5 s of ground
   truth motion.
3. **More iterations on the same data also backfires.** V1.3 kept the
   V1.1 recipe (std 0.2 / lr 1e-5) but trained 2500 more iterations.
   Global error degrades monotonically with iteration count
   (1500 it → 128.2 mm, 2000 it → 147.4 mm, 2500 it → 157.7 mm): after the
   V1.1 refinement the loss landscape around this checkpoint is over-fit for
   1500 iterations, and continued optimization erodes the root-tracking
   components while the local pose stays flat. We evaluated the intermediate
   `model_step_002000.pt` to confirm the trend is monotonic, not a
   checkpoint-storage artifact.
4. **Conclusion.** V1.1 is the peak of this data/recipe pipeline: the
   single-motion fine-tune converges at 1500 iterations, and no amount of
   continued optimization on the same single motion recovers the root
   accuracy lost after that point. Further gains would require either
   (a) a root-specific reward term, or (b) multiple reference clips
   (tempo / mirror variants) to widen the training distribution — the
   latter is planned as a follow-up, and would also strengthen the
   "difficulty & originality" case.

## Reproducibility of V1.1

The exact `config.yaml` / `meta.yaml` of the V1.1 run is published as the
`run-config-v11` release of this repository (2 small files), together with
the checkpoints in `v1-checkpoints`. Re-running the commands in
`config/training_config.md` from those files reproduces the numbers above.
