# 当前本地代码 vs Git 最新提交的差异记录

> Git 最新提交: `1e835cb 对齐 AMP 观测空间与 RL 部署配置，修复对称性代码维度`
> 最后更新: 2025-05-25

## 当前 local diff 涉及 2 个文件

---

### 1. `amp_env_cfg.py` — heading command 禁用

**文件路径**: `legged_lab/source/legged_lab/legged_lab/tasks/locomotion/amp/amp_env_cfg.py`

```python
# commit 版本:
rel_heading_envs=1.0,
heading_command=True,
heading_control_stiffness=0.5,
ranges=mdp.UniformVelocityCommandCfg.Ranges(
    lin_vel_x=(-0.1, 0.1), lin_vel_y=(-0.1, 0.1), ang_vel_z=(-0.1, 0.1), heading=(-math.pi, math.pi)
)

# 当前版本:
rel_heading_envs=0.0,  # [MODIFIED]
# heading_command=True,  # [MODIFIED] disabled
# heading_control_stiffness=0.5,
ranges=mdp.UniformVelocityCommandCfg.Ranges(
    lin_vel_x=(-0.1, 0.1), lin_vel_y=(-0.1, 0.1), ang_vel_z=(-0.1, 0.1)  # [MODIFIED] removed heading
)
```

**改动原因**: deploy 发的是 ang_vel_z（手柄摇杆），不是 heading angle。禁用后 velocity_commands = [vx, vy, wz]（3D），匹配 deploy.yaml。

**状态**: 必须保留

---

### 2. `g1_amp_env_cfg.py` — heading 覆盖禁用

**文件路径**: `legged_lab/source/legged_lab/legged_lab/tasks/locomotion/amp/config/g1/g1_amp_env_cfg.py`

```python
# commit 版本:
self.commands.base_velocity.ranges.heading = (-math.pi, math.pi)  # G1AmpEnvCfg
self.commands.base_velocity.ranges.heading = (0.0, 0.0)           # G1AmpEnvCfg_PLAY

# 当前版本:
# self.commands.base_velocity.ranges.heading = (-math.pi, math.pi)  # [MODIFIED] disabled
# self.commands.base_velocity.ranges.heading = (0.0, 0.0)           # [MODIFIED] disabled
```

**状态**: 保留

---

## 已回退的改动（不再出现在 diff 中）

### ~~unitree.py PD gains 全部改回 commit 原版~~（已回退）

曾改过：
- waist_pitch/roll stiffness: 40→200（后改回 40）
- shoulder/elbow/wrist_roll damping: 1→10（后改回 1）
- W4010-25 damping: 1→10（后改回 1）

**回退原因**: TA 的 unitree_rl_lab 训练代码也是 K=40/D=1，和 commit 一致。deploy.yaml 虽然不同（waist K=200, arm D=10），但 TA 的 RL 策略在 K=40/D=1 训练后能在 deploy 上正常工作，说明 sim-to-real 的 PD 差异是可以容忍的。

**回退时间**: 2025-05-25

---

### 策略A 的确切配置（已确认）

2025-05-25 用户找到了策略A训练时的确切 unitree.py 配置：

**策略A = git commit + 以下改动：**
- `unitree.py` N5020-16: stiffness=40, **damping: shoulder=10, elbow=10, wrist_roll=10**, ankle=2, waist=5
- `unitree.py` W4010-25: stiffness=40, **damping=10**
- `amp_env_cfg.py`: heading 禁用
- `g1_amp_env_cfg.py`: heading 覆盖禁用（reward 函数不确定）
- `rsl_rl_ppo_cfg.py`: task_style_lerp=0.3

**关键发现**: 策略A的 arm damping 是 10（和 deploy.yaml 一致），不是 1。waist stiffness 是 40（和 commit 一致）。

**当前磁盘代码 vs 策略A:**
- 当前代码: arm damping=1, W4010 damping=1（和 unitree_rl_lab/TA 一致）
- 策略A: arm damping=10, W4010 damping=10（和 deploy.yaml 一致）
- 差异: arm/wrist damping 1 vs 10

---

### 最新训练结果（2025-05-25）

用当前代码（arm damping=1, 和 unitree_rl_lab 一致）训练，效果和策略A差不多（走路效果相近）。

说明 arm damping 1 vs 10 对 AMP 训练效果影响不大。两个版本都能产出可用的策略。

当前 local diff 只剩 heading 禁用（2 个文件），其余全部和 git commit 一致。

---

### ~~reward 函数: base frame → yaw/world frame~~（已回退到 commit 原版）

```python
# 曾改成的版本（已回退）:
func=velocity_mdp.track_lin_vel_xy_yaw_frame_exp  # yaw frame 追踪线速度
func=velocity_mdp.track_ang_vel_z_world_exp        # world frame 追踪角速度

# 当前版本（= commit 原版）:
func=mdp.track_lin_vel_xy_exp   # base frame 追踪线速度 (root_lin_vel_b)
func=mdp.track_ang_vel_z_exp    # base frame 追踪角速度 (root_ang_vel_b[:, 2])
```

**回退时间**: 2025-05-25

---

### ~~task_style_lerp 0.3 → 0.5~~（已回退）

**回退时间**: 2025-05-24

---

### ~~观测格式 480-dim → 495-dim~~（已回退）

**回退时间**: 2025-05-24（由用户手动回退）

---

## 训练配置 vs deploy.yaml 完整对比

### 完全匹配的项

| 配置项 | 训练值 | deploy值 |
|--------|-------|---------|
| 策略观测维度 | 480 | 480 |
| step_dt | 0.02 | 0.02 |
| action scale | 0.25 | 0.25 |
| action offset | default_joint_pos | default_joint_pos |
| base_ang_vel scale | 0.2 | [0.2,0.2,0.2] |
| projected_gravity scale | 1.0 | [1.0,1.0,1.0] |
| velocity_commands scale | 1.0 | [1.0,1.0,1.0] |
| joint_pos_rel scale | 1.0 | [1.0×29] |
| joint_vel_rel scale | 0.05 | [0.05×29] |
| last_action scale | 1.0 | [1.0×29] |
| history_length | 5 | 5 |
| velocity_commands dim | 3D [vx,vy,wz] | 3D [vx,vy,wz] |

### PD Gains 对比（所有 29 个关节）

当前训练配置和 unitree_rl_lab（TA 的 RL 训练代码）完全一致，和 deploy.yaml 的差异：
- waist_pitch/roll: 训练 K=40, deploy K=200（TA 的 RL 也是这么做的，能正常工作）
- shoulder/elbow/wrist: 训练 D=1, deploy D=10（TA 的 RL 也是这么做的，能正常工作）

| 关节 | Train_K | Deploy_K | Train_D | Deploy_D |
|------|---------|----------|---------|----------|
| left_hip_pitch | 100 | 100 | 2 | 2 |
| left_hip_roll | 100 | 100 | 2 | 2 |
| left_hip_yaw | 100 | 100 | 2 | 2 |
| left_knee | 150 | 150 | 4 | 4 |
| left_ankle_pitch | 40 | 40 | 2 | 2 |
| left_ankle_roll | 40 | 40 | 2 | 2 |
| right_hip_pitch | 100 | 100 | 2 | 2 |
| right_hip_roll | 100 | 100 | 2 | 2 |
| right_hip_yaw | 100 | 100 | 2 | 2 |
| right_knee | 150 | 150 | 4 | 4 |
| right_ankle_pitch | 40 | 40 | 2 | 2 |
| right_ankle_roll | 40 | 40 | 2 | 2 |
| waist_yaw | 200 | 200 | 5 | 5 |
| waist_pitch | 40 | 200 | 5 | 5 |
| waist_roll | 40 | 200 | 5 | 5 |
| shoulder/elbow/wrist | 40 | 40 | 1 | 10 |
| wrist_pitch/yaw (W4010) | 40 | 40 | 1 | 10 |

### 仍有差异的项（不影响站立）

| 配置项 | 训练值 | deploy值 | 影响 |
|--------|-------|---------|------|
| lin_vel_y range | [-0.5, 0.5] | [-0.3, 0.3] | 训练范围更宽，不影响站立 |
| ang_vel_z range | [-1.0, 1.0] | [-0.2, 0.2] | 训练范围更宽，不影响站立 |

---

## 关键配置速查

| 配置项 | 当前值 |
|--------|-------|
| 策略观测维度 | 480 = 96 × 5 |
| velocity_commands | 3D [vx, vy, wz] |
| action scale | 0.25, use_default_offset=True |
| task_style_lerp | 0.3 |
| reward 函数 | mdp.track_lin_vel_xy_exp + mdp.track_ang_vel_z_exp (base frame) |
| 运动数据 | model_walk/ 8 个文件 |
| 速度范围 | vx=[-0.5,1.0] vy=[-0.5,0.5] wz=[-1.0,1.0] |
| waist stiffness | **40 (和 unitree_rl_lab/TA 一致)** |
