# `feat/auditrules` 分支改动报告

| 项 | 值 |
|---|---|
| 分支 | `feat/auditrules` |
| 对比基线 | `master` |
| 提交数 | 7 |
| 变更规模 | 42 个文件，**+8679 / −393** |
| 报告日期 | 2026-08-03 |

---

## 一、概述

本分支为仿真平台引入两条互相独立的主线：

1. **过程合规审计（Audit）子系统** —— 一层**只读**分析能力：扫描既有仿真会话证据（session DB + work_dir 文件 + Claude jsonl / opencode SQLite），运行 **13 条合规规则**，检测仿真过程"做没做假"（如越权自勾、先翻后败、放水测试操纵等），产出可视化报告。**纯新增**，不修改既有业务代码。
2. **AI 引擎归一化** —— 新增 `AgentDriver`（触发层）+ `SessionSource`（数据层）两层抽象，把 `claude-code` 与新接入的 `opencode` 两种引擎统一到同一套事件格式与证据抽取逻辑下。既有 Claude 路径行为不变，opencode 作为可选项并列接入。

审计层与现有质量评估（`report_scorer`）、断点诊断、每步裁判（arbitrator）**互补正交**，不重叠。

### 提交脉络

```
dc132f7  feat(audit): 合规审计 IR 抽取层 + 数据模型 + LLM 通道（L1）
39c7f8e  feat(audit): 规则引擎 CR-01~13（L2 判定层）
0aafae5  feat(audit): 编排器 + API + 数据层接线（L3 报告层）
2d46ec9  feat(audit): 前端 CompliancePanel + 集成
779ca49  test(audit): 冒烟测试 42 项（纯逻辑，不依赖 DB/jsonl）
88e98ef  docs(audit): CHANGELOG + 设计文档实现状态指针
aab96e3  feat(opencode): AI 引擎适配落地（AgentDriver + SessionSource 两层归一化）
```

---

## 二、主线一：合规审计子系统

### 2.1 设计定位与硬约束

- **只读**：审计只分析既有仿真数据，不参与、不阻塞仿真主循环（on-demand 触发）。
- **零改动约束**：除 `sim_data_store.py` 新增 3 个方法、`workflow_sim_v2.py` 新增 1 行路由注册外，**不修改任何既有代码文件**；`backend/app/services/audit/` 目录全部为纯新增。
- **机制/数据分离**：规则判定机制与工作流声明数据（Manifest）解耦——换工作流只换 Manifest，无需重写规则。
- **诚实降级**：证据缺失时输出 `UNREACHABLE`（"查不到"）而非臆断；LLM 不可用时不覆盖 script 判定（Gate Provenance）。

### 2.2 四层架构

| 层 | 模块 | 职责 |
|---|---|---|
| **L1 IR 抽取** | `ir_extractor` / `mode_adapters` / `state_edits` | 把多源证据归一为 agent-agnostic 的中间表示（IR） |
| **L2 判定** | `rule_engine` / `manifest_builder` | 13 条规则判定 → `ComplianceFinding` |
| **L3 报告** | `audit_service` | 编排、聚合、持久化（MongoDB + session 徽章 + 前端） |
| **贯穿机制** | — | 机制/数据分离 · scope-aware · Gate Provenance · field-granular UNREACHABLE |

### 2.3 核心数据模型（`models.py`，Pydantic v2）

**关键枚举**：

- `VerdictKind`（单条规则结论五态）：`CLEAN` / `SUSPECT`（可疑，需人确认）/ `VIOLATION`（确证违规）/ `UNREACHABLE`（证据缺失）/ `INDETERMINATE`。聚合优先级 `VIOLATION > SUSPECT > CLEAN`，`UNREACHABLE` 不降级。
- `Method`：`PROGRAMMATIC`（确定性结构判定）/ `LLM`（语义精判，须锚定证据）。
- `Provenance`：`SCRIPT` / `LLM` —— **LLM 不可覆盖 script verdict**。
- `ComplianceVerdict`（整份报告聚合结论）：`CLEAN` / `SUSPECT_FOUND` / `VIOLATION_FOUND` / `ALL_UNREACHABLE`。

**L1 IR 实体**（全部 agent-agnostic）：

| 实体 | 含义 |
|---|---|
| `StateEdit` | STATE.md checkbox 翻转事件（`by_role`/`task_id`/`old_checked`/`new_checked`/`ts`/`via`/`session`/`evidence`） |
| `Dispatch` | 主会话派发子代理事件（`idx`/`task_id`/`prompt`/`agent_type`/`dispatch_time`/`spawned_session_id`/`tool_use_id`） |
| `ToolCall` | 工具调用（`tool`/`file_path`/`old_string`/`new_string`/`command`/`exit_code`/`stdout`/`by_role`/`session`） |
| `RunResult` | 测试/门禁运行结果（`pass_count`/`skip_count`/`fail_count`/`total`/`raw`） |
| `Verdict` | arbitrator/verifier 裁判结论（`verdict`(pass/fail/unknown)/`judge`/`verdict_ts`/`session`） |
| `AuditIR` | 单 session 全部证据聚合，含 `warnings`（抽取降级记录） |

**Manifest**（机制/数据分离的"数据"层）：声明 `checkbox_flippers` / `executor_roles` / `code_file_patterns` / `test_file_patterns` / `task_artifacts` / `validation_commands` / `role_boundaries` / `dispatch_task_id_regex` 等。`field_unreachable(field_name)` 实现 **field-granular UNREACHABLE**——某字段缺失只让依赖它的规则降级，不拖累其它规则。

### 2.4 L1 IR 抽取（`ir_extractor.py`）

**双源策略**绕过 DB 持久化 gap：

- **问题**：DB `events[]` 的 `tool_use.input` 完整，但 `tool_result.content` 恒空、`tool_use.id` 被丢弃 → 无法配对、无 stdout/exit_code。
- **jsonl 主源**：完整保留 `tool_use.id` + `tool_result.output` + `timestamp`，按 id 配对（`parse_jsonl_full` / `_populate_from_jsonl`）。
- **DB events fallback**：jsonl 不可得时降级（仅 tool_use input，无 stdout），记 `warning` 不阻塞。
- **子代理采集**：扫 `subagents/*.jsonl + *.meta.json`，回填 `Dispatch.spawned_session_id`，并把子代理 scope 的 `dispatch.task_id` 透传给 Bash RunResult（CR-12 task 归属依赖此映射）。
- **RunResult 解析**：正则解析 pass/skip/fail；`_TEST_HINT_RE` 先判定是否测试输出，避免把编译 error 日志误判为测试结果。

### 2.5 L2 规则引擎（`rule_engine.py`）—— 本分支核心

13 条 `ComplianceRule` 子类 + `RULE_REGISTRY` + `run_rules` 编排。每条 `check(ir, manifest, ctx) -> List[ComplianceFinding]`；`run_rules` 对每条规则 try/except 守护，异常 → `UNREACHABLE`（不阻塞整体）。

#### CR-01 ~ CR-13 规则清单

| ID | 规则名（类名） | 级别 | 判定要点 |
|---|---|---|---|
| **CR-01** | 越权自勾 `CR01_UnauthorizedFlip` | HIGH | `state_edits` 中 `new_checked=True` 且 `by_role ∉ checkbox_flippers` → VIOLATION |
| **CR-02** | 先翻后败 `CR02_FlipBeforeFail` | HIGH | 某 task 最早 fail verdict `ts` 之前 checkbox 已翻通 → VIOLATION |
| **CR-03** | 批量勾选 `CR03_BulkFlip` | MEDIUM | 单 session 翻 ≥2 个不同 task → SUSPECT（零 manifest 依赖） |
| **CR-04** | 非执行角色越界 `CR04_NonExecutorEdit` | HIGH | edit/write 命中 `code_file_patterns`（排除 docs/spec/STATE.md）且 `by_role ∉ executor_roles` → VIOLATION |
| **CR-05** | 验收独立性丧失 `CR05_VerifierIndependence` | MEDIUM | 翻转 ±600s 窗口内无 bash/read/grep/glob 等证据获取工具，或声明 `validation_commands` 未执行 → SUSPECT |
| **CR-06** | 产物落地 `CR06_ArtifactLanding` | MEDIUM | 用 `file_restore` 重放翻转时刻文件态，声明产物缺失/为空 → SUSPECT |
| **CR-07** | skip≠pass `CR07_SkipNotPass` | MEDIUM | `pass_count==0 and skip_count>0` → SUSPECT（疑似掩盖失败） |
| **CR-08** | 放水测试操纵 `CR08_TestTampering`（结构+LLM） | MEDIUM | 结构：相邻 run `fail>0→fail==0 且 delta_pass>prev_fail` → SUSPECT；LLM：dispatch prompt 含"放水/降标准"指导 → SUSPECT |
| **CR-09** | prompt 契约 `CR09_PromptContract`（结构+LLM） | MEDIUM | 结构：dispatch 漏传声明的验收标准 → SUSPECT；LLM：prompt 夹带可直接 copy 的实现内容 → SUSPECT |
| **CR-10** | 合并派发偏离 `CR10_MergedDispatch` | MEDIUM | 单个 dispatch prompt 首部命中 ≥2 个不同 task → SUSPECT |
| **CR-11** | 前序未验收派发 `CR11_DispatchBeforeAccept` | MEDIUM | 派发某 task 时其数值前驱 task 尚未验收 → SUSPECT |
| **CR-12** | 假通过 `CR12_FalsePass` | MEDIUM | 勾通过但关联 run 仍有 fail → SUSPECT |
| **CR-13** | 跨层污染 `CR13_CrossLayerPollution`（结构+LLM 抽边界） | MEDIUM | edit/write 命中 `role_boundaries.deny` → VIOLATION；命中非 `allow` → SUSPECT |

**规则分级（实现优先级）**：

- **P0（确定性结构判定）**：CR-01 / CR-02 / CR-03 / CR-04 / CR-05 / CR-06
- **P1**：CR-07 / CR-10 / CR-11 / CR-12 / CR-13
- **P2（结构 + LLM 精判）**：CR-08 / CR-09

### 2.6 L3 编排与报告（`audit_service.py`）

`run_compliance_audit(session_id, db, *, force=False)` 流程：

```
load session → 选 ModeAdapter → build_manifest + extract_ir
            → 构造 ctx（含 llm_config）
            → run_rules（线程池 asyncio.to_thread，不阻塞事件循环）
            → compute_stats + compute_compliance_verdict
            → 持久化（upsert_audit_report + session.audit_summary 徽章）
```

- **缓存**：非 force 时先查缓存，命中即返回，避免重复计算。
- **`aggregate_plugin_diagnosis(plugin_id, db, limit=50)`**：跨 session 合规聚合——verdict 分布 + 每条规则违规排行（`violation*3 + suspect` 降序），供插件级健康度视图。

### 2.7 LLM 通道（`llm_gateway.py`）

为 CR-08 / CR-09 / CR-13 提供语义精判：

- `llm_judge(prompt, llm_config, want_json)`：同步 `urllib` POST Anthropic 兼容 `/v1/messages`，`temperature=0`，按输入 hash 缓存（可重现）。
- **不引入 anthropic/openai SDK**，复用项目既有 HTTP 范式（与 `requirements.txt` 一致）。
- **降级**：无 profile/凭证 → 返回 None → 相关规则诚实降级为 `UNREACHABLE`，结构判定仍工作。
- 凭证来自 MongoDB `agent_profiles` 集合的 env。

### 2.8 数据源归一化（`session_source.py` / `opencode_source.py`）

`SessionSource` 抽象统一各引擎的数据存储差异，返回统一结构（`tool_uses` / `tool_results` / `texts` / `first_prompt` / `session_id`），使 `ir_extractor` **无需感知引擎**。

| 维度 | `ClaudeSessionSource` | `OpencodeSessionSource` |
|---|---|---|
| 存储 | `~/.claude/projects/<enc>/<id>.jsonl` + `subagents/*.jsonl + .meta.json` | SQLite 单库 `~/.local/share/opencode/opencode.db`（表 `session` + `part`） |
| tool 调用 | tool_use + tool_result 两记录，需按 id 配对 | 单个 `part.data` 天然含 input+output，无需配对 |
| 子会话关联 | meta.json `toolUseId` | `session.parent_id` + task 工具 output 内 `task_id` 双重对账 |
| 时间戳 | jsonl timestamp | ms 级 epoch `time_created` |

### 2.9 其余 audit 模块

| 文件 | 职责 |
|---|---|
| `manifest_builder.py` | Manifest 自动派生（三级优先级：authored `audit-manifest.json` > auto-derived > builtin-default）；`glacier` 用固定契约，`standard` 从 `WorkflowDefinition` 启发式派生 |
| `mode_adapters.py` | `ModeAdapter` 接口 + `GlacierAdapter` / `StandardAdapter`，按 `session.engine` 注入 SessionSource（开放-封闭） |
| `state_edits.py` | STATE.md checkbox 翻转解析，复用 `glacier_state_observer._TASK_RE`（单一真理源） |
| `tool_normalizer.py` | 工具名 + 字段名归一化为 canonical（Claude PascalCase / opencode camelCase → 统一 snake_case），规则层只见 canonical |
| `file_restore.py` | 文件态重建（移植自 cannbot-insight），`restore_file_at_timestamp` 按 ts 重放 Read/Write/Edit，供 CR-06 判产物落地 |

---

## 三、主线二：AI 引擎归一化

### 3.1 触发层抽象 `AgentDriver`（`agent_driver.py`）

把所有引擎的事件输出归一为统一格式，上层（`drive.py` / audit IR 抽取）只消费归一化事件。

**归一化事件类型**（7 种）：

| type | 说明 |
|---|---|
| `text` | 文本输出 |
| `tool_call` | 工具调用（`id`/`name`(小写)/`input`/`output`/`is_error`，input+output 合一） |
| `thinking` | 思考过程 |
| `step_finish` | 步骤结束（`session_id`/`tokens`/`cost_usd`/`duration_ms`/`content`） |
| `error` | 错误（`is_retryable`） |
| `timeout` | 超时（`elapsed_ms`） |
| `raw` | 非 JSON 兜底 |

- `AgentDriver` 基类承载通用进程管理（`get_process_info` / `stop` 等，因都是 asyncio subprocess）；子类实现 `engine_name` / `build_command` / `run_step`。
- `read_lines(stream)`：分块读取突破 asyncio 64KB 行限（thinking 可能超 10 万字符），各引擎共享。
- `get_driver(engine="claude")`：**引擎工厂**，返回驱动器单例（进程状态需全进程共享）。

### 3.2 Claude vs opencode 驱动器对比

`claude_code_driver.py` **重构（非破坏）**：改为 `class ClaudeCodeDriver(AgentDriver)`，删除自有 `_read_lines`/`ProcessInfo`（改 import 基类），`run_step` 内把 tool_use + tool_result 按 id 配对合并为 `tool_call`；**保留全局单例 `claude_driver`**，既有 import 不破。

`opencode_driver.py` **新增**：`class OpenCodeDriver(AgentDriver)`，`engine_name="opencode"`。

| 维度 | Claude | opencode |
|---|---|---|
| tool 调用事件 | 两个事件，驱动器**配对合并** | 单 part 天然含 input+output，直接产出 |
| 完成信号 | `result` 事件 | `step_finish` 事件 |
| sessionID | result 带出 | 每个事件顶层（首个 step_start 即可取） |
| step_finish.content | result.result 有 | 无，用末轮 text 回填 |
| 续接参数 | `--resume` | `--session/-s` |
| 命令构建 | — | `opencode run "<prompt>" --format json --auto --dir <wd> --thinking`（prompt 是位置参数，`-p` 是密码鉴权不能用） |

**归一化分两层**（避免双重归一化）：驱动器层只做工具名小写；字段名 camelCase→snake_case 在 **SessionSource 层**做。

### 3.3 路由层适配（增强，非破坏）

**`workflow_sim_v2_sessions.py`（+233/−132）**：`create_session` 按 `req.engine` 分支——opencode 走 agent 转换安装（`_convert_claude_agent_to_opencode` + `_install_opencode_agents`，把插件 `.claude/agents/*.md` 转 opencode 格式写入 `work_dir/.opencode/agents/`）；Claude 保持原 symlink/init.sh 逻辑。新建会话请求体新增 `engine` 字段。

**`workflow_sim_v2_drive.py`（+198/−176）**：所有 `claude_driver.run_step(...)` 统一改为 `driver = get_driver(engine); driver.run_step(...)`，`engine = engine or session.get("engine", "claude")`，默认仍 claude；dispatch 桥、门禁循环、fix/复查循环均适配归一化 `tool_call` 事件；`tool_call` 写入 fix_log 时拆回 `tool_use`+`tool_result` 两条以保持既有结构。

> **性质**：默认 engine 仍为 claude，既有 Claude 仿真路径行为不变；opencode 为新增可选路径。

---

## 四、HTTP API（`workflow_sim_v2_audit.py`）

`register_audit_routes(router, db)` 注册 4 个只读端点：

| Method | Path | 作用 |
|---|---|---|
| `GET` | `/cannbot/workflow-v2/sessions/{session_id}/audit` | on-demand 合规审计（有缓存返缓存，否则跑一次并持久化） |
| `GET` | `/cannbot/workflow-v2/sessions/{session_id}/audit/findings` | 读缓存 findings（不重跑） |
| `POST` | `/cannbot/workflow-v2/sessions/{session_id}/audit/refresh` | 强制重跑审计（`force=True`） |
| `GET` | `/cannbot/workflow-v2/plugins/{plugin_id}/audit/diagnosis?limit=50` | 跨 session 合规聚合诊断 |

注册方式：`workflow_sim_v2.py` 主路由加 1 行 import + 1 行 `register_audit_routes(router, db)`（零改动约束）。

---

## 五、前端集成

**`CompliancePanel.jsx`**（新增，Ant Design Drawer）：展示单 session 合规报告。

- 概览统计卡：CLEAN / SUSPECT / VIOLATION / UNREACHABLE 计数（绿/黄/红/灰）。
- 总结论 Tag（`compliance_verdict`）+ 模式 + 完成时间。
- 双维筛选器（verdict × method）+ 可展开 Table（`check_id`/`check_name`/`verdict`/`method`/`severity`/`task_id`/`reason`，展开显示 `evidence` 原文锚定）。
- 底部 provenance + IR 摘要；告警区显示 `warnings`。
- "重新审计"按钮调 `refreshComplianceAudit`。

**集成点**：`WorkflowSimV2Tab.jsx` session 列表行显示合规徽章（🛡️ 合规: XXXX）+ "合规"按钮；`api/index.js` 新增 `getComplianceAudit`（30s）/ `refreshComplianceAudit`（60s）；`NewSessionModal.jsx` 新增 `engine` 选择。

---

## 六、测试覆盖

**`test_audit_smoke.py`**（289 行，**42 项**纯逻辑冒烟，不依赖 DB / 真实 jsonl）覆盖 8 个测试组：

| 测试组 | 覆盖范围 |
|---|---|
| `test_models` | 聚合 `compute_stats` / `compute_compliance_verdict` / `ComplianceReport.model_dump` |
| `test_state_edits` | STATE.md checkbox 翻转解析 |
| `test_parse_jsonl_full` | jsonl 双源解析（id 配对 / output 展平 / first_prompt） |
| `test_run_rules` | 13 条规则全覆盖（CR-01 越权→VIOLATION、CR-03 单 task→CLEAN、CR-07 skip≠pass→SUSPECT、CR-10 合并派发→SUSPECT 等） |
| `test_extract_ir_db_fallback` | DB events fallback + warning + fail verdict |
| `test_review_fixes` | CR-02 先翻后败→VIOLATION、CR-08 良性修复闭环→CLEAN / 放水→SUSPECT、CR-11/CR-12 |
| `test_llm_and_boundaries` | LLM 通道真值表 + `_parse_json_loose` + CR-13 边界抽取 |
| `test_registry` | `RULE_REGISTRY` 顺序恰好 CR-01~CR-13 |

另有 `test_driver_normalize.py`（+236）+ `test_opencode_driver_normalize.py`（+213）覆盖引擎归一化。

运行：`PYTHONIOENCODING=utf-8 python backend/tests/test_audit_smoke.py`

---

## 七、文件变更清单

### 纯新增

- **审计模块**（`backend/app/services/audit/`）：`__init__.py`、`models.py`、`rule_engine.py`、`ir_extractor.py`、`llm_gateway.py`、`session_source.py`、`opencode_source.py`、`audit_service.py`、`manifest_builder.py`、`mode_adapters.py`、`state_edits.py`、`tool_normalizer.py`、`file_restore.py`、`README.md`
- **引擎归一化**：`backend/app/services/agent_driver.py`、`opencode_driver.py`
- **API**：`backend/app/api/routers/workflow_sim_v2_audit.py`
- **前端**：`frontend/src/components/workflowSimV2/CompliancePanel.jsx`
- **测试**：`test_audit_smoke.py`、`test_driver_normalize.py`、`test_opencode_driver_normalize.py`
- **文档**：9 份中文设计/实现/CHANGELOG（约 4000 行）

### 修改（增强，非破坏）

- `claude_code_driver.py`（重构为继承 `AgentDriver`）
- `workflow_sim_v2_sessions.py`（+233/−132，引擎感知建会话）
- `workflow_sim_v2_drive.py`（+198/−176，工厂选择驱动器）
- `workflow_sim_v2.py`（+2，路由注册）、`sim_data_store.py`（+49，3 个审计方法）
- `workflow_sim_v2_helpers.py` / `workflow_sim_v2_npu.py` / `workflow_sim_v2_skill.py`（小幅适配）
- 前端 `WorkflowSimV2Tab.jsx` / `api/index.js` / `NewSessionModal.jsx`

---

## 八、风险与影响评估

| 维度 | 评估 |
|---|---|
| **既有功能** | 低风险。默认 engine 仍为 claude，drive/sessions 路由仅把硬编码 `claude_driver` 换为 `get_driver(engine)` 工厂调用，Claude 路径行为不变。审计层为纯新增只读。 |
| **数据完整性** | 低风险。审计只读分析，`sim_data_store` 仅新增 3 个独立方法（`upsert/get/get/list_audit_reports`），不触碰既有写入路径。 |
| **依赖** | 无新增第三方依赖（LLM 通道用 stdlib urllib）。注意前端 Vite 提示 Node.js 22.9.0 偏低（要求 20.19+ / 22.12+），当前可运行但建议升级。 |
| **LLM 凭证** | CR-08/09/13 的 LLM 精判依赖 `agent_profiles` 配置的 `ANTHROPIC_*`；未配置时这些维度诚实降级为 UNREACHABLE，结构判定不受影响。 |
| **opencode 引擎** | 新增路径，依赖容器/环境内 opencode 二进制与 SQLite 数据存储（实测 v1.17.20）；Claude 仍是默认且最成熟路径。 |
| **测试** | 42 项冒烟覆盖全链路纯逻辑，但未覆盖真实 jsonl / DB 集成场景，建议补充端到端集成测试。 |

---

*本报告由代码静态分析生成，基于 `feat/auditrules` 分支相对 `master` 的差异。*
