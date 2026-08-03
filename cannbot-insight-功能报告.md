  # `cannbot-insight` 功能与新增内容报告

| 项 | 值 |
|---|---|
| 工具定位 | **CANNBot-Insight** — LLM 编码 Agent 的**会话级可观测性工具**（Session-level observability） |
| 技术栈 | Next.js 16 (App Router) + Prisma 6 + SQLite + better-sqlite3 |
| 仓库位置 | `external/cannbot-skills/plugins-community/cannbot-insight/` |
| 并入 cannbot-skills | 2026-07-03（`9baab2b`，293 文件 / 62275 行整体引入） |
| 最近一次改动 | 2026-07-16（`591d650`，CLI 参数处理加固） |
| 版本范围 | v0.01 (2025-05-24) → v1.03 (2026-07-13)，UI 版本记于 `src/lib/version.ts` |
| 本次 cannbot-skills pull | **无变化**（`514afb0..9e349d5` 未触及） |

---

## 一、它是什么

CANNBot-Insight 把 **opencode / Claude Code 的会话日志**（opencode `sessions.db` 或 Claude JSONL）导入本地 SQLite，做**turn 级**的细粒度拆解与可视化分析：token 五项拆解、上下文窗口追踪、subagent 层级、skill 事件、工具调用链路、workflow 阶段切分等。支持 **Web UI / TUI / CLI** 三种前端，共享同一套 15 个 `/api/observe/*` 后端端点。

> 注：仿真平台 `feat/auditrules` 分支的 `audit/file_restore.py` 即移植自本工具的 `file-restore.ts`。

---

## 二、核心架构

### 数据流

```
opencode sessions.db / Claude .jsonl（外部）
  → better-sqlite3 读取
  → adapter（opencode-db / claude-jsonl）归一化
  → turn-split → bridge-builder → execution-split
  → data-service 编排
  → Prisma 写入（8 个模型，SQLite）
```

- **多适配器架构**：`opencode-db`（SQLite）+ `claude-jsonl`（JSONL），注册表 `src/lib/ingest/adapters/index.ts`。
- **8 个 Prisma 模型**：Session / Turn / ToolCall / SkillEvent / Bridge / Execution / ExecutionSkill / SessionSkill。

### 三种前端模式（纯 API 客户端，共享后端）

| 模式 | 入口 | 渲染 |
|---|---|---|
| **Web UI** | `src/app/page.tsx`（9 Tab 详情页） | Next.js + shadcn/ui v4 + Tailwind v4 + @base-ui/react |
| **TUI** | `src/cli/tui/App.tsx` | Ink 7 + React 19（ESM strict，组件全自建） |
| **CLI** | `src/cli/index.ts`（Commander） | chalk + string-width + cli-truncate |

### Session 详情 9 个 Tab

Workflow ✦（阶段切分）/ Trace 🔍（概念溯源）/ Context 📊（上下文追踪）/ Interactions / Turns / Subagents / Tools / Skills / Overview。

---

## 三、功能演进（"新加了什么"——按阶段归纳）

### 阶段 0：初始版本 v0.01（2025-05-24）
opencode DB 导入、Turn 级存储、Token 五项拆解、上下文治理、Subagent 追踪、Skill 事件、交互链路图、Web UI 9 Tab。

### 阶段 1：分析视图奠基 v0.09 ~ v0.15（2025-05 ~ 06）
- **Workflow ✦ Tab** — phase-split 核心算法 + WorkflowTreeView（v0.09）
- **Interactions** 页面重构 + 时间轴（v0.12）
- **Trace 🔍 Tab** — 概念溯源追踪：关键词搜索、传播链路、DAG 图（v0.13）
- **Context 📊 Tab** — 上下文追踪，按 subagent session 展示增长曲线（v0.15）
- **AI Workflow (beta)** — LLM 驱动的 workflow 分析（v0.15）

### 阶段 2：多源 + 对比 v0.20 ~ v0.30（2025-06）
- **Claude Code JSONL 适配器** + 多适配器架构（opencode-db + claude-jsonl）（v0.20）
- **Session Compare** — 首页选两个 session 对比 token/费用/耗时/工具调用/subagent（v0.30）

### 阶段 3：CLI/TUI 全栈对齐 v0.32 ~ v0.36（2026-06）
- **CLI 地基** — Commander 入口、API client（15 端点）、类型、hooks（v0.32）
- **CLI 核心命令** — sessions/session/turn/search/compare/stats/import/delete/config（v0.33）
- **TUI 交互模式** — Ink + React，五屏（SessionList/SessionDetail/TurnDetail/CompareView/ImportPanel）+ 自建组件（v0.34）
- **analyze 命令** — AI workflow 分析（`--base-url/--api-key/--model/--json`）；`start.sh -c` 后台起后端（v0.36）

### 阶段 4：导入性能大优化 v0.37 ~ v0.41（2026-06-15）
- **批量写入** — `create` → `createMany` + `$transaction`，单 session **35–100×** 加速；batchCreateMany 分批 500 条/批防 SQLite 变量溢出（v0.37）
- **导入事务超时**提升 5s→60s（v0.39）
- **Web 目录浏览器** + `/api/ingest/browse-directory` + CLI FilePicker（v0.38）
- **opencode-db N+1 优化** — listSessions ~200+ 查询→5-6 次；readSession N×3→3 次（v0.41）
- Tool Calls 卡片显示 token 消耗（total/in/out + latency）

### 阶段 5：Turn 级 & Workflow 深化 v0.42 ~ v0.51（2026-06）
- **Session Compare Turn-by-Turn** — 每轮左右对比完整 prompt/输入/输出/thinking（v0.42）
- **Workflow 准备阶段**（phaseIndex=0）+ WorkflowTurnNode 纯文本 turn 节点（v0.43）
- Step node 展开 subagent turns；Turns tab 独立滚动（v0.44/0.45）
- **TurnTimeline 重设计** — root turns 主线 + subagent 并行车道（v0.51）

### 阶段 6：上下文可视化重构 v0.54 ~ v0.67（2026-06）
- WorkflowTreeView 重设计 + TurnContextPanel 组件（v0.54）
- **Context Replay 动画** — subagent spawn/death 标记可视化（v0.55）
- Token 拆分为 Tokens + Cache Read 两卡；Workflow 右侧 Context panel（v0.56）
- **Context panel 双指标条** — start→end pct（上下文窗口起止占比）（v0.61）
- **Import 性能再优化** 23.5s → 1.6s（batch write + 并发）（v0.60）
- **System (hidden) 稳定值** — 从首轮 assistant turn 计算，per-agent 独立（v0.67）

### 阶段 7：导出 & 多源并存 v0.63（2026-06-16，里程碑）
- **Session 导出**为独立 SQLite（`cannbot_session_<taskId>.db`），Web 按钮 + CLI `cbin export`
- **CANNBot Insight 导入适配器** — 导出的 SQLite 可回导（`import --source cannbot-insight`）
- **复合唯一键** `@@unique([taskId, framework])` — 同一 session 不同源（opencode-db vs cannbot-insight）并存两条
- **Agent Timeline** 图表 — token heatmap + 点击导航 Turn
- observe API 全端点新增 `framework` 参数；session 端点新增 `sessionId` 参数

### 阶段 8：上下文计算精准化 v0.65 ~ v0.76（2026-06）
- `contextWindowPct` / `inputMessagesTokens` **纳入 cacheReadTokens**（真实 LLM 输入）（v0.65）
- `start.sh -f` 清 `.next` 缓存重编译（v0.65）
- **LlmContextView** 分离 visible messages 与 hidden system context（v0.66）
- **LLM Input 自动展开**（可见 token < 6000 全展开）；TokenBarChart 按上下文窗口显示 Input/Output/Tool Calls（v0.67）
- **inputMessagesJson 重建** — 含 prior assistant tool_call args+result，修复 8.1k context gap（v0.69）
- **Claude Code subagent 导入** — toolUseId→subagentSessionId 精确映射（v0.73）
- **费用估算**（JSONL 无 cost，从 token×定价估）+ **耗时推断**（timestamp 差值）（v0.73）
- **JSONL 流式行合并** — thinking/text/tool_use 多行合一，消除 context 0% 跳变（v0.74）
- **Input 统一** — Input = input + cacheRead + cacheWrite；Cached 单独黄色区段（v0.75/0.76）

### 阶段 9：上传 & 导入易用性 v0.95 ~ v0.96（2026-06-29）
- **CANNBay 上传对话框** — 预填模板（提交人/描述/问题/日志路径/备注），sourcePath 自动填充
- **CLI `upload --file <path>`** — 一步导入+上传，支持 opencode .db 和 Claude JSONL
- **源类型自动识别**（按扩展名选 adapter，免 `--source`）
- **多 session 交互式选择** + **交互式描述填写** + **后端自动管理**（未运行则起、传完则关）
- TUI SessionList 新增 `u` 键上传入口

### 阶段 10：稳定化 v1.03（2026-07-13）
- `start.sh -c` CLI 子命令逐项传递，避免未引用展开的词分割/路径名展开；附加参数用 `-- <cli-args...>` 显式传入

---

## 四、关键能力清单（按主题）

| 主题 | 能力 |
|---|---|
| **多源导入** | opencode SQLite、Claude Code JSONL、CANNBot-insight 自有 SQLite（导出回导）；源类型自动识别 |
| **Turn 级拆解** | turn-split + bridge-builder + execution-split，8 模型持久化 |
| **Token 分析** | 五项拆解（input/output/cacheRead/cacheWrite/reasoning）、按模型上下文窗口占比、费用估算、耗时推断 |
| **上下文追踪** | Context 📊 Tab 增长曲线、Context Replay 动画、start→end pct 双指标、per-agent System(hidden) 稳定值 |
| **Subagent** | 层级追踪、toolUseId→subagentSessionId 精确映射、并行车道时间线、Agent Timeline heatmap |
| **Workflow** | phase-split 阶段切分（含准备阶段）、WorkflowTreeView、AI Workflow 分析（beta） |
| **对比** | Session Compare（总览）+ Turn-by-Turn（逐轮 prompt/IO/thinking） |
| **CLI/TUI** | 10+ 命令、Ink TUI 五屏、与 Web UI 功能对齐 |
| **易用性** | 目录浏览器、交互式上传、后端自动管理、`start.sh` 一键起（自动端口/迁移/装依赖/开浏览器） |
| **性能** | 导入 batch write（35-100×）、opencode-db N+1 优化、23.5s→1.6s |

---

## 五、在 cannbot-skills 仓库中的两次提交

| 提交 | 日期 | 内容 |
|---|---|---|
| `9baab2b` | 2026-07-03 | **整体引入** cannbot-insight 工具（293 文件 / +62275 行） |
| `591d650` | 2026-07-16 | fix：`start.sh` CLI 参数处理加固（+ `tests/start-script.test.ts` 111 行测试、CHANGELOG/README/version 同步） |

> cannbot-insight 自身的开发历史（v0.01→v1.03）远早于并入 cannbot-skills 的时间——它是作为一个成熟工具整体迁移进来的，CHANGELOG 一并带入。本次 `cannbot-skills` 的 pull（`514afb0..9e349d5`）**未触及** cannbot-insight。

---

*本报告基于 cannbot-insight 的 `CHANGELOG.md`（v0.01→v1.03）与 `CLAUDE.md` 架构说明生成。*
