# 策略A 完整配置存档

> 来源: `/home/reverie/AMP_final_project/2026-05-24_12-16-18/params/env.yaml`
> 这是策略A（"比较不错的策略"）训练时的确切配置，由 IsaacLab 在训练开始时自动导出
> 创建时间: 2026-05-24 12:17

## 策略A = git commit `1e835cb` + 以下改动

### 1. `unitree.py` — 3 处改动

**1a. N5020-16 stiffness: uniform 40 → dict（waist 200）**

```python
# commit 版本:
stiffness=40.0,

# 策略A（env.yaml 第218-224行）:
stiffness={
    "waist_roll_joint": 200.0,
    "waist_pitch_joint": 200.0,
},
# 其余关节保持 40.0（dict 未匹配的用默认值）
```

| 关节 | commit | 策略A | deploy.yaml |
|------|--------|-------|------------|
| shoulder/elbow/wrist_roll/ankle | K=40 | K=40 | K=40 |
| waist_pitch | K=40 | **K=200** | K=200 |
| waist_roll | K=40 | **K=200** | K=200 |

**1b. N5020-16 damping: arm 1→10**

```python
# commit 版本:
damping={
    ".*_shoulder_.*": 1.0,
    ".*_elbow_.*": 1.0,
    ".*_wrist_roll.*": 1.0,
    ...
}

# 策略A（env.yaml 第226-230行）:
damping={
    ".*_shoulder_.*": 10.0,
    ".*_elbow_.*": 10.0,
    ".*_wrist_roll.*": 10.0,
    ".*_ankle_.*": 2.0,
    "waist_.*_joint": 5.0,
}
```

**1c. W4010-25 damping: 1→10**

```python
# commit 版本:
damping=1.0,

# 策略A（env.yaml 第245行）:
damping=10.0,
```

### 2. `g1_amp_env_cfg.py` — reward 函数改动

```python
# commit 版本:
func=mdp.track_lin_vel_xy_exp        # base frame
func=mdp.track_ang_vel_z_exp          # base frame

# 策略A（env.yaml 第833-843行）:
import legged_lab.tasks.locomotion.velocity.mdp as velocity_mdp
func=velocity_mdp.track_lin_vel_xy_yaw_frame_exp   # yaw frame
func=velocity_mdp.track_ang_vel_z_world_exp         # world frame
```

需要添加 `import velocity_mdp`

### 3. `amp_env_cfg.py` — heading 禁用（与当前代码一致）

```python
rel_heading_envs=0.0,           # 1.0→0.0
# heading_command=True,          # 注释掉
# heading_control_stiffness=0.5, # 注释掉
# heading range 移除
```

### 4. `amp_env_cfg.py` — 观测格式（已在commit中改好，无需额外改动）

480-dim: base_ang_vel(3) + projected_gravity(3) + velocity_commands(3) + joint_pos_rel(29) + joint_vel_rel(29,scale=0.05) + last_action(29)

### 5. 不需要改动的文件

| 文件 | 说明 |
|------|------|
| `rsl_rl_ppo_cfg.py` | task_style_lerp=0.3, style_reward_scale=5.0，和策略A一致 |
| `symmetry/g1.py` | 96-dim per step，和480-dim匹配 |
| 运动数据 | model_walk 8个文件，已在commit中配好 |
| 速度范围 | vx=[-0.5,1.0] vy=[-0.5,0.5] wz=[-1.0,1.0]，已在commit中配好 |

## 策略A 完整 PD Gains 对照

| 关节 | Train_K | Deploy_K | Train_D | Deploy_D |
|------|---------|----------|---------|----------|
| hip_pitch/yaw | 100 | 100 | 2 | 2 |
| hip_roll | 100 | 100 | 2 | 2 |
| knee | 150 | 150 | 4 | 4 |
| ankle | 40 | 40 | 2 | 2 |
| waist_yaw | 200 | 200 | 5 | 5 |
| **waist_pitch** | **200** | **200** | 5 | 5 |
| **waist_roll** | **200** | **200** | 5 | 5 |
| shoulder/elbow/wrist_roll | 40 | 40 | **10** | **10** |
| wrist_pitch/yaw | 40 | 40 | **10** | **10** |

**策略A的 PD Gains 和 deploy.yaml 完全一致。**

## 训练效果

- MuJoCo 中能站住、能走路
- 手柄无法控制方向，机器人会绕大圈走
- 原因: task_style_lerp=0.3 → style reward 占 70%，policy 主要模仿动作数据而非响应指令
