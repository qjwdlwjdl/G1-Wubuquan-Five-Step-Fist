---
license: other
task_categories:
- robotics
tags:
- humanoid
- motion
- g1
---

# G1 Five-Step Chain Fist — Training Dataset

Single generated motion used to fine-tune SONIC (SuperSONIC Challenge,
Martial Arts track).

## Contents
- `part1_0_13s.pkl` — SONIC motion_lib format (root_trans_offset, pose_aa,
  dof, root_rot, smpl_joints), 649 frames @ 50 fps, G1 29-DOF
- `raw/` — source CSV bundle (joint_pos/vel, body_pos/quat/vel) from the
  Kimodo export used for conversion

## Motion
12.98-second Chinese wushu routine — five stance types (bow, horse, drop,
crossed-leg, empty) chained with straight punches and palm strikes, two
direction changes, 3.06 m forward travel. Generated with NVIDIA Kimodo
(G1-skeleton variant) via Ultimate Bots Studio; converted with
`convert_soma_csv_to_motion_lib.py` (fps 50).

## Usage
```python
d = joblib.load('part1_0_13s.pkl')
entry = d['part1_0_13s']   # root_trans_offset, pose_aa, dof, root_rot, fps=50
```

## Credit
Motion Data by Bones Studio
