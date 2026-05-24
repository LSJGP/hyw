# HYW 工作区

本目录 **不是** git 仓库，仅用于并排放置四个独立项目：

```
~/hyw/
├── hyw-proto/       # .proto 唯一来源（11 个文件）
├── hyw-grading/     # C++ 评分（原 grading_mini）
├── hyw-sim/         # C++ 仿真（原 sim）
└── hyw-workbench/   # 集成：tools、web、scenarios、pysim
```

## 首次构建

```bash
# 1. 评分
cd hyw-grading
bazel build //src/entry:grading_main

# 2. 仿真（依赖 hyw-proto）
cd ../hyw-sim
bazel build //cpp:sim_runner

# 3. Dashboard
cd ../hyw-workbench
./start_dashboard.sh
```

## Proto 一致性

所有 `.proto` 只存在于 `hyw-proto/`。修改后请在 grading/sim 中重新 build。

校验与旧单体仓是否逐字节一致：

```bash
./hyw-proto/scripts/verify_proto_sync.sh /path/to/hyw_grading
```

## 环境变量（可选）

见 `hyw-workbench/hyw.env.example`。默认假定四个目录均在 `~/hyw/` 下。
