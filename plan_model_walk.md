# 计划：将 G1 AMP 训练从"多样化运动"简化为"模特步态向前走"

## 背景

当前项目用 30 段动作数据（走/跑/转弯/侧步/后退）训练 G1 学会多种运动。用户只想让 G1 向前走路（已有对应 .pkl 动作数据），需要简化配置。

## 需要修改的文件（共 1 个文件）

### 1. `legged_lab/source/legged_lab/legged_lab/tasks/locomotion/amp/config/g1/g1_amp_env_cfg.py`

这是唯一需要改的代码文件。改动点：

#### a) 替换动作数据路径和权重（~第 115-149 行）

- 将 `motion_data_dir` 指向用户的新动作数据目录（或直接放到现有目录下）
- `motion_data_weights` 只保留用户的 .pkl 文件（去掉 30 段走跑动作）

```python
# 改前：28 段走跑动作
self.motion_data.motion_dataset.motion_data_dir = os.path.join(
    LEGGED_LAB_ROOT_DIR, "data", "MotionData", "g1_29dof", "amp", "walk_and_run"
)
self.motion_data.motion_dataset.motion_data_weights = {
    "B10_-__Walk_turn_left_45_stageii": 1.0,
    ... # 28 entries
}

# 改后：只指向用户的动作数据
self.motion_data.motion_dataset.motion_data_dir = os.path.join(
    LEGGED_LAB_ROOT_DIR, "data", "MotionData", "g1_29dof", "amp", "model_walk"  # 新目录
)
self.motion_data.motion_dataset.motion_data_weights = {
    "model_walk": 1.0,  # 用户的 .pkl 文件名（不含 .pkl 后缀）
}
```

#### b) 简化速度指令范围（~第 201-205 行）

只允许向前走，去掉后退、侧移、大幅转弯：

```python
# 改前
self.commands.base_velocity.ranges.lin_vel_x = (-0.5, 3.0)
self.commands.base_velocity.ranges.lin_vel_y = (-0.5, 0.5)
self.commands.base_velocity.ranges.ang_vel_z = (-1.0, 1.0)

# 改后：固定匀速向前，几乎不给转向指令
self.commands.base_velocity.ranges.lin_vel_x = (0.3, 1.0)   # 只向前
self.commands.base_velocity.ranges.lin_vel_y = (0.0, 0.0)    # 无侧移
self.commands.base_velocity.ranges.ang_vel_z = (0.0, 0.0)    # 无转向
```

## 需要用户做的准备工作

1. **把 .pkl 文件放到正确目录**：
   ```
   legged_lab/source/legged_lab/legged_lab/data/MotionData/g1_29dof/amp/
     └── model_walk/           ← 新建这个文件夹
           └── model_walk.pkl  ← 用户的动作数据
   ```

2. **确认 .pkl 文件格式正确**，必须包含以下字段：
   - `fps`: int（帧率）
   - `root_pos`: (N, 3) numpy 数组
   - `root_rot`: (N, 4) numpy 数组，四元数 **(w, x, y, z)** 格式
   - `dof_pos`: (N, 29) numpy 数组，按 Lab DOF 顺序
   - `key_body_pos`: (N, 6, 3) numpy 数组，6 个关键体点世界坐标
   - `loop_mode`: int（0=不循环, 1=循环）

   如果用户数据来自 GMR 重定向并经过 `dataset_retarget.py` 处理，格式应该已经正确。

3. **确认 loop_mode**：向前走路建议设为 `1`（WRAP，循环），这样动画可以无缝循环。

## 验证方法

1. 先用 `list_envs.py` 确认环境注册正常
2. 跑短训练测试：
   ```bash
   python scripts/rsl_rl/train.py --task LeggedLab-Isaac-AMP-G1-v0 --headless --max_iterations 100
   ```
   观察日志中 MotionDataManager 是否成功加载了新动作数据，以及 duration 是否合理
3. 如果需要调参，主要调整：
   - `style_reward_scale`（amp_cfg.py 中的判别器奖励缩放）
   - `task_style_lerp`（任务奖励 vs 风格奖励的混合比例）
   - 速度指令范围
