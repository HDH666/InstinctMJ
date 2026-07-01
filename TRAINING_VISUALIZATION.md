# 训练曲线与训练画面可视化说明

本文档总结 `InstinctMJ` 当前可用的训练曲线、训练画面、回放画面、W&B 与视频录制方式。

## 训练曲线

当前训练曲线默认写入 TensorBoard event 文件。

训练日志目录结构为：

```bash
logs/instinct_rl/<experiment_name>/<timestamp_run>/
```

例如当前仓库已有：

```bash
logs/instinct_rl/g1_locomotion_flat/2026-06-29_18-14-45/events.out.tfevents...
```

查看曲线：

```bash
uv run tensorboard --logdir logs/instinct_rl
```

TensorBoard 中可以查看 reward、loss、episode metrics、performance metrics 等训练指标。

## W&B

当前仓库依赖中包含 `wandb`，但训练指标并没有默认上报到 W&B。

原因是当前使用的 `instinct_rl.runners.OnPolicyRunner` 内部硬编码使用 `tensorboardX.SummaryWriter` 写 TensorBoard 日志，没有 `wandb.init()` 或 `logger=wandb` 分支。

仓库中 W&B 主要用于 artifact 下载，例如通过：

```bash
--registry-name <wandb_artifact_name>
```

拉取 motion 文件，而不是作为训练曲线 logger。

如果想用 W&B 看当前训练曲线，可以先保留 TensorBoard 日志，再用 W&B 同步：

```bash
wandb sync logs/instinct_rl/<experiment_name>/<run>
```

如果要在训练时直接上报到 W&B，需要修改 `InstinctMJ` 或上游 `instinct_rl` runner 的日志逻辑。

## 训练画面

训练脚本支持 MuJoCo native viewer。

可选 viewer：

```text
none
native
```

训练时打开 native viewer：

```bash
uv run instinct-train Instinct-Locomotion-Flat-G1-v0 --viewer native
```

注意：`native` viewer 需要可用的 `DISPLAY` 或 `WAYLAND_DISPLAY`。如果在无图形界面的服务器上运行，训练脚本会默认使用 headless EGL 路径，不适合打开 native viewer。

## 回放画面

回放脚本支持更多 viewer 后端。

可选 viewer：

```text
auto
native
viser
none
```

默认 `auto` 逻辑：

- 有 `DISPLAY` 或 `WAYLAND_DISPLAY` 时使用 `native`
- 无图形显示环境时使用 `viser`

回放示例：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 --load-run <run_name>
```

显式指定 native：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 --load-run <run_name> --viewer native
```

显式指定 viser：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 --load-run <run_name> --viewer viser
```

结论：`viser` 目前只在 `instinct-play` 回放入口中支持，训练入口只支持 `none` 和 `native`。

## 视频录制

视频录制通过 `mjlab.utils.wrappers.VideoRecorder` 实现。

### 训练视频

训练视频默认参数：

```python
video_interval = 2000
video_length = 200
```

含义：

- 每 `2000` 个环境 step 触发一次录制
- 每段视频录 `200` 个环境 step

训练录视频：

```bash
uv run instinct-train Instinct-Locomotion-Flat-G1-v0 --video
```

修改录制间隔和长度：

```bash
uv run instinct-train Instinct-Locomotion-Flat-G1-v0 \
  --video \
  --video-interval 5000 \
  --video-length 400
```

默认输出位置：

```bash
logs/instinct_rl/<experiment_name>/<run>/videos/train/
```

### 回放视频

回放视频默认参数：

```python
video_length = 400
```

回放视频只在 step 0 触发一次，也就是默认录一段 `400` 个环境 step 的视频。

回放录视频：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 \
  --load-run <run_name> \
  --video
```

修改回放视频长度：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 \
  --load-run <run_name> \
  --video \
  --video-length 800
```

指定输出目录：

```bash
uv run instinct-play Instinct-Locomotion-Flat-G1-Play-v0 \
  --load-run <run_name> \
  --video \
  --video-dir <output_dir>
```

默认输出位置：

```bash
<checkpoint_dir>/videos/play/
```

如果没有 checkpoint，则输出到：

```bash
logs/instinct_rl/<experiment_name>/<task_id>/videos/play/
```

## 视频时长换算

视频参数中的 `video_interval` 和 `video_length` 都是环境 step 数，不是秒数。

多数任务当前配置为：

```python
decimation = 4
mujoco.timestep = 0.005
```

因此：

```text
step_dt = decimation * mujoco.timestep = 4 * 0.005 = 0.02 秒
```

也就是环境频率约为 `50 Hz`。

按这个默认配置换算：

- 训练默认每 `2000` step 录一次，约每 `40` 秒录一次
- 训练默认每段 `200` step，约 `4` 秒
- 回放默认每段 `400` step，约 `8` 秒

如果某个任务修改了 `decimation` 或 `mujoco.timestep`，实际秒数需要按该任务的 `step_dt` 重新换算。
