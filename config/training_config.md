# Training Configuration

## Motion Data

- **Source**: Generated with NVIDIA Kimodo (G1-skeleton variant) via Ultimate
  Bots Studio
- **Motion**: 12.98-second Five-Step Chain Fist routine (5 stance types,
  straight punches/palm strikes, two direction changes, 3.06 m travel)
- **Duration**: 649 frames @ 50 fps, G1 29-DOF joint trajectories
- **Conversion**:
  ```bash
  python gear_sonic/data_process/convert_soma_csv_to_motion_lib.py \
    --input data/part1_0_13s --output data/part1_0_13s.pkl --fps 50
  ```
- **Quality checks**: No NaN/Inf; joint angle range within limits;
  replay-verified (no teleporting, no floor penetration).

## Training Commands (reproducible)

### Stage V1 — full fine-tune on the target motion
```bash
python gear_sonic/train_agent_trl.py \
  +exp=manager/universal_token/all_modes/sonic_release \
  +checkpoint=sonic_release/last.pt \
  num_envs=2048 headless=True \
  ++algo.config.num_learning_iterations=4000 \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file=data/part1_0_13s.pkl \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### Stage V1.1 — root-focused refinement (final model)
```bash
python gear_sonic/train_agent_trl.py \
  +exp=manager/universal_token/all_modes/sonic_release \
  +checkpoint=<V1 checkpoint> \
  num_envs=2048 headless=True \
  ++algo.config.num_learning_iterations=1500 \
  ++algo.config.actor_learning_rate=1e-5 \
  ++manager_env.rewards.tracking_anchor_pos.weight=1.0 \
  ++manager_env.rewards.tracking_anchor_pos.params.std=0.2 \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file=data/part1_0_13s.pkl \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### Evaluation (official pipeline)
```bash
python gear_sonic/eval_agent_trl.py \
  +checkpoint=<checkpoint> +headless=True ++num_envs=32 \
  ++eval_callbacks=im_eval ++run_eval_loop=False \
  '+manager_env/terminations=tracking/eval' \
  ++manager_env.commands.motion.motion_lib_cfg.motion_file=data/part1_0_13s.pkl \
  ++manager_env.commands.motion.motion_lib_cfg.smpl_motion_file=zeros \
  ++use_wandb=false
```

### ONNX export
```bash
python gear_sonic/eval_agent_trl.py \
  +checkpoint=<checkpoint> +headless=True ++num_envs=1 \
  +export_onnx_only=true ++use_wandb=false
```

## Compute

Single Nvidia L40S (Nebius/Modal), ~2048 parallel envs. V1 ≈ 3.5 h,
V1.1 ≈ 1.5 h.
