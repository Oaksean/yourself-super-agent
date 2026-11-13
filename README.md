# Local Super Agent

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

一个本地桌面版 **Router / Worker 双层智能体**。Router 常驻，只负责会话、长期记忆、任务分派、审批与汇总；Worker 按任务拉起、独立进程执行、用完即退。

> 本仓库是 `0.4.0` 的可公开源码（MIT License）。安装包通过 GitHub Releases 单独发布，不进入 Git 历史。安全问题请按 [SECURITY.md](SECURITY.md) 私密报告，不要提交 API Key、数据库、日志或私有路径。

## 架构

```
Tauri 2 桌面壳
    │  (启动/监督)
    ▼
Router Daemon (Node, 常驻)         LLM Provider (DeepSeek / OpenAI 兼容)
    ├── Session Store  长期记忆 + 向量检索
    ├── Task Router    规则优先 + LLM 分派
    ├── Watchdog       卡死/循环检测
    ├── Watcher        文件变化 -> Trigger
    └── Worker 沙箱池
            │ Windows Job Object / AppContainer
            ▼
       一次性 Worker 进程（读写/命令工具 + 审批门）
```

三条不可妥协的边界：

1. **Router 不干活。** 不直接读写文件、不执行命令，只做理解、分派、汇总。
2. **Worker 用完即弃。** 独立进程、独立上下文，不能直接写长期记忆。
3. **执行按能力分级隔离。** 只读 / 可写 / 可执行三种能力面对应不同沙箱与审批策略。

## 快速开始

### 本地开发（仅 Node，最快）

```powershell
node bin/super-agent.mjs
```

打开 <http://127.0.0.1:4317>。未配置 API Key 时进入只读回退模式。

配置模型：

```powershell
$env:DEEPSEEK_API_KEY="你的 key"
node bin/super-agent.mjs
```

或在 UI 右上角「设置」中填写 Base URL、模型与密钥（仅保存在本机 `data/secrets.json`）。

### 桌面应用（Tauri 2）

桌面壳使用 Rust 官方的 **gnullvm** 目标（LLVM + UCRT），因此**不需要 Visual
Studio / Windows SDK**。链接器由 LLVM-MinGW 提供，首次运行脚本会自动尝试用
winget 安装；也可以手动执行：

```powershell
winget install MartinStorsjo.LLVM-MinGW.UCRT
```

安装包会内置 `runtime/node.exe`，桌面应用不依赖系统 PATH 或单独安装 Node.js。每轮构建会自动从当前 Node 环境复制运行时。

工具链固定在 `src-tauri/rust-toolchain.toml`，构建参数在
`src-tauri/.cargo/config.toml`（含 `crt-static`，见下方"打包说明"）。

```powershell
# 只编译可执行文件（最快，不生成安装包）
.\scripts\build-tauri.ps1

# 打包 NSIS + MSI 安装包
.\scripts\build-tauri.ps1 -Build

# 开发模式（带 devtools）
.\scripts\build-tauri.ps1 -Dev

# 已装好依赖、希望跳过自动安装
.\scripts\build-tauri.ps1 -Build -SkipInstall
```

产物：

| 文件 | 说明 |
|---|---|
| `src-tauri\target\release\local-super-agent.exe` | 主程序 |
| `src-tauri\target\release\bundle\nsis\*.exe` | NSIS 安装包（含中文安装界面） |
| `src-tauri\target\release\bundle\msi\*.msi` | MSI 安装包 |

打包要点（避免重复踩坑）：

- **`crt-static` 必需**：否则 gnullvm 产物会依赖 `libunwind.dll` /
  `libwinpthread-1.dll`，这些在纯净 Windows 上不存在，程序会以
  `0xC0000135`（找不到 DLL）直接退出。
- **`WebView2Loader.dll` 必须显式打包**：Tauri 打包器只在目标名以 `-gnu`
  结尾时才自动附带该 DLL，`gnullvm` 会被漏掉，导致安装后无法启动。
  工程已在 `tauri.conf.json` 的 `bundle.resources` 中显式映射它。
- **资源放在安装根目录**：`bundle.resources` 使用对象映射（而非 `../**`
  通配），避免 Tauri 把父目录资源塞进 `_up_/` 子目录。
- **内置 Node 运行时必需**：桌面快捷方式不会继承开发终端的 PATH；安装包必须包含 `runtime/node.exe`，否则 Router 与设置保存都无法工作。
- **可写数据在用户目录**：SQLite、日志与 UI 保存的设置写入
  `%LOCALAPPDATA%\LocalSuperAgent`，安装目录只读也不影响运行。

### 选择任务文件夹

主界面顶部会显示当前“任务文件夹”的完整路径。点击路径或“选择文件夹”会打开
Windows 原生目录选择器；保存后：

- 新任务固定使用该目录作为 `workspace`，读取、搜索、写入和命令工作目录都以它
  为基准解析相对路径。
- 该目录会加入 Worker 的 `allowedRoots` 白名单，重启后仍保留。
- 只有勾选“自动写入”时，Worker 才会直接写入当前任务文件夹；关闭后仍会逐次
  进入审批。命令执行始终受独立审批控制。
- “任务”侧栏的每个任务会显示自己实际使用的文件夹，避免切换后无法追溯。

非桌面浏览器模式会提示手动输入绝对路径，便于调试和远程本机访问。

首次打包需要联网下载 NSIS / WiX 工具（约 10 MB）。若直连 GitHub 失败，可先设置镜像：

```powershell
$env:TAURI_BUNDLER_TOOLS_GITHUB_MIRROR = "https://ghproxy.net/https://github.com"
.\scripts\build-tauri.ps1 -Build
```

## Windows 沙箱

`scripts/build-sandbox.ps1` 使用系统自带的 .NET Framework 编译器构建
`native/WindowsSandboxHost.exe`，无需额外 SDK。

两种隔离级别：

| 模式 | 机制 | 适用 |
|---|---|---|
| `job` | Windows Job Object，`KILL_ON_JOB_CLOSE` + 可选内存/子进程上限 | 默认，稳定可用 |
| `appcontainer` | AppContainer 受限令牌 + 按包 SID 授权 ACL | 需系统支持，不支持时自动回退 Job Object |

配置方式（`config/default.json` 或 UI「设置」）：

```json
{
  "sandboxMode": "auto",
  "sandboxMemoryMb": 2048,
  "sandboxAllowChildren": true
}
```

- `auto`：优先 AppContainer，失败回退 Job Object。
- `job` / `appcontainer`：强制指定。
- `off`：仅独立进程（不建议）。
- `allowProcessFallback: false`：沙箱不可用时直接拒绝启动，而不是降级。

验证：

```powershell
.\native\WindowsSandboxHost.exe --probe
node -e "import('./src/core/sandbox.mjs').then(m=>console.log(m.sandboxCapabilities({sandboxMode:'auto',projectRoot:process.cwd()})))"
```

## 文件监听 Trigger

把目录变成事件源：文件变化 → Trigger → 命中订阅则直通 Worker，否则交给 Router 决策。

```powershell
# 监听当前会话的工作区，仅 .md 文件
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:4317/api/watchers `
  -ContentType 'application/json' `
  -Body '{"sessionId":"<会话ID>","path":".","pattern":"\\.md$"}'
```

API：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/watchers` | 监听快照 |
| POST | `/api/watchers` | 新增监听（sessionId/path/pattern/events） |
| DELETE | `/api/watchers/:id` | 停止监听 |
| GET | `/api/watcher-events` | 监听事件历史 |
| POST | `/api/triggers` | 手动注入外部触发 |
| POST | `/api/subscriptions` | 注册 Thread Subscription（直通） |

忽略规则与防抖在 `config/default.json` 的 `watchers` 段配置。

## 长期记忆（Transcripts → Notes → Wiki → Index）

长期记忆是 Router 的独立基础设施，不把历史对话直接倾倒进上下文。系统按四层组织知识，并保留可回溯的原始底账：

完整技术设计、数据表、检索公式、Dreaming 状态机和验收清单见
[`docs/long-term-memory.md`](docs/long-term-memory.md)。

1. **原始流水 Transcripts**：用户消息、助手消息、Worker 工具调用与结果自动无损写入 `memory_transcripts`。它是纠错和灾难恢复的最终事实来源，不参与直接上下文注入。
2. **记忆笔记 Memory Notes**：保存事实、决定、偏好、任务结果、失败教训和技能。每条笔记带作用域、重要度、置信度、访问次数、时效、来源和指纹，支持人工增删改、置顶、反馈与归档。
3. **Wiki**：Dreaming 把稳定知识按主题整理成 Markdown 页面。页面正文存放在磁盘 Wiki，数据库保留摘要、版本、来源记忆和链接元数据。
4. **轻量注入索引 MEMORY.md**：只把目录型信息和最关键的笔记注入 Router 提示词，细节由 Router 或 Worker 通过 `memory_search`、`memory_read_page` 按需检索。

**硬性上下文预算**：`MEMORY.md` 每次最多写入 **200 行 / 25 KB**，按 UTF-8 字节严格截断；超出内容不会进入提示词，只保留在 Wiki 与原始流水中。可通过 UI 记忆中心查看当前行数、字节数和是否截断。

**作用域隔离**：记忆支持 `session`、`project`、`global` 三种作用域。`session` 只属于当前会话；`project`/`global` 可跨会话复用，但始终绑定当前任务文件夹，切换工作区不会发生记忆串扰。指纹去重同样按作用域隔离。

**语义召回**：默认使用内置确定性本地嵌入（384 维、字符 n-gram 哈希），零依赖、离线可用；配置 `embeddingModel` 后优先调用 OpenAI 兼容的 `/v1/embeddings`，失败自动回退本地。综合评分由向量相似度、关键词、置信度、重要度、时效、访问频次和置顶共同决定。

**Dreaming 后台整理**：Runtime 启动后按设定间隔执行离线整理，也可在记忆中心手动触发。它对照当前工作区的原始流水纠错、去重、合并、更新置信度、把长期未访问的笔记转为休眠/归档，并重新生成 Wiki 和 `MEMORY.md`。即使模型只返回 `keep`，确定性的去重与时效衰减仍会执行；置顶记忆不衰减。每次运行写入 `memory_runs` 并推送 `memory.dream.started/completed/failed` SSE 事件。

**文件位置**：

```text
%LOCALAPPDATA%\LocalSuperAgent\memory\workspaces\<工作区名>-<哈希>\MEMORY.md
%LOCALAPPDATA%\LocalSuperAgent\memory\workspaces\<工作区名>-<哈希>\wiki\*.md
```

开发模式下对应 `<dataDir>/memory/workspaces/...`。SQLite 原始流水、记忆笔记、Wiki 元数据和 Dreaming 运行记录均保存在 `dataDir/agent.db`。

```json
{
  "memory": {
    "embeddingDim": 384,
    "topK": 8,
    "minScore": 0.18,
    "maxInlineMemories": 12,
    "indexMaxLines": 200,
    "indexMaxBytes": 25600,
    "dreaming": {
      "enabled": true,
      "intervalHours": 6,
      "startupDelayMs": 60000,
      "batchSize": 24,
      "minMemories": 8,
      "staleDays": 30,
      "dormantDays": 90,
      "decayPerMonth": 0.08
    }
  }
}
```

设置页可开关 Dreaming 并调整间隔；其余参数可通过 `config/default.json` 或本机设置覆盖。记忆中心提供统计、语义搜索、手动写入、编辑、置顶、反馈、归档/恢复、删除、Wiki 阅读、原始流水和整理记录。

## 自动监督与完成通知

任务不是“发出去就结束”。Router 会把任务状态、尝试轮次、心跳和部分结果持久化，并由 Watchdog 持续监督到终态：

- Worker 每 10 秒发送心跳；停滞超过 3 分钟、重复进度或单轮超过 30 分钟时，Router 会终止当前 Worker 并自动续跑，不再直接取消长任务。
- 自动续跑会把上一轮的部分结果和中断原因交给下一轮，最多 3 轮；只有重试耗尽才标记为失败。
- 关闭桌面窗口只会隐藏到系统托盘，Router、Watchdog 和 Worker 继续运行；从托盘选择“退出”才会结束应用。
- 任务完成、最终失败或等待审批时，会发送 Windows 桌面通知并提示用户处理。
- Router 重启后会扫描 `queued` / `running` / `waiting_approval` 任务并自动恢复。历史 `cancelled` 任务不会自行重跑，可在任务卡点击“重新执行”或调用 retry API。
- UI 任务卡显示 `第 n/最大轮次`、最近心跳、当前进度和最终结果。

```json
{
  "taskTimeoutMs": 1800000,
  "taskMaxAttempts": 3,
  "taskRetryDelayMs": 3000,
  "taskHeartbeatMs": 10000,
  "watchdogIntervalMs": 15000,
  "watchdogStallMs": 180000
}
```

任务完成、失败和审批仍通过 `/api/events` SSE 实时推送；桌面壳在窗口隐藏时也会保持该连接。

## 多智能体协同（MAS）

Router/Worker 解决的是“一个任务谁来做”，MAS 解决的是“一个复杂目标由一支 Agent 团队怎么做”。模块位于 `src/core/mas/`，不引入新依赖，与现有 Router/Worker 共用同一套模型接入、SQLite、事件总线与工作区安全校验。

**架构：中心化编排（Orchestrator-Centric）**。编排者负责目标分析与拆解，专家 Agent 并行执行，独立 Verifier 回抓原文校验引用，形成“写作—审校—仲裁”的收敛闭环，最后由 Archivist 落盘。首版把子 Agent 控制在 3–4 个（深度 1–3 对应 2–4 个检索 Agent），与“≤5–7 个再分层扩容”的结论一致。

**编排引擎（`mas/graph.mjs`）**：自研 StateGraph，零依赖。

- 节点规格：`{ label, role, kind, deps, run, timeoutMs, retries, failurePolicy }`；`deps` 支持字符串或 `{ id, optional, when(state) }`。
- 同一层、无依赖的节点真正的并发执行（默认并发 3）；`when` 为假则跳过并级联跳过下游。
- 节点失败按 `retries` 重试；`failurePolicy: 'abort'` 中断整图，`'continue'` 只标记该节点失败。
- `validate()` 在编译期检测未知依赖与强依赖环（条件边不参与环检测）；`AbortController` 取消返回 `cancelled` 而不是 `failed`。

**通用任务协作层（CollaborationCoordinator）**

聊天主链路不再把所有 Worker 一次性平铺发出。Router 产生的多任务计划会升级为持久化协作组，模块位于 `src/core/collaboration.mjs`：

- **中心化 DAG**：每个 Agent 带 `agentRole`、`dependsOn`、`handoff`、`verification`；只允许引用更早节点，执行前拒绝自依赖、未知依赖、循环和超过 5 个 Agent 的计划。
- **并行与交接**：无依赖节点立即并行；下游只在上游全部成功后启动。Coordinator 把直接上游及祖先结果压缩成结构化交接上下文注入下游提示词，Agent 之间不建立隐藏 P2P 通信。
- **独立校验/仲裁**：计划可包含 `reviewer` / `arbiter` Agent，校验节点依赖实现节点，并在验收标准未满足时明确列出问题；高风险写入和命令仍走原有审批门。
- **失败收敛**：`failurePolicy=continue` 隔离失败分支并继续独立分支；`stop` 在关键失败后取消未完成 Agent。协作组还有总超时、Agent 上限、任务重试上限和调度停滞检测。
- **只统一通知一次**：协作成员完成时 Router 不逐条生成报告；整个 DAG 进入终态后，Coordinator 才调用 Router 汇总并写入一条 `collaboration_report`，同时通知桌面端。
- **可恢复与可审计**：新增 `collaboration_runs` 表，任务表增加运行、角色、依赖、交接和校验字段；重启后恢复未完成协作组。UI “协作”页展示角色、DAG 上游、状态、轮次、交接和最终交付。

API：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/collaborations?sessionId=...` | 当前会话协作组 |
| GET | `/api/collaborations/:id` | 协作组与全部 Agent 详情 |
| POST | `/api/collaborations/:id/cancel` | 取消协作组及未完成 Agent |
| POST | `/api/collaborations/:id/retry` | 按相同 DAG 创建新的协作运行 |
**首个业务场景：技术调研报告流水线**

```
brief(目标分析) → plan(任务拆解) → scout_1..N(并行联网检索) → verify(引用回抓校验)
                                                              → write(撰写) → review(审校)
                                                              → arbitrate(仲裁) → finalize(归档落盘)
```

- **检索层（`mas/tools.mjs`）**：中文查询走百度、非中文走 Bing 并在失败时降级百度；百度 `/link?url=` 跳转逐跳还原为原始网址；结果按 query token 做相关性过滤，避免把词典页当证据。学术场景可直连 arXiv，代码场景可直连 GitHub REST。`fetchPage` 只接受 html/text，超时/超限返回 `{ok:false}` 而不是抛异常。
- **引用校验**：每条论断带 `quote` 原文与 `sourceId`，Verifier 重新抓取该页并做整句/词覆盖匹配，产出 `verified.ok`、`passRate`。做不到回抓的论断会被标记为未通过，不允许 Writer 自行“补齐引用”。
- **闸门**：`passRate < 0.6` 或审校存在高优先级问题时仲裁返回 `revise` 并进入下一轮（最多 `maxRevisions` 轮，返工只重写不重检索）；轮次用尽强制交付，未解决问题写入 `openRisks`。若整轮零条可校验论断，同样在风险中显式披露，而不是伪装成“通过”。
- **产物**：`<workspace>/{reportDir}/{slug}-{时间戳}.md` 与同名 `.verify.json`（evidence ↔ claim 映射），路径经 `resolveSafePath` 校验；同时写入 `mas_runs` / `mas_steps` / `mas_artifacts` 三张表，UI 可直接回看每一步。

**配置**

```json
{
  "mas": {
    "enabled": true,
    "maxConcurrentRuns": 2,
    "maxSourcesPerScout": 4,
    "searchTimeoutMs": 12000,
    "fetchTimeoutMs": 12000,
    "nodeTimeoutMs": 300000,
    "reportDir": "多智能体报告"
  }
}
```

**UI**：右侧 inspector 的“协同”页签可填研究主题、选择检索深度与返工上限后启动；运行卡展示轮次、引用通过率、模型调用次数，展开后是按轮分组的执行链路（节点、状态、耗时、摘要、跳过原因），交付后可在线预览、复制或下载 `.md`。SSE 事件 `mas.run.*` / `mas.node.*` 实时驱动刷新。

**已知边界**：单机顺序/并发编排，无跨机 Agent 通信；DuckDuckGo、Google 抓取在本机不可用（前者网络不可达、后者页面已 JS 化），Bing 对“首词为中文”的 query 会返回词典意图结果，因此中文统一走百度；360 的 `/link?m=` 无法还原原始 URL。这些是检索后端约束，不是编排层缺陷。

## API 一览

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/health` | 运行时状态（沙箱 / 监听 / 记忆 / 模型） |
| GET/POST | `/api/sessions` | 会话列表 / 新建 |
| GET | `/api/sessions/:id` | 会话详情（消息、任务、记忆、审批） |
| POST | `/api/chat` | 发送用户消息 |
| GET | `/api/tasks` | 任务列表 |
| GET | `/api/tasks/:id` | 任务详情 |
| POST | `/api/tasks/:id/cancel` | 取消任务 |
| POST | /api/tasks/:id/retry | 重新执行失败或取消的任务 |
| GET | `/api/approvals` | 审批列表 |
| POST | `/api/approvals/:id/resolve` | 批准 / 拒绝 |
| GET/PUT | `/api/config` | 读取 / 更新设置 |
| POST | `/api/config/test` | 使用当前配置测试模型连接 |
| GET | `/api/events` | SSE 实时事件流 |
| GET | `/api/memory/overview` | 记忆统计、索引预算、Dreaming 状态 |
| GET/POST | `/api/memory/records` | 查询 / 手动写入长期记忆 |
| PATCH/DELETE | `/api/memory/records/:id` | 编辑、置顶、归档或删除记忆 |
| POST | `/api/memory/records/:id/feedback` | 对记忆质量给出正负反馈 |
| GET | `/api/memory/transcripts` | 按会话 / 任务 / 工作区检索原始流水 |
| GET | `/api/memory/pages` | Wiki 页面列表（可含正文） |
| GET | `/api/memory/pages/:id` | 读取单个 Wiki 页面 |
| POST | `/api/memory/search` | 向量 + 关键词混合语义检索 |
| POST | `/api/memory/sync` | 立即重建 `MEMORY.md` 与 Wiki 文件 |
| POST | `/api/memory/backfill` | 为历史记忆补齐向量索引 |
| POST | `/api/memory/dream` | 异步启动一次 Dreaming |
| GET | `/api/memory/runs` | Dreaming 运行历史与结果统计 |
| GET | `/api/memory/stats` | 当前会话 / 工作区的向量索引统计 |
| GET | `/api/collaborations` | 通用多智能体协作组列表 |
| GET | `/api/collaborations/:id` | 协作 DAG 与 Agent 详情 |
| POST | `/api/collaborations/:id/cancel` | 取消协作组 |
| POST | `/api/collaborations/:id/retry` | 重跑协作组 |
| GET | `/api/mas/templates` | 专项协同模板（当前为调研报告流水线） |
| GET/POST | `/api/mas/runs` | 协同运行列表 / 新建（`{ topic, depth, maxRevisions }`） |
| GET | `/api/mas/runs/:id` | 运行详情（含按轮分组的步骤与公开状态） |
| POST | `/api/mas/runs/:id/cancel` | 取消运行（中止在跑的节点） |
| POST | `/api/mas/runs/:id/retry` | 以相同参数重跑一次 |
| GET | `/api/mas/runs/:id/report` | 交付报告正文（`{ name, path, content }`） |
| GET | `/api/mas/artifacts/:id` | 单个产物（报告 / 校验清单） |

## 测试

```powershell
# 首次测试前从 C# 源码构建沙箱宿主；生成的 exe 不会进入 Git
.\scripts\build-sandbox.ps1
node --test @((Get-ChildItem tests -Filter '*.test.mjs').FullName)
```

沙箱宿主二进制是可重建产物，源码仓库只保留 `sandbox/windows/WindowsSandboxHost.cs`；
`native/WindowsSandboxHost.exe` 被 `.gitignore` 排除。

覆盖：规则路由、能力分级、路径越界、敏感文件、Worker 工具、写入审批、端到端任务、
订阅直通、沙箱解析与宿主、向量嵌入与语义召回、文件监听触发、
数据库迁移、Watchdog 自动恢复、重试轮次与重新执行；
长期记忆由 `tests/long-term-memory.test.mjs` 覆盖旧库迁移、消息流水、作用域隔离、
指纹去重、200 行 / 25 KB 硬限制、向量检索上限、反馈置信度、Dreaming 去重/衰减/Wiki
以及 Worker 只读记忆工具；`tests/memory-api.test.mjs` 覆盖完整记忆 API 闭环；
通用协作层由 `tests/collaboration.test.mjs`（DAG 顺序、并行、交接、失败隔离、循环/上限拒绝）和 `tests/router-collaboration.test.mjs`（Router 升级协作组且仅统一汇报一次）覆盖；专项协同模块另有 `tests/mas-graph.test.mjs`（编排引擎：依赖顺序、同层并发、条件边跳过、
重试、abort/continue 策略、超时、成环拒绝、取消）与 `tests/mas-pipeline.test.mjs`
（端到端落盘、写作失败路径、取消路径，均用 stub 模型与 stub 检索，不联网）。

注：`tests/appcontainer.test.mjs` 依赖 Windows AppContainer 的实际配置，在部分机器上
provisioning 会超过 60 秒而失败，属环境相关的既有用例，与协同模块无关。

## 目录

```
runtime/            随包分发的 Node.js 运行时
bin/                Node 入口
src/core/           Router、会话库、记忆、沙箱、Watchdog
src/core/mas/       多智能体协同：StateGraph 编排、检索工具、角色提示词、流水线
src/core/embed/     本地/远端嵌入
src/core/watchers/  文件监听服务
src/worker/         一次性 Worker 与工具
src/server/         HTTP + SSE
src/web/            桌面 UI
src-tauri/          Tauri 2 桌面壳（Rust）
sandbox/windows/    Job Object / AppContainer 宿主源码 (C#)
native/             编译产物 WindowsSandboxHost.exe
scripts/            构建与启动脚本
```
## 开源、发布与隐私

- **License**：源码采用 [MIT License](LICENSE)；第三方组件和构建工具的权利声明见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
- **源码与安装包分离**：源码仓库只保存可构建文件和文档；NSIS/MSI 安装包作为 GitHub Release 附件发布，校验和与发布步骤见 [RELEASE.md](RELEASE.md)。
- **本地数据**：安装版默认把会话、任务、长期记忆、日志和设置保存在 `%LOCALAPPDATA%\LocalSuperAgent`。升级或卸载不应静默删除这些数据；需要彻底清理时请先备份并手动删除。
- **网络边界**：默认无遥测。只有配置的 OpenAI 兼容模型端点、MAS 检索目标以及构建期下载工具会收到网络请求；模型端点会收到该次请求所需的提示词和任务上下文。详见 [PRIVACY.md](PRIVACY.md)。
- **参与贡献**：开发环境、测试命令和安全约束见 [CONTRIBUTING.md](CONTRIBUTING.md)，行为规范见 [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)。
- **长期记忆**：实现细节和验收边界见 [docs/long-term-memory.md](docs/long-term-memory.md)。
