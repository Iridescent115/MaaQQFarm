# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在此仓库中工作时提供指导。

## 项目简介

MaaQQFarm 是基于 [MaaFramework](https://github.com/MaaAssistantArknights/MaaFramework) 的自动化项目。MaaFramework 是一个以图像识别驱动黑盒自动化（如游戏脚本）的 C++ 框架。本仓库定义了任务流水线、自定义 Python 逻辑以及多平台打包配置。

## 常用命令

### 环境配置

```bash
# 安装 Node 开发工具
npm ci

# 安装 MaaFramework Python SDK（预发布版）
python -m pip install --upgrade maafw --pre

# 下载并配置 OCR 模型（运行 agent 前必须执行）
python tools/configure.py
```

### 校验

```bash
# 校验 MaaFramework 资源文件与 schema
npx @nekosu/maa-tools check

# 校验流水线/接口文件的 JSON Schema
python tools/validate_schema.py --schema-dir deps/tools --resource-dirs assets/resource --exclude-dirs assets/resource/announcement --interface-files assets/interface.json
```

### 构建 / 打包

```bash
# 针对指定平台打包（CI 自动调用，本地按需使用）
python tools/install.py <版本> <系统> <架构>
# 系统: win | macos | linux | android
# 架构: x86_64 | aarch64
```

### 发布

推送版本标签即可触发 CI 完整构建和发布：

```bash
git tag v1.0.0 && git push origin v1.0.0
```

## 架构说明

### 核心概念

MaaFramework 执行**流水线**：以 JSON 定义的任务序列，每个节点包含**识别**阶段（定位 UI 元素或图像）和**动作**阶段（点击、滑动、按键等）。超出框架原生能力的自定义逻辑通过 Python 实现。

### 各层职责

| 层级 | 位置 | 职责 |
|------|------|------|
| UI / 任务配置 | `assets/interface.json` | 声明可用任务、控制器（ADB/Win32）及可选 agent |
| 任务流水线 | `assets/resource/pipeline/*.json` | MaaFramework 执行的 JSON 任务图 |
| 自定义 Python 逻辑 | `agent/` | 流水线调用的自定义识别与动作 |
| 图像 / OCR 模型 | `assets/resource/image/`、`assets/resource/model/ocr/` | 识别节点使用的资源文件 |
| Schema 定义 | `deps/tools/*.schema.json` | 用于校验的 JSON Schema（从上游同步） |

### Agent（`agent/`）

- `main.py` — 启动 `AgentServer`（基于 socket 的 IPC），需在此导入所有自定义类以完成注册。
- `my_action.py` — 使用 `@AgentServer.custom_action()` 装饰的 `CustomAction` 示例子类。
- `my_reco.py` — 使用 `@AgentServer.custom_recognition()` 装饰的 `CustomRecognition` 示例子类。

自定义类接收 `Context` 对象，可在运行时覆盖流水线节点并访问任务参数。

### 流水线 JSON 约定

- 文件位于 `assets/resource/pipeline/`。
- 每个顶层键为任务节点名称。
- 节点通过装饰器中注册的名称引用自定义 Python 类，例如：`"recognition": "CustomRecognition", "custom_recognition_param": "my_reco_222"`。
- 流水线文件支持带注释的 JSON（JSONC）；`tools/validate_schema.py` 在校验前会自动去除注释。

### interface.json（v2）

定义最终用户看到的项目 UI：
- `controllers`：ADB（Android）和/或 Win32（Windows 桌面）目标。
- `tasks`：用户可见的任务列表及选项。
- `resource`：`assets/resource` 的路径。
- `agent`：可选，指向 `agent/main.py` 的路径，用于启用自定义 Python 逻辑。

## 提交规范

本仓库使用**约定式提交**（`feat:`、`fix:`、`docs:`、`chore:` 等）。变更日志通过 `git-cliff` 从提交信息自动生成（配置文件：`.github/cliff.toml`）。

## CI 工作流

| 工作流 | 触发条件 | 功能 |
|--------|----------|------|
| `check.yml` | push / PR | 运行 schema 校验 |
| `install.yml` | 标签 `v*.*.*` | 构建并上传多平台发布包 |
| `sync_schema_files.yml` | 手动 / 上游变更 | 同步 `deps/tools/*.schema.json` |
