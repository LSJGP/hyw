# HYW 工作区

本仓库是 **元仓库（meta repo）**：通过 Git Submodule 把四个独立项目固定在同一套目录布局下，便于本地开发与版本对齐。

```
hyw/                          ← 本仓库（记录 submodule 指针）
├── hyw-proto/                ← 子模块：.proto 唯一来源
├── hyw-grading/              ← 子模块：C++ 评分 grading_main
├── hyw-sim/                  ← 子模块：C++ 闭环仿真 sim_runner
└── hyw-workbench/            ← 子模块：场景、工具、Web 控制台、输出目录
```

| 子模块 | 仓库 | 职责 |
|--------|------|------|
| `hyw-proto` | [hyw-proto](https://github.com/LSJGP/hyw-proto) | `hyw_sim.proto` / `grading_mini.proto` 定义 |
| `hyw-grading` | [hyw-grading](https://github.com/LSJGP/hyw-grading) | 读取每帧 `MetricFrameInput`，输出评分报告 |
| `hyw-sim` | [hyw-sim](https://github.com/LSJGP/hyw-sim) | 读场景 JSON → 规划 / 动力学 / 碰撞 → 可选推流给 grading |
| `hyw-workbench` | [hyw-workbench](https://github.com/LSJGP/hyw-workbench) | Waymo 转换、批跑、Web Dashboard、`scenarios/`、`output/` |

---

## 克隆与构建

```bash
git clone --recurse-submodules https://github.com/LSJGP/hyw.git
cd hyw

# 评分（只需首次或 grading 有改动）
cd hyw-grading && bazel build //src/entry:grading_main && cd ..

# 仿真（依赖同级的 hyw-proto）
cd hyw-sim && bazel build //cpp:sim_runner && cd ..
```

可选环境变量见 `hyw-workbench/hyw.env.example`（`HYW_ROOT` 默认指向本目录的父路径即 `hyw` 本身）。

---

## 数据链路（总览）

整条链路可以分成三段：**场景制备** → **闭环仿真** → **评分与可视化**。磁盘上场景是三个 JSON；运行时 sim 会转成 `hyw-proto` 里的 proto 结构。

### 竖向总览

```
  [Waymo Motion tfrecord]
           │
           ▼
  hyw-workbench/tools/waymo_to_scenario.py
  （conda + waymo-open-dataset）
           │
           ▼
  scenarios/<name>/          ← 仿真入口数据
    ├── scenario_meta.json   （init/goal、world_offset、统计）
    ├── dynamic_objects.json （NPC + SDC 逐帧轨迹）
    └── lane_graph.json      （静态地图，已减 world_offset）
           │
           ▼
  hyw-sim/run_sim.py → bazel run //cpp:sim_runner
    · 读 JSON（JsonStringToMessage / proto_io）
    · World：planner → 车辆模型 → OBB 碰撞
    · 每帧 FrameRecord
           │
           ├──────────────────────────┐
           ▼                          ▼
  output/log/<name>_sim_log.json   grading_main（可选）
  （离线 SimLog / both 模式）         stdin 流式 MetricFrameInput（online）
           │                          │
           │                          ▼
           │              output/report/<name>_grading_report.json
           ▼
  hyw-workbench/tools/viz_sim.py（可选）
           ▼
  output/viz/<name>_sim.gif
```

### 1. 场景制备（Waymo → JSON）

```
Waymo Scenario protobuf（tfrecord 单条）
        │
        │  waymo_to_scenario.py
        ▼
┌───────────────────────┐
│ scenario_meta.json    │  source / scenario_id / world_offset
│                       │  init_pose、goal_pose、ScenarioStats
└───────────────────────┘
        │
        ├──────────────────────────────────────┐
        ▼                                      ▼
┌───────────────────────┐            ┌───────────────────────┐
│ dynamic_objects.json  │            │ lane_graph.json       │
│ tracks[] + 逐帧 state │            │ lanes / road_lines /  │
│ SDC 锚到局部 (0,0,0)  │            │ edges / crosswalks …  │
└───────────────────────┘            └───────────────────────┘
```

- 默认以 **SDC 首帧有效位姿** 为原点，地图与 NPC 统一平移（`world_offset`）。
- 与 `hyw-proto/proto/sim/*.proto` 字段语义一致；JSON 是便于工具链读写的落盘格式。

### 2. 仿真主循环（C++ sim_runner）

```
scenario_meta + dynamic_objects + lane_graph
        │
        ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ LaneGraph   │────▶│ MapQuery     │────▶│ Planner     │
│ 路由/参考线 │     │ 最近车道等   │     │ local_dwa / │
└─────────────┘     └──────────────┘     │ goal_seek … │
        │                    │           └──────┬──────┘
        │                    │                  │
        ▼                    ▼                  ▼
┌─────────────────────────────────────────────────────┐
│ World：NPC 插值、自车积分、OBB/SAT 碰撞、帧记录      │
└─────────────────────────────────────────────────────┘
        │
        ▼
  FrameRecord（hyw_sim.proto / runtime）
        │
        ├──▶ grading_convert → JSON 行 → grading_main --stream
        └──▶ sim_log.json（--output，供离线批处理或 viz）
```

### 3. 评分与可视化

```
每帧 FrameRecord + StaticMap + VehicleParams
        │
        ▼
  MetricFrameInput（grading proto，JSON 行）
        │
        ▼
  grading_main  ← metrics JSON（前端勾选 metric 名生成）
        │
        ▼
  grading_report.json（overallPassed、summaries）

sim_log.json + 场景 JSON
        │
        ▼
  viz_sim.py --animate
        │
        ▼
  output/viz/<scenario>_sim.gif
```

### Proto 在链路中的位置

```
hyw-proto/proto/sim/*.proto
        │
        ├──▶ hyw-sim（Bazel local_repository ../hyw-proto）
        └──▶ hyw-grading（同上）

JSON 场景文件 ──加载──▶ C++ Message ──仿真──▶ FrameRecord / SimLog
```

修改 `.proto` 后需在 **hyw-proto 提交**，并在 **hyw-sim / hyw-grading** 重新 `bazel build`；元仓库 `hyw` 再 bump 对应 submodule 指针。

---

## 用 Web 前端跑仿真（推荐）

### 启动

```bash
cd hyw-workbench
./start_dashboard.sh
# 默认 http://127.0.0.1:8765/
# 换端口: ./start_dashboard.sh --port 9000
# 局域网访问: HOST=0.0.0.0 ./start_dashboard.sh
```

页面标题：**Hywgrading 批量仿真控制台**。左侧选参数，右侧看实时日志与结果表（sim / grading / gif）。

启动前请确认页底提示：

- `grading_main ✓`：已 `bazel build //src/entry:grading_main`
- `sim_runner` 提示：首次运行会由 `run_sim.py` 触发 `bazel run`（也可预先 `bazel build //cpp:sim_runner`）

### 操作流程

```
打开浏览器 → 勾选场景 → 调 Planner / Metrics / 日志 / GIF
        │
        ▼
  「开始批量运行」
        │
        ▼
  POST /api/jobs → batch_run_scenarios.run_batch
        │
        ├── 每个场景: run_sim.py（hyw-sim）
        └── 可选: viz_sim.py → output/viz/*.gif
        │
        ▼
  结果表：sim OK/FAIL · grading PASS/FAIL · gif OK/FAIL
```

输出目录（默认在 `hyw-workbench/output/`）：

| 路径 | 内容 |
|------|------|
| `log/<scenario>_sim_log.json` | 仿真 SimLog |
| `report/<scenario>_grading_report.json` | 评分报告 |
| `viz/<scenario>_sim.gif` | 动画（勾选生成 GIF 时） |
| `batch/metrics_<job_id>.json` | 本次任务临时 metrics 配置 |

---

## 前端参数说明

### 场景 `scenarios/`

| 控件 | 说明 |
|------|------|
| 场景列表（多选） | 仅列出同时含有 `scenario_meta.json`、`dynamic_objects.json`、`lane_graph.json` 的子目录 |
| 全选 / 清空 | 批量勾选 |

### Planner

| 参数 | 默认 | 说明 |
|------|------|------|
| **规划器** | `local_dwa` | `local_dwa`：局部 DWA；`reference_tracker`：沿参考线跟踪；`goal_seek`：朝 goal 行驶 |
| **参考线** `reference_source` | `map` | `map`：车道图 BFS 路由参考线；`sdc`：Waymo 自车轨迹作 reference |
| **参考步长** `reference_step` (m) | `1.0` | `map` 模式下参考线重采样间距 |
| **期望速度** `desired_speed` (m/s) | `13.9` | 规划器目标速度（约 50 km/h） |

### Metrics

| 参数 | 说明 |
|------|------|
| **指标多选** | 写入临时 metrics JSON，传给 `grading_main`。可选：`planning_limit_checker`、`speed_checker`、`regulatory_collision_checker`、`lane_departure_checker`、`drivable_area_checker` |
| **grading 模式** `cpp_mode` | 见下表 |

**`cpp_mode`（对应 `run_sim.py --cpp-mode`）**

| 值 | 行为 |
|----|------|
| `online` | 仿真每帧经 pipe 流式送入 `grading_main`，结束写 report |
| `offline` | 仅写 `sim_log.json`，结束后由 grading 离线读 log |
| `both` | 流式 + 落盘 SimLog（默认） |
| `off` | 不启 grading（与取消「运行 grading」等效） |

取消勾选 **运行 grading** 时，实际传 `--cpp-mode off`，且不传 `grading-bin`。

### 仿真 / 日志

| 参数 | 默认 | 说明 |
|------|------|------|
| **dt** (s) | `0.1` | 仿真时间步 |
| **max-seconds** | `0` | `0` = 跑满场景时长；`>0` 只跑前 N 秒 |
| **spdlog 等级** `log_level` | `info` | `trace`…`off`；`debug` 时每帧写 planner 观测等 tag 到 `sim_*.log` |
| **log-dir** | 空 | 非空则 spdlog 文件目录；空且 level 为 `info`/`off` 可不落盘详细文件 |
| **运行 grading** | 勾选 | 控制是否调用 `grading_main` |

### GIF 可视化

| 参数 | 默认 | 说明 |
|------|------|------|
| **生成 GIF** | 勾选 | 仿真成功后调用 `viz_sim.py` |
| **fps** | `120` | 动画帧率 |
| **dpi** | `100` | 输出分辨率 |
| **viz 参考步长** `gif_reference_step` (m) | `1.0` | 绘图时参考线采样步长（可与仿真 `reference_step` 不同） |

### 未在 UI 暴露、CLI 可用的项

完整仿真参数见 `hyw-sim/启动参数说明.md`，例如自车 `--ego-length`、`--ego-width`、`--stop-on-collision` 等。批跑 CLI：

```bash
cd hyw-workbench
python3 tools/batch_run_scenarios.py --scenarios waymo_scenario_5 --planner local_dwa
```

---

## Submodule 日常更新

在子仓改代码并 `git push` 后，回到元仓记录新指针：

```bash
cd hyw
git add hyw-sim    # 或 hyw-proto / hyw-grading / hyw-workbench
git commit -m "Bump hyw-sim"
git push
```

他人同步整套：

```bash
git pull --recurse-submodules
# 或
git submodule update --init --recursive
```

---

## 相关文档

- 仿真参数全文：`hyw-sim/启动参数说明.md`
- Workbench 工具：`hyw-workbench/README.md`
- Proto 约定：`hyw-proto/README.md`
