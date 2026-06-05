# 代码改动记录

---

## 2026-05-25 修改：修复 N5020-16 stiffness dict 缺失关节

**原因**: stiffness dict 只写了 waist 两个关节，IsaacLab 对未匹配的关节赋 stiffness=0，导致 shoulder/elbow/wrist_roll/ankle 共 14 个关节完全无力，机器人无法站立。

**文件**: `legged_lab/source/legged_lab/legged_lab/assets/unitree.py` 第 148-155 行

```python
# 修复前（错误，14个关节 K=0）:
stiffness={
    "waist_roll_joint": 200.0,
    "waist_pitch_joint": 200.0,
},

# 修复后（和 env.yaml 一致，所有关节显式列出）:
stiffness={
    ".*_shoulder_.*": 40.0,
    ".*_elbow_.*": 40.0,
    ".*_wrist_roll.*": 40.0,
    ".*_ankle_.*": 40.0,
    "waist_roll_joint": 200.0,
    "waist_pitch_joint": 200.0,
},
```

---

## 2026-05-25 恢复策略A完整配置（基于 env.yaml）

从 `/home/reverie/AMP_final_project/2026-05-24_12-16-18/params/env.yaml` 中提取策略A的真实配置，恢复以下改动：

### 文件1: `unitree.py`

**1a. N5020-16 stiffness: uniform 40 → dict（waist=200, 其余=40）**
- 第 148 行：`stiffness=40.0` 改为包含所有关节的 dict
- `waist_roll_joint` 和 `waist_pitch_joint` 从 40 改为 200

**1b. N5020-16 damping: arm 1→10**
- 第 153-155 行：`shoulder/elbow/wrist_roll` damping 从 1.0 改为 10.0

**1c. W4010-25 damping: 1→10**
- 第 167 行：damping 从 1.0 改为 10.0

### 文件2: `g1_amp_env_cfg.py`

**2a. 添加 velocity_mdp import**
- 第 9 行：添加 `import legged_lab.tasks.locomotion.velocity.mdp as velocity_mdp`

**2b. reward 函数恢复为 yaw_frame/world_exp**
- 第 35-47 行：
  - `mdp.track_lin_vel_xy_exp` → `velocity_mdp.track_lin_vel_xy_yaw_frame_exp`
  - `mdp.track_ang_vel_z_exp` → `velocity_mdp.track_ang_vel_z_world_exp`

### 文件3: `amp_env_cfg.py`（无变化，保持 heading 禁用）

---

## 2026-05-25 回退 reward 函数到 commit 原版

**文件**: `g1_amp_env_cfg.py`
- 移除 `import velocity_mdp`
- `velocity_mdp.track_lin_vel_xy_yaw_frame_exp` → `mdp.track_lin_vel_xy_exp`（base frame）
- `velocity_mdp.track_ang_vel_z_world_exp` → `mdp.track_ang_vel_z_exp`（base frame）

**注**: 此改动在找到 env.yaml 后又被恢复（见上方 2026-05-25 恢复策略A配置）

---

## 2026-05-25 unitree.py 多次 PD gains 调整

按时间顺序：

1. arm damping 1→10 + W4010 damping 1→10（匹配 deploy.yaml）
2. waist pitch/roll stiffness 40→200（后来改回 40）
3. 全部改回 commit 原版（arm=1, W4010=1, waist=40），对齐 unitree_rl_lab
4. 又改回 arm=10, W4010=10, waist=40（用户找到策略A的 unitree.py）
5. 最终从 env.yaml 确认策略A实际是 arm=10, W4010=10, waist=200，恢复到此配置

---

## 2026-05-25 task_style_lerp 回退

**文件**: `rsl_rl_ppo_cfg.py`
- `task_style_lerp` 从 0.5 改回 0.3（和 git commit 一致）

---

## 2026-05-24 观测格式 480-dim → 495-dim → 480-dim（用户手动操作）

**文件**: `amp_env_cfg.py`
- PolicyCfg 中 `projected_gravity`(3D) 换成 `root_local_rot_tan_norm`(6D)，`joint_pos_rel` 换成 `joint_pos`，`joint_vel_rel` 换成 `joint_vel`
- 产生 495-dim 策略 B，在 MuJoCo 中站不住
- 用户手动改回 480-dim

---

## 2026-05-24 heading 禁用

**文件**: `amp_env_cfg.py`, `g1_amp_env_cfg.py`
- `heading_command=True` 注释掉（默认 False）
- `rel_heading_envs` 1.0→0.0
- heading range 移除
- **保留至今，不可回退**（deploy 发 ang_vel_z，不是 heading）
