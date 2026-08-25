# Evaluation Metrics

Measured with the official SONIC evaluation pipeline (`im_eval` callback,
32 envs) on the Wubuquan reference motion (649 samples @ 50 Hz).
Terminology:

- **mpjpe_g** — mean per-joint position error in global (world) space.
- **mpjpe_l** — per-joint error after aligning the predicted and reference
  root (root-aligned, "local").
- **mean root-position error** — per-frame Euclidean norm of the
  root/pelvis position error, averaged over frames (this is what earlier
  versions of this doc called "root drift"; it is not a cumulative
  drift measure).

## A. Apples-to-apples window (fair comparison)

No tracking termination (`terminations=fixed_horizon`, time-out only).
The stock policy physically falls at sample 225 (~4.5 s), so all three
policies are compared on **exactly the same 225 samples** (0–4.5 s):

| Metric | Stock SONIC | Wubuquan V1 | **Fine-tuned V1.1 (final)** |
|---|---|---|---|
| mpjpe_g — mm | 198.1 | 108.4 | **76.0** |
| mpjpe_l — mm | 41.4 | 35.8 | **33.0** |
| mean root-position error — mm | 190.1 | 100.7 | **64.1** |

V1.1 vs stock in this window: mpjpe_g **−62 %**, root error **−66 %**,
mpjpe_l **−20 %** — every metric improves.

## B. Full-horizon (no tracking termination)

Same eval configuration; metrics are averaged over each policy's full
rollout (V1 and V1.1 cover all 649 samples; the stock policy falls at
sample 225 and stops producing tracking data):

| Metric | Stock SONIC | Wubuquan V1 | **Fine-tuned V1.1 (final)** |
|---|---|---|---|
| mpjpe_g — mm | 198.1 (225 samples) | 180.3 | **133.1** |
| mpjpe_l — mm | 41.4 | 48.3 | **46.8** |
| mean root-position error — mm | 190.1 | 172.9 | **123.7** |

## C. Official eval (SONIC `tracking/eval` termination)

The reference pipeline with early termination (anchor/EE thresholds);
metrics are averaged over the frames **before** termination, which favors
models that terminate early (stock dies at 42 % of the motion) — we keep
it for comparability with our earlier reports, and note the caveat:

| Metric | Stock SONIC | Wubuquan V1 | **Fine-tuned V1.1 (final)** |
|---|---|---|---|
| mpjpe_g — mm | 168.2 | 179.4 | **128.2** |
| mpjpe_l — mm | 42.9 | 48.4 | **46.8** |
| mean root-position error — mm | 159.7 | 172.0 | **118.6** |
| Reached final reference sample (index 648 of 649) | 272 (42 %) | 648 (100 %) | **648 (100 %)** |

*Caveat:* the official pipeline's mpjpe_l for stock (42.9 mm) is measured
on its early, easy frames only. Under matched-window/full-horizon evals
(Sections A/B) the fine-tuned model's local MPJPE is **lower** than
stock's (33.0 vs 41.4 mm), so there is no real local-tracking regression.

## Backup clip (`action_clip_martial_fixed`, 5.14 s in-place)

Official terminated eval:

| Metric | Stock | V1 |
|---|---|---|
| mpjpe_g — mm | 138.8 | **103.8** |
| mpjpe_l — mm | — | **38.4** |
| mean root-position error — mm | — | **95.8** |

Raw rollout data (per-env trajectories, 14 tracked bodies):
- `eval_baseline-wbq.npz` — stock SONIC (official terminated eval)
- `eval_v1-wbq.npz` — V1 (official terminated eval)
- `eval_v11-wbq.npz` — V1.1 (official terminated eval)
- `eval_fh-stock.npz`, `eval_fh-v1.npz`, `eval_fh-v11.npz` — no-tracking-
  termination fixed-horizon evals for Sections A/B (per-env trajectories
  with 225/649/649 samples respectively)
- `eval_baseline-martial.npz` / `eval_v1-martial.npz` — backup clip
