---
license: other
task_categories:
- robotics
tags:
- humanoid
- motion
- g1
---

# G1 Wubuquan (Five-Step Fist) — Training Dataset

Single generated motion used to fine-tune SONIC (SuperSONIC Challenge,
Martial Arts track).

## Contents
- `part1_0_13s.pkl` — SONIC motion_lib format (root_trans_offset, pose_aa,
  dof, root_rot, smpl_joints), 649 samples @ 50 Hz, G1 29-DOF
- `raw_part1_0_13s/` — source CSV bundle (joint_pos/vel, body_pos/quat/vel)
  from the Kimodo export used for conversion
- `raw_action_clip_martial_fixed/` — shorter 5.14 s in-place backup clip
  (8 CSV files, same layout) used for the alternate V1 checkpoint
  (`model_step_003000_*`); the submission policy trains on the primary
  12.5+ s motion only

## Motion

A **Wubuquan-inspired** Chinese wushu sequence (generated, not canonical
syllabus): five stance types (bow, horse, drop, crossed-leg, empty)
chained with straight punches and palm strikes, two direction changes,
**~3.06 m net root displacement** in the horizontal plane (start→end;
the root-trajectory path length is ~4.16 m). Duration: 649 samples @ 50 Hz
= ~13.0 s. Generated with NVIDIA Kimodo (G1-skeleton variant) via
Ultimate Bots Studio; converted with `convert_soma_csv_to_motion_lib.py`
(fps 50).

## Validation & known artifacts

- No NaN/Inf; every joint angle within the G1 joint-limit range;
  replay-verified (no teleporting, no floor penetration).
- **Velocity spike (retarget transient, investigated):** the raw
  `joint_vel.csv` contains a transient of up to 44.9 rad/s at frame 322
  (DOFs 11/15/19 co-occurring at 44.9/−30.7/31.3 rad/s); the underlying
  joint position jumps ~0.9 rad within 40 ms and then settles for the
  rest of the clip — a retarget/IK discontinuity rather than a ballistic
  motion characteristic. The converted motion remains within joint
  limits, but this transient is **not** advertised as character speed
  anywhere in this submission.

## Usage
```python
d = joblib.load('part1_0_13s.pkl')
entry = d['part1_0_13s']   # root_trans_offset, pose_aa, dof, root_rot, fps=50
```

## Credit
Motion Data by Bones Studio
