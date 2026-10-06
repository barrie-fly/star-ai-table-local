# starAItable AI 本地部署操作指南

> 文档版本：v1.0 ｜ 适用产品：星智台 starAItable（内置 Ollama 运行时 + AI 本地部署向导）
> 适用对象：运维工程师 / AI 部署实施人员 ｜ 更新日期：2026-10-06

---

## 目录

1. [部署架构概述](#1-部署架构概述)
2. [环境配置要求](#2-环境配置要求)
3. [依赖项安装](#3-依赖项安装)
4. [部署流程](#4-部署流程)
5. [参数设置](#5-参数设置)
6. [性能优化](#6-性能优化)
7. [系统维护](#7-系统维护)
8. [常见问题排查](#8-常见问题排查)

---

## 1. 部署架构概述

### 1.1 组件结构

```
┌─────────────────────────────────────────────────┐
│                starAItable 服务端                │
│  LocalServe.Api（Web 宿主）                       │
│     ├── 内置 Ollama 运行时（OllamaEmbeddedRuntime）│
│     ├── 角色网关 gateway.py（OpenAI 兼容 :8010）  │
│     └── Python 控制桥                             │
└──────────────────┬──────────────────────────────┘
                   │ WTM_MODEL_HOST
                   ▼
        ┌─────────────────────┐
        │  Ollama 模型服务   │  http://127.0.0.1:11434
        │  （本地推理）      │
        └─────────────────────┘
```

### 1.2 内置运行时工作原理

1. **找**：按序探测 `OLLAMA_BIN` → 内置运行时 → `PATH` → 常见安装路径。
2. **装**：找不到且允许自动安装时，按平台下载官方分发包，校验后解包到 `<root>/runtime/ollama/<rid>/`。
3. **跑**：拉起 `ollama serve`（限次自愈、日志/PID 落盘、只关停自己拉起的实例）。
4. **取模型**：走 Ollama `/api/pull` 流式接口，分层聚合进度，断点续传。

> **设计原则**：模型是可选依赖。下载/安装/拉取失败只记录状态与日志，主服务照常运行。

---

## 2. 环境配置要求

### 2.1 操作系统要求

| 平台 | 支持架构 | 说明 |
|---|---|---|
| Windows | x64 | 10/11，客户端与部署向导仅 Windows |
| Linux | x64 / arm64 | 主流发行版（Ubuntu 20.04+ / Fedora 34+ / Debian 11+） |
| macOS | x64 / arm64 | 12+ |

### 2.2 硬件建议

| 规模 | CPU | 内存 | 磁盘 | GPU（可选） |
|---|---|---|---|---|
| 试用/开发 | 4 核 | 16 GB | 20 GB 可用 | 无 |
| 生产（7B 模型） | 8 核 | 32 GB | 50 GB 可用 | 建议 |
| 生产（32B 模型） | 16 核 | 64 GB | 100 GB 可用 | 强烈建议 |

### 2.3 软件依赖

| 组件 | 版本 | 是否必需 |
|---|---|---|
| .NET SDK | **10.0** 及以上 | 必需（构建/运行） |
| Node.js | ≥ 18 | 可选（重新装配前端时） |
| Ollama | 任意近期版本 | 可选（内置运行时自动管理） |
| Docker | ≥ 20 | 可选（命令沙箱 docker 后端） |

### 2.4 网络要求

- **在线部署**：可访问 ollama.com / GitHub Releases 以下载运行时与模型。
- **离线部署**：需预先在内网镜像或离线包中准备运行时与模型文件（见 §4.5）。

---

## 3. 依赖项安装

### 3.1 安装 .NET SDK

```bash
# Windows（winget）
winget install Microsoft.DotNet.SDK.10

# Linux (Ubuntu)
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update && sudo apt-get install -y dotnet-sdk-10.0

# macOS（brew）
brew install --cask dotnet-sdk
```

验证：`dotnet --version` 输出 `10.0.x`。

### 3.2 获取源码并构建

```bash
git clone https://github.com/barrie-fly/star-ai-table.git
cd star-ai-table
dotnet build localserve.slnx
```

预期结果：`0 个错误`（警告可忽略）。

### 3.3 运行测试（建议）

```bash
dotnet test localserve.slnx
```

三套测试（Core / Api / Cli）全绿即环境就绪。

### 3.4 安装内置 Ollama 运行时

```bash
# 方式一：CLI 手动安装
dotnet run --project src/LocalServe.Cli -- runtime install

# 方式二：API 后台触发（需 admin）
POST /api/model-runtime/embedded/install

# 方式三：启动服务自动安装（默认开启，可用 WTM_OLLAMA_AUTO_INSTALL=0 关闭）
dotnet run --project src/LocalServe.Api
```

### 3.5 安装 AI 本地部署向导（Windows 桌面）

在 Windows 安装包中运行 `StarAI.AiDeployer\StarAI.AiDeployer.exe`，按向导完成：

1. 选择安装目录（默认 `%USERPROFILE%\.starai\ai-runtime`）。
2. 点击「一键安装内置 Ollama 运行时」。
3. 选择基座模型并拉取。
4. 完成后客户端自动连接 `http://127.0.0.1:11434`。

---

## 4. 部署流程

### 4.1 快速部署（源码运行）

```bash
# 启动 Web 服务
dotnet run --project src/LocalServe.Api
# 默认监听 http://localhost:5100
```

### 4.2 生产部署（发布产物）

```bash
# 发布单文件自包含产物（以 Linux x64 为例）
dotnet publish src/Server/StarAI.Server.csproj -c Release -r linux-x64 \
  --self-contained true -p:PublishSingleFile=true -o dist/server-linux-x64

# 运行
export WTM_DATA_DIR=/var/lib/staraitable
export JWT_SECRET=<强随机值>
./dist/server-linux-x64/StarAI.Server --urls http://0.0.0.0:5180
```

### 4.3 拉取模型

```bash
# 拉取默认基座模型（角色清单基座）
dotnet run --project src/LocalServe.Cli -- runtime pull

# 拉取指定模型
dotnet run --project src/LocalServe.Cli -- runtime pull qwen3:8b

# 查看拉取进度（API 轮询）
GET /api/model-runtime/pull/status
```

### 4.4 验证部署

```bash
# 健康检查
curl http://localhost:5100/api/health

# 端到端自检（14 项）
dotnet run --project src/LocalServe.Cli -- selftest

# 运行时状态
dotnet run --project src/LocalServe.Cli -- runtime status
```

### 4.5 离线部署（内网/隔离环境）

方案 A：整目录分发

1. 在有网机器上完成安装与模型拉取。
2. 将 `<root>/runtime/ollama/` 整目录（含 `models/`）拷贝到目标机。
3. 启动服务，运行时自动复用已安装实例。

方案 B：内网镜像

1. 配置 `WTM_OLLAMA_URL` 指向内网镜像（支持 `{asset}` 占位符，分号分隔多个、按序回退）。
2. 模型文件放入 `OLLAMA_MODELS` 指定目录。

---

## 5. 参数设置

### 5.1 环境变量总表

| 变量 | 默认 | 说明 |
|---|---|---|
| `PORT` | 5100 | 监听端口 |
| `JWT_SECRET` | 开发随机 | **生产必填** |
| `DATABASE_PATH` | 数据目录 | SQLite 路径 |
| `TRUST_PROXY` | 0 | =1 采信 `X-Forwarded-For` |
| `WTM_OLLAMA_EMBEDDED` | 1 | =0 关闭内置运行时子系统 |
| `WTM_OLLAMA_AUTO_INSTALL` | 1 | =0 禁止隐式下载 |
| `WTM_OLLAMA_URL` | — | 显式下载来源（内网镜像） |
| `WTM_OLLAMA_VERSION` | latest | 锁定版本（可复现安装） |
| `WTM_MODEL_HOST` / `OLLAMA_BASE` | `http://127.0.0.1:11434` | 模型服务地址 |
| `LOCALSERVE_RUNTIME_AUTOSTART` | 1 | =0 关闭启动即拉起 |
| `LOCALSERVE_PM_BRIDGE` | 1 | =0 关闭 PM 事件桥后台服务 |
| `BACKUP_INTERVAL_HOURS` | 24 | 备份间隔（0 关闭） |
| `BACKUP_KEEP` | 7 | 备份保留份数 |

### 5.2 模型倍率配置（积分系统联动）

AI 调用消耗积分 = 基础消耗 × 模型倍率。倍率在管理端「积分管理 → 模型 AI 倍率」中按模型配置。

### 5.3 运行时来源优先级

`WTM_OLLAMA_URL` > `WTM_OLLAMA_VERSION`（GitHub Release）> ollama.com 官方最新 > GitHub Releases latest。

### 5.4 模型库选择优先级（`OLLAMA_MODELS` 未设置时）

1. `D:\ollama`（本机既有约定目录）
2. `~/.ollama`（外部 Ollama 默认库——保住已有模型）
3. 内置自包含库 `<root>/runtime/ollama/models`
4. 都不满足 → 交给 Ollama 默认行为

---

## 6. 性能优化

### 6.1 服务端性能基线

- `/api/health` P50 约 **1.19ms**（C# 版实测，2026-09-19 同机 400 次）。
- 权限验证 P95 < 200ms；权限变更传播 ≤5 分钟。
- 并发用户目标 ≥10,000（多实例 + SQLite WAL / PostgreSQL）。

### 6.2 模型推理优化

| 优化项 | 说明 |
|---|---|
| 量化模型 | 使用 Q4_K_M 等量化版本，显存占用降低约 75% |
| 批大小 | Ollama 环境变量 `OLLAMA_NUM_PARALLEL` 控制并行请求数 |
| 上下文长度 | 按需设置，避免过长上下文拖慢推理 |
| 模型选择 | 7B 模型适合日常补全/问答；32B 适合复杂代码任务 |
| GPU 加速 | 配置 `CUDA_VISIBLE_DEVICES` 指定 GPU |

### 6.3 应用层优化

- **缓存**：命令缓存 TTL 可调（`cache ttl 120`），只缓存只读命令。
- **后台任务**：长任务使用 `--priority low` 后台执行，避免阻塞主流程。
- **数据库**：SQLite WAL 模式支持并发读写；大数据量建议迁移 PostgreSQL（预留数据访问层）。

---

## 7. 系统维护

### 7.1 升级

1. 备份：`dotnet run --project src/LocalServe.Cli -- backup run`。
2. 停止服务 → 替换二进制 → 启动。
3. 启动时 SchemaMigrator 自动应用新迁移（只增不删）。
4. 验证：`selftest` 通过，登录与授权正常。

### 7.2 模型升级

```bash
# 锁定版本后删除旧运行时目录重装
export WTM_OLLAMA_VERSION=0.12.6
rm -rf <root>/runtime/ollama/<rid>/
dotnet run --project src/LocalServe.Cli -- runtime install
```

模型库不受影响。

### 7.3 备份与恢复

- 定时备份：`BACKUP_INTERVAL_HOURS=24`，`VACUUM INTO` 一致性快照。
- 手工备份：`dotnet run --project src/LocalServe.Cli -- backup run --keep 7`。
- 恢复：停止服务 → 用最近全量备份 + WAL 恢复 → 启动 → `PRAGMA integrity_check`。

### 7.4 日志与监控

- 运行日志：`logs/` 目录（打包日志）与程序数据目录（运行日志）。
- 访问统计：`/api/admin/stats/*`（PV/UV/DAU/错误率/耗时）。
- 开发自省：`/api/dev`（内存与缓存；生产默认 404）。

### 7.5 卸载

删除源码目录与数据目录即可；内置运行时卸载 = 删除 `<root>/runtime/` 目录。

---

## 8. 常见问题排查

### 8.1 症状对照表

| 症状 | 排查步骤 |
|---|---|
| 构建报 `LocalServe.Api.dll` 被占用 | `taskkill //F //IM LocalServe.Api.exe` 后重试 |
| 测试莫名拉起 ollama | 设置 `LOCALSERVE_RUNTIME_AUTOSTART=0` |
| 改角色清单不生效 | 运行 `python scripts/wtm_models.py sync` |
| 管理后台不显示 | 检查 `users.role` 是否为 admin + 重新登录换 JWT |
| 下载 ZIP 返回 500 | 检查是否同步写 `Response.Body`（须临时文件 + `CopyToAsync`） |
| UI 改动丢失 | 改 `www/index.html` 而非 `.nexus/` 源，被装配回退 |
| 双调度器重复执行自动化 | 只保留一侧调度循环 |

### 8.2 运行时相关

**Q：首次要下多大？**
A：Windows 约 1-2 GB（zip），Linux 约 1 GB（tgz）；模型另计（qwen3:8b 约 5 GB）。

**Q：已经装了 Ollama 会怎样？**
A：直接复用正在运行的实例（连下载都不会发生）；未运行时内置运行时优先。

**Q：如何离线部署？**
A：在有网机器上安装后，把 `<root>/runtime/ollama/` 整目录随包分发；或用 `WTM_OLLAMA_URL` 指向内网镜像。

**Q：下载/安装失败怎么办？**
A：降级不阻断——主服务照常运行。检查 `WTM_OLLAMA_URL` 与网络；下载器支持 Range 断点续传，重试代价低。

**Q：如何彻底卸载 AI 运行时？**
A：删除 `<root>/runtime/` 目录（已加入 .gitignore，不入库）。

---

*如遇问题，请参阅配套文档：《内置Ollama运行时.md》、《安装指南.md》、《部署运维手册》。*