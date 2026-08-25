# Writeup — G1 Wubuquan (Five-Step Fist)

*SuperSONIC Challenge | GHOST TRIAL 04 | Martial Arts track*

## What We Taught It

We fine-tuned NVIDIA's SONIC (GEAR-SONIC) whole-body control policy so a
Unitree G1 humanoid **performs a complete Wubuquan-inspired wushu routine**
— a continuous ~13.0-second sequence (649 samples @ 50 Hz) with five
stance types chained together — **straight punches and palm strikes, two
direction changes, and ~3.06 m of travel across the floor, ending back on
its feet**.

The motion sees the pelvis drop to **0.38 m** in the deepest stance and
return to full standing height. Because we generated the reference with
NVIDIA Kimodo text-to-motion (G1-skeleton variant, Ultimate Bots Studio)
rather than a mocap actor, we label the routine **Wubuquan-inspired**: the
steps follow the Five-Step Fist style (bow, horse, drop, crossed-leg and
empty stances), but we did not verify the sequence move by move against a
canonical syllabus, and we say so publicly.

The stock SONIC model cannot perform the move: it **physically collapses
at ~4.5 s**. After fine-tuning, the policy executes the full 13 s routine
— **in simulation** (Isaac Sim physics, no playback; the motion is
executed end-to-end by the learned policy).

## Why It's Hard

1. **Long horizon, large root motion.** 13 s of continuous stance
   transitions with 3 m of net displacement is at the edge of what the
   stock checkpoint's training distribution can track. Root (translation +
   heading) error accumulates monotonically over the sequence; the last
   three seconds are where tracking degrades first.

2. **Wide dynamic range.** The routine jumps between regimes — deep
   stances with a 0.38 m pelvis, fast straight punches, and quick weight
   shifts — in the same motion. A policy that is good at either extreme
   separately is not automatically good at alternating between them.

3. **Contact-modulated balance.** Each of the five stances changes the
   support polygon: feet plant, shift, pivot and step throughout. This is
   not a pattern-matching problem — the robot must stay balanced under
   Isaac Sim physics while the reference changes support. Hand/arm
   coordination matters too: the punches are fast enough that joint
   velocity, contact and balance must be controlled jointly.

4. **Evaluation ease is a trap.** The default SONIC evaluation
   termination (`tracking/eval`) ends an episode the moment tracking error
   exceeds thresholds. The stock model survives only the first 272 samples
   (42 %) — the easy part — while a good policy runs the whole 13 s.
   Comparing "before vs after" with that pipeline silently compares
   different time horizons and even makes local (root-aligned) MPJPE look
   worse, a pure artifact of early termination. A fair comparison needs a
   fixed-horizon protocol (below).

## How We Did It

### 1. Motion data (generated, not mocap)

- **Generated** the 649-sample, 50 Hz, 29-DOF sequence with NVIDIA Kimodo
  (G1-skeleton variant) via Ultimate Bots Studio.
- **Converted** to SONIC motion_lib format with the official
  `gear_sonic/data_process/convert_soma_csv_to_motion_lib.py` (`--fps 50`),
  producing the `part1_0_13s.pkl` motion-library entry.
- **Validated**: no NaN/Inf, every joint angle within G1 limits,
  replay-verified (no teleporting, no floor penetration). One retarget
  velocity transient exists at sample 322 (44.9 rad/s, multi-joint) — we
  investigated it, established it is a retarget/IK discontinuity rather
  than a characteristic motion speed, documented it in the dataset card,
  and do **not** advertise it as difficulty.

### 2. Fine-tuning (SONIC)

- **Setup**: one NVIDIA L40S, 2048 parallel envs, Isaac Sim 5.1.0 +
  Isaac Lab 2.3.2 + GEAR-SONIC, fine-tuning from the official SONIC
  release checkpoint.
- **V1**: 4000 iterations on the single motion. Result: 179.4 mm global
  MPJPE — root drift stayed high (172.0 mm), so plain fine-tuning was not
  enough for long-horizon root motion.
- **V1.1 (final)**: +1500 iterations of root-focused refinement with
  `tracking_anchor_pos` weight 1.0 (std 0.2) and actor lr 1e-5.
- **Rejected experiments (documented honestly)**: tightening the anchor
  penalty (std 0.15) collapsed the policy into a "correct pose, drifting
  root" optimum; extending the V1.1 recipe to 2500 iterations degraded
  monotonically. V1.1 at 1500 iterations is the peak of the single-motion
  pipeline (see `docs/TUNING-NOTES.md`).

### 3. Evaluation (fairness-first, two protocols)

We reran **every** policy — stock, V1, V1.1 — under both protocols:

| Protocol | Stock SONIC | V1 | **V1.1 (final)** |
|---|---|---|---|
| *Fixed-horizon, matched 0–4.5 s window (no tracking termination; stock collapses at ~4.5 s)* | | | |
| mpjpe_g | 198.1 mm | 108.4 mm | **76.0 mm (−62 %)** |
| mpjpe_l (root-aligned) | 41.4 mm | 35.8 mm | **33.0 mm (−20 %)** |
| mean root-position error | 190.1 mm | 100.7 mm | **64.1 mm (−66 %)** |
| *Fixed-horizon, full rollout (stock 225 samples, V1/V1.1 all 649)* | | | |
| mpjpe_g | 198.1 mm | 180.3 mm | **133.1 mm** |
| *Official SONIC `tracking/eval` (early termination; kept for comparability)* | | | |
| mpjpe_g | 168.2 mm | 179.4 mm | **128.2 mm** |
| Reached final reference sample | 272 (42 %) | 648 | **648 (100 %)** |

Under the fair matched-window protocol **every metric improves**: global
MPJPE −62 %, root error −66 %, and local MPJPE −20 % (the official
pipeline's "local regression" disappears once time horizons match).

### 4. Deliverables

- **ONNX exports** of the final policy (V1.1, `model_step_001500_*`,
  five files: g1 / encoder / decoder / smpl / teleop) + HF model repo.
- **Motion data + conversion commands** ; dataset repo; config cards for
  training, evaluation and ONNX export; exact V1.1 run config
  (`run-config-v11`) for reproducibility.
- **Before/after videos** (Isaac Sim, follow camera): `wbq_before.mp4`
  (stock — collapses at ~4.5 s), `wbq_after.mp4` (V1.1 — full routine),
  `wbq_before_after.mp4` (labelled side-by-side).
- **Licensing**: fine-tuned weights are derivatives of NVIDIA SONIC —
  distributed under the **NVIDIA Open Model License** with NOTICE
  attribution; the original GPU base model is not re-hosted (it comes from
  the official SONIC release). Motion data is credited to Bones Studio.

*Reproducible commands, exact configs, raw trajectories and the full tuning
history are in this repository (`config/`, `eval/`, `docs/`).*
