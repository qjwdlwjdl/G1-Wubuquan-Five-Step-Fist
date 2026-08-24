# Evaluation Metrics

Measured with the official SONIC evaluation pipeline (im_eval callback),
32 envs, same reference motion and config
(`manager_env/terminations=tracking/eval`).

| Metric | Stock SONIC (baseline) | Wubuquan V1 | **V1.1 (final)** |
|---|---|---|---|
| mpjpe_g (global) — mm | 168.2 | 179.4 | **128.2** |
| mpjpe_l (local, root-aligned) — mm | 42.9 | 48.4 | **46.8** |
| root drift — mm | 159.7 | 172.0 | **118.6** |
| Survival (of 649 steps) | 272/649 | 648/649 | **648/649** |

Vs. the stock baseline the final model reduces global MPJPE by **24%** and
root drift by **26%**, and every eval environment now completes the full
12.98 s motion (the stock baseline survives only 5.4 s).

## Backup clip (`action_clip_martial_fixed`, 5.14 s in-place)

| Metric | Stock | V1 |
|---|---|---|
| mpjpe_g | 138.8 mm | **103.8 mm** |
| mpjpe_l | — | **38.4 mm** |
| root drift | — | **95.8 mm** |

Raw rollout data (per-env trajectories, 14 tracked bodies):
- `eval_baseline-wbq.npz` — stock SONIC on the Five-Step Chain Fist
- `eval_v1-wbq.npz` — after 4000 iterations
- `eval_v11-wbq.npz` — final (V1.1)
- `eval_baseline-martial.npz` / `eval_v1-martial.npz` — backup clip
