# Training Configuration

## Motion Data

- **Source**: Generated with NVIDIA Kimodo (G1-skeleton variant) via
  Ultimate Bots Studio
- **Motion**: ~13.0-second (649 samples @ 50 Hz) Wubuquan-inspired
  martial-arts sequence (5 stance types, straight punches/palm strikes,
  two direction changes, ~3.06 m net root displacement)
- **Conversion** (from this repository's raw CSVs to SONIC motion_lib):
  ```bash
  export SONIC_ROOT=/path/to/GR00T-WholeBodyControl
  export PROJECT_ROOT=/path/to/G1-Wubuquan-Five-Step-Fist
  cd "$SONIC_ROOT"
  python gear_sonic/data_process/convert_soma_csv_to_motion_lib.py \
    --input "$PROJECT_ROOT/data/raw_part1_0_13s" \
    --output "$PROJECT_ROOT/data/part1_0_13s.pkl" \
    --fps 50
  ```
- **Quality checks**: No NaN/Inf; joint angle range within limits;
  replay-verified (no teleporting, no floor penetration).
- **Known artifact**: a retarget velocity transient at frame 322 of the
  raw `joint_vel.csv` (44.9 rad/s, multi-joint co-occurring) — see
  `config/dataset_README.md`.

## Training Commands (reproducible)

Run from the SONIC/GEAR-SONIC checkout. The base checkpoint is **not
redistributed in this repository** — obtain `sonic_release/last.pt` from
the official SONIC release (NVIDIA
[GR00T-WholeBodyControl](https://github.com/NVlabs/GR00T-WholeBodyControl))
and place it at `$SONIC_ROOT/sonic_release/last.pt`, or point
`+checkpoint=` at your copy.

### Stage V1 — full fine-tune on the target motion
```bash
cd "$SONIC_ROOT"
python gear_sonic/train_agent_trl.py \
  +exp=manager/universal_token/all_modes/sonic_release \
  +checkpoint=sonic_release/last.pt \
  num_envs=2048 headless=True \
  ++algo.config.num_learning_iterations=4000 \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file="$PROJECT_ROOT/data/part1_0_13s.pkl" \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### Stage V1.1 — root-focused refinement (final model)
```bash
cd "$SONIC_ROOT"
python gear_sonic/train_agent_trl.py \
  +exp=manager/universal_token/all_modes/sonic_release \
  +checkpoint=<V1 checkpoint> \
  num_envs=2048 headless=True \
  ++algo.config.num_learning_iterations=1500 \
  ++algo.config.actor_learning_rate=1e-5 \
  ++manager_env.rewards.tracking_anchor_pos.weight=1.0 \
  ++manager_env.rewards.tracking_anchor_pos.params.std=0.2 \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file="$PROJECT_ROOT/data/part1_0_13s.pkl" \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### Evaluation (official pipeline — early termination)
```bash
cd "$SONIC_ROOT"
python gear_sonic/eval_agent_trl.py \
  +checkpoint=<checkpoint> +headless=True ++num_envs=32 \
  ++eval_callbacks=im_eval ++run_eval_loop=False \
  '+manager_env/terminations=tracking/eval' \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file="$PROJECT_ROOT/data/part1_0_13s.pkl" \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### Evaluation (fixed-horizon — no tracking termination, fair comparison)
Create `gear_sonic/config/manager_env/terminations/fixed_horizon.yaml`
with:
```yaml
defaults:
  - terms/motion_time_out@_here_
_target_: gear_sonic.envs.manager_env.mdp.terminations.TerminationsCfg
```
then run the same eval command with
`'+manager_env/terminations=fixed_horizon'` instead of `tracking/eval`.

### ONNX export
```bash
python gear_sonic/eval_agent_trl.py \
  +checkpoint=<checkpoint> +headless=True ++num_envs=1 \
  +export_onnx_only=true ++use_wandb=false
```

## Compute

Single Nvidia L40S (Nebius/Modal), ~2048 parallel envs. V1 ≈ 3.5 h,
V1.1 ≈ 1.5 h.
