# 代码改动记录

---

## 2026-06-06 修改：fine-tune 阶段参数调整（冻结 disc + 提高 style/task 权重）

**原因**: disc_lr=1e-5 的训练结果：600 轮步态自然但不响应指令，2200 轮步态再次退化。降 disc_lr 解决了步态问题但没解决指令跟踪，且退化只是被推迟。决定从 600 轮权重 fine-tune，冻结 discriminator 防止退化，提高 style 代价防止跟踪指令时步态走形，提高 task 权重推动指令跟踪。

**症状**: (1) 600轮时在mujoco里面的动作非常自然，可惜就是不会对手柄方向做出相应；(2) 2200轮时在isaacsim中就已经表现和之前一样，会前倾然后向后抬脚了，在mujoco里面也效果不好，且同样不能对手柄方向做出相应。

### 文件1: `legged_lab/.../amp/config/g1/agents/rsl_rl_ppo_cfg.py`

**修改1: 第 59 行 — 冻结 discriminator**

```python
# 修改前（disc_lr=1e-5，discriminator 仍在训练，最终会追上策略导致退化）:
disc_learning_rate=1.0e-5,

# 修改后（冻结 discriminator，fine-tune 阶段保持 style reward 稳定）:
disc_learning_rate=0.0,
```

**修改2: 第 62 行 — 提高 style_reward_scale + task_style_lerp**

```python
# 修改前:
style_reward_scale=5.0, task_style_lerp=0.3

# 修改后（scale 5→20 让 style 变差的代价变大，防止跟踪指令时步态走形；lerp 0.3→0.5 提高指令跟踪权重）:
style_reward_scale=20.0, task_style_lerp=0.5
```

### 文件2: `legged_lab/scripts/rsl_rl/train.py`

**修改3: 第 251-254 行 — resume 后强制覆盖 disc lr**

```python
# 修改前（load 会从 checkpoint 恢复 disc lr=1e-5，覆盖 config 里的 0）:
runner.load(resume_path)

# 修改后（load 之后强制把 disc lr 重置为 config 的值，确保冻结生效）:
runner.load(resume_path)
if hasattr(runner.alg, "disc_optimizer"):
    for pg in runner.alg.disc_optimizer.param_groups:
        pg["lr"] = runner.alg.amp_cfg["disc_learning_rate"]
```

**详细分析**: 见 `history/training_analysis.md` → 2026-06-05_20-30-37 章节
```

**详细分析**: 见 `history/training_analysis.md` → 2026-06-05_20-30-37 章节

---

## 2026-06-05 修改：降低 discriminator 学习率（修复步态退化 + 指令不响应）

**原因**: `disc_learning_rate=1e-4` 与策略学习率相同，但 discriminator 的任务（二分类）远比策略（高维连续控制）简单。相同学习率下 discriminator 进步远快于策略，导致 style reward 被压到接近零（disc_score ≈ -0.865，gap ≈ 1.73），AMP 失去约束步态的能力。

**症状**: (1) 机器人在 MuJoCo 中不响应手柄指令，只会固定速度走；(2) 训练后期步态退化（身体前倾、脚后跟抬太高），早期（~1000步）步态反而更自然。

**文件**: `legged_lab/source/legged_lab/legged_lab/tasks/locomotion/amp/config/g1/agents/rsl_rl_ppo_cfg.py` 第 59 行

```python
# 修复前（discriminator 学习率与策略相同，进步太快）:
disc_learning_rate=1.0e-4,

# 修复后（恢复 amp_cfg.py 默认值，给策略追赶的机会）:
disc_learning_rate=1.0e-5,
```

**详细分析**: 见 `history/training_analysis.md` → 2026-06-05 章节

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
