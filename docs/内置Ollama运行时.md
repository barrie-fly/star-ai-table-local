# 内置 Ollama 运行时

> v2.1.0 起，starAItable 可**自行下载、拉起并管理 Ollama**——用户无需安装 Ollama 桌面端 /
> 命令行或任何其他外部组件。本文说明模块划分、工作方式与全部开关。

## 一、工作原理

```
首次 runtime start / runtime pull / API 触发
        │
        ▼
┌─────────────────────────────┐
│ OllamaEmbeddedRuntime 编排器 │  ← 状态机 + single-flight
└──────┬──────────────┬───────┘
       │ 1. 取件       │ 2. 拉模型
       ▼              ▼
┌──────────────┐  ┌──────────────────────┐
│ HttpFile     │  │ POST /api/pull (NDJSON)│
│ Downloader   │  │ 分层进度 + 失败重试     │
└──────┬───────┘  └──────────┬───────────┘
       ▼                     ▼
┌──────────────┐   ┌─────────────────────┐
│ Archive      │   │ SupervisedProcess    │  ← ollama serve 托管
│ Extractor    │   │ （复用外部实例 / 只关 │
│ zip/tgz 解包 │   │   停自己拉起的实例） │
└──────────────┘   └─────────────────────┘
```

1. **找**：`ModelRuntime.FindBinary` 按序探测 `OLLAMA_BIN` 环境变量 → 内置运行时 →
   `PATH`（where/which）→ 常见安装路径。
2. **装**：全部找不到且允许自动安装时，按平台（win/linux/osx × x64/arm64）下载官方分发包，
   校验后解包到 `<root>/runtime/ollama/<rid>/`（先解 staging、再整体换名，安装目录任意时刻完整可用）。
3. **跑**：`SupervisedProcess` 拉起 `ollama serve`（限次自愈、日志/PID 落盘、只关停自己拉起的实例）；
   外部已有实例在跑时直接复用，不做任何安装动作。
4. **取模型**：`runtime pull` / `POST /api/model-runtime/pull` 走 Ollama `/api/pull` 流式接口，
   分层聚合进度；中断后重试由 Ollama 断点续传，代价极低。

### 安装布局（全部自包含，卸载 = 删除 `runtime/` 目录）

```
<root>/runtime/ollama/
├── <rid>/                  # 平台运行时（win-x64 / linux-arm64 / osx-arm64 / …）
│   ├── ollama(.exe)        #   可执行文件（官方包内布局）
│   └── embedded-manifest.json   # 安装清单（版本标记 / 来源 / 字节数 / 时间）
└── models/                 # 模型库（OLLAMA_MODELS 默认落点）
```

## 二、模块划分（`src/LocalServe.Core/Runtime/OllamaEmbedded/`）

| 文件 | 职责 |
|---|---|
| `HttpFileDownloader.cs` | 通用下载器：多来源回退、限次重试（指数退避 + 抖动）、Range 断点续传、SHA-256 / 魔数校验、读流空闲看门狗、进度节流、原子落盘 |
| `OllamaDistribution.cs` | 平台寻址：RID 识别、资产 URL 优先级、安装/staging/模型库目录、二进制定位、安装清单读写 |
| `ArchiveExtractor.cs`   | zip / tgz 解包：魔数分派、zip-slip / tar 穿越 fail-closed 防护、POSIX 可执行位 |
| `OllamaEmbeddedRuntime.cs` | 编排：安装状态机（single-flight）+ 模型拉取任务（流式进度 / 重试 / 取消） |

全部 BCL 实现（HttpClient、ZipArchive、System.Formats.Tar），Core 保持零第三方依赖。

## 三、使用方式

### CLI

```bash
localserve runtime install          # 手动获取内置运行时（进度条 + 重试提示）
localserve runtime pull             # 拉取角色清单基座模型（如 qwen3:8b）
localserve runtime pull qwen3:8b    # 拉取指定模型
localserve runtime start            # 拉起模型服务（无二进制时自动安装内置运行时）
localserve runtime status           # 状态（含「运行时来源：内置/外部安装」）
```

### API（`/api/model-runtime`）

| 端点 | 方法 | 权限 | 说明 |
|---|---|---|---|
| `/api/model-runtime/embedded` | GET | 登录 | 安装快照 + 拉取任务快照（前端进度条轮询） |
| `/api/model-runtime/embedded/install` | POST | admin | 后台触发安装（single-flight，202 已受理） |
| `/api/model-runtime/pull` | POST | admin | 拉取模型（缺省 = 角色清单基座模型） |
| `/api/model-runtime/pull/status` | GET | 登录 | 拉取进度（分层聚合百分比 / 速度 / 状态文本） |
| `/api/model-runtime/pull/cancel` | POST | admin | 取消拉取（已下载的层不丢失） |

## 四、环境变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `WTM_OLLAMA_EMBEDDED` | `1` | `=0` 关闭内置运行时子系统（回到「只找外部安装」的旧行为） |
| `WTM_OLLAMA_AUTO_INSTALL` | `1` | `=0` 禁止**隐式**下载（`runtime start` / API 自举不再触发；显式 `runtime install` 不受限） |
| `WTM_OLLAMA_URL` | — | 显式下载来源，分号分隔多个、按序回退；支持 `{asset}` 占位符，不含占位符则视为目录前缀。内网镜像 / 代理环境从这里接管 |
| `WTM_OLLAMA_VERSION` | latest | 锁定版本（如 `0.12.6`），走 GitHub Release 资产，可复现安装 |
| `WTM_MODEL_HOST` / `OLLAMA_BASE` | `http://127.0.0.1:11434` | 模型服务地址（拉取模型走同一地址） |
| `LOCALSERVE_RUNTIME_AUTOSTART` | `1` | `=0` 关闭启动即拉起（集成测试约定；自然也关掉自举下载） |

默认下载来源优先级：`WTM_OLLAMA_URL` > `WTM_OLLAMA_VERSION`（GitHub Release）>
ollama.com 官方最新 > GitHub Releases latest。

## 五、模型库选择优先级（`OLLAMA_MODELS` 未显式设置时）

1. `D:\ollama`（本机既有约定目录，含 `manifests/`）
2. `~/.ollama`（外部 Ollama 的默认库——**保住用户已有模型**，避免升级到内置后「基座模型未导入」）
3. 内置自包含库 `<root>/runtime/ollama/models`（全新机器的默认落点）
4. 都不满足 → 交给 Ollama 默认行为

## 六、可靠性设计

- **降级不阻断**：下载 / 安装 / 拉取失败只记录状态与日志，主服务照常运行（模型是可选依赖）。
- **single-flight**：同一时刻至多一个安装任务、一个拉取任务；重复触发返回现有快照。
- **校验落盘**：魔数（PK / 1F 8B）+ 可选 SHA-256，通过后才原子改名；`.part` 半成品永不生效。
- **断点续传**：下载器 Range 续传；模型拉取靠 Ollama 分层机制天然续传。
- **僵死检测**：读流空闲 90s（下载）/ 120s（拉取）视为连接僵死 → 自动重试或换源。
- **安全**：解包 zip-slip / tar 路径穿越 fail-closed 拒绝；镜像 URL 由使用者显式配置，默认仅官方来源。
- **测试**：`tests/LocalServe.Core.Tests/OllamaEmbeddedTests.cs` 19 例（网络用例走本机 HttpListener，不出网）。

## 七、常见问题

- **首次要下多大？** Windows 约 1-2 GB（zip），Linux 约 1 GB（tgz）；模型另计（qwen3:8b 约 5 GB）。
- **已经装了 Ollama 会怎样？** 直接复用你正在运行的实例（连下载都不会发生）；未运行时内置运行时优先。
- **如何离线部署？** 在有网机器上安装后，把 `<root>/runtime/ollama/` 整目录随包分发即可；
  或用 `WTM_OLLAMA_URL` 指向内网镜像。
- **如何升级？** 设置 `WTM_OLLAMA_VERSION` 后删除 `<root>/runtime/ollama/<rid>/` 重装；
  模型库不受影响。
- **如何彻底卸载？** 删除 `<root>/runtime/` 目录（本目录已加入 `.gitignore`，不入库）。
