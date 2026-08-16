  # Executor 与 Judge 代码逻辑详解

> 基于 `developer_experience_agent` 仓库 `judge-refactor` 分支（2026-08-16 工作区状态，含当日优化）。
> 所有行号对应当前工作区文件。

---

## 目录

1. [总体调用链](#1-总体调用链)
2. [Executor：执行器](#2-executor执行器)
3. [Advisor：执行器的大脑](#3-advisor执行器的大脑)
4. [Judge：裁判节点](#4-judge裁判节点)
5. [触发策略 trigger_policy](#5-触发策略-trigger_policy)
6. [判定策略 doc_conformance.judge_decision](#6-判定策略-doc_conformancejudge_decision)
7. [痛点质控 faithfulness](#7-痛点质控-faithfulness)
8. [Pain Review 阶段与复现提交](#8-pain-review-阶段与复现提交)
9. [State 字段读写总表](#9-state-字段读写总表)
10. [Prompt 体系](#10-prompt-体系)
11. [日志落盘](#11-日志落盘)

---

## 1. 总体调用链

整个评估是 LangGraph 风格状态机，节点函数接收 `state: dict`、返回"变更字段"由框架合并：

```
planner ──> executor ──> tools ──┐
   ^            ^               │ last_tool_result
   │            └── judge ←─────┘（每个工具调用后复盘，pass/retry）
   │                │
   │                └─ retry_instruction 注入回 executor（advisor prompt 前置）
   │
   └─ 阶段任务队列 pending_tasks
        │ 阶段完成
        ▼
   evaluator（体验分析：observations + pain_points）
        ▼
   judge.pain_review（痛点质控：签名预拦截 → LLM 分类 → 归因校验 → 升级复核 → 去重批准）
        ▼
   repro（异步复现终审：资格 → 快照 → 计划 → 执行 → 四道闸门 → 终态）
        ▼
   reporter（报告 + 附录 D 审计）
```

三个 LLM 角色共用 `src/infra/llm.LLMAdapter`（HTTP 重试/退避、env 覆盖、token 统计），
实例经 `devx_main.setup_llm()` / `setup_judge_llm()`（可选独立配置 `JUDGE_LLM_*`，默认
temperature 0.1）依赖注入到各节点。

---

## 2. Executor：执行器

文件：`src/workers/executor/worker.py`（1686 行）

### 2.1 数据结构 `ExecutionResult`（行 35–60）

```python
@dataclass
field: task_id, success, output, error, duration_seconds,
       tool_request,            # dict | None：下一工具调用请求（None=任务已结束）
       observations, pain_points, no_issue_to_report, content,
       parse_failed,            # LLM 输出解析全部失败
       search_hit, hit_rank, matched_url, match_reason,     # discovery 命中
       direct_hit, direct_hit_rank, indirect_hit, indirect_hit_rank
```

### 2.2 `ExecutorWorker`（行 63–763）

构造（行 70–87）：`__init__(llm, tool_registry, skill_loader)` → 内部创建
`self.advisor = TaskAdvisor(llm, skill_loader, tool_registry)`。

`execute_task(task, context)`（行 89–147）四路分支（优先级从上到下）：

| # | 条件 | 去向 |
|---|---|---|
| 1 | `_maybe_handle_discovery_search_without_llm` 命中（行 120） | discovery 搜索任务固定关键词驱动，**不调 LLM** |
| 2 | `context["tool_result_context"]` 非空（行 123） | `_handle_tool_result`：advisor 分析结果决定下一步 |
| 3 | `self.llm` 存在（行 126） | `_plan_with_judge`：advisor 规划首个工具调用 |
| 4 | else | `_execute_directly`：无 LLM 时按 task_type 映射工具 |

异常兜底（行 139–147）：任何异常 → `ExecutionResult(success=False)`。

### 2.3 节点主流程 `executor_node`（create_executor_node 行 883–1669）

1. **取任务**（行 887–898）：`pending_tasks[0]` → `TaskStep`。
2. **旧状态迁移**（行 901–916）：`task_retry_counts → phase_retry_state`、
   `task_retry_triggered → pending_retry_request`。
3. **工具结果判定**（行 918–931）：`last_tool_result.task_step_id == task.step_id` 时
   `has_tool_result=True`；若 `task_retry_active.status=="pending"`，调
   `classify_retry_outcome` 计算任务重跑结果。
4. **搜索历史**（行 932–964）：仅 `web_search` 成功时记 `search_history[step_id]`，
   同 query 最多一次（blocked 豁免两条）。
5. **Context 构建**（行 966–1103）：current_phase/role/project/environment/
   target_branch/tool_result_context/last_tool_result/evaluation_focus/
   prev_task_last_command/search_history/task_tool_history/prior_failures_hint/
   contrib_*_findings/**judge_retry_instruction/judge_lessons**/state。
   其中 `judge_lessons`（行 1074–1082）从 `state["judge_history"]` 提取**除当前任务外**
   的 lesson，拼 `- tool: lesson`，取尾 1500 字符——Reflexion 式跨任务教训记忆。
6. **执行**（行 1121）：`worker.execute_task(task, context)`。
7. **后处理**：
   - `tool_request` 非空（行 1140–1260）：写
     `updates["pending_tool_call"] = {**tool_request, "task_step_id": ...}`，
     **不弹出任务**，等 tools 节点执行后下一轮回来。
   - `tool_request` 为 None（行 1262 起，任务终局）：
     - 弹出 `pending_tasks[0]`；
     - contrib_discover/contrib_search 结论持久化（行 1266–1301）；
     - `no_issue_to_report=True` → 后续 contribution 任务全部转
       `skipped_tasks(skip_type="no_issue_to_report")`（行 1304–1337）；
     - **任务级重跑判定**（行 1346–1504，见 2.4）；
     - **早退判定**（行 1513–1599，见 2.5）；
     - 任务切换清理（行 1596–1604）：`judge_retry_count=0`、
       `judge_retry_instruction=None`、清当前任务 `tool_call_counts/search_history/
       task_tool_history`（**judge_history 保留**）；
     - `all_execution_results` 追加（上限 50 条，行 1606–1611）。
   - **阶段重试消费**（行 1613–1663）：幂等校验（retry_id vs last_consumed_retry_id）
     + TTL 保护（超时→early_exit），消费后 `judge_stage` 回切 `"execution"`。

### 2.4 任务级重跑（行 1346–1504）

触发条件（行 1390–1400，全部同时满足）：

```python
not task_achieved                    # 任务未达成
and task_retry.enabled and max>0     # 配置启用
and retry_count < max                # 预算未耗尽（phase_retry_state[phase]）
and phase in on_phases               # 阶段白名单
and phase != "discovery"             # discovery 不重跑
and not _repeat_root_cause           # 失败指纹去重：连续两次相同指纹不重跑
and _tool_calls_ok                   # 巻具调用数死循环保护
```

关键数据：
- `phase_retry_state`：`{phase: {used, max, last_retry_id, status}}` 阶段预算。
- `task_failure_fingerprints`：`{task_id: [指纹…]}`，连续同指纹 → `_repeat_root_cause`。
- `task_retry_active`：重跑开始写 `{retry_id, task_id, phase, attempt, status:"pending",
  before_tool}`；结束由 `classify_retry_outcome` 判定 `resolved/partial/ineffective`
  （比较 tool_fingerprint SHA256 与失败类型是否变化，纯确定性不调 LLM），
  写入 `task_retry_history`（保留 50 条）。

重跑流程（行 1418–1504）：任务重新入队头 → 清当前任务历史 → 阶段预算 +1 →
`_analyze_failure_root_cause`（行 675–718，LLM 根因分析，60s 超时兜底）拼成
`_retry_instruction` 写入 `state["judge_retry_instruction"]` → 下一轮 advisor 前置注入。

### 2.5 早退 early_exit（行 1513–1599）

条件：任务未达成 + 非 discovery + 有剩余任务（若任务重跑已启用但超限/不在白名单，
也落到这里）。

行为：
1. 剩余任务分组：非 contribution 阶段时 `contrib_tasks`（保留）与
   `non_contrib_tasks`（跳过）；contribution 阶段自身失败则全部跳过（防循环）。
2. `phase_status`：当前阶段 `failed`，中间阶段 `skipped`；写 `early_exit_reason`。
3. `non_contrib_tasks` 全部 → `skipped_tasks(skip_type="early_exit")`。
4. 有 contrib_tasks → 继续贡献阶段；无 → `early_exit=True` 直接进报告。

> 已知问题（待优化项 12）：早退不区分失败归因——AGENT/ENVIRONMENT 责任的可恢复
> 失败（如装错依赖）与 PROJECT 责任的硬失败同等对待，一次重试无效即砍掉后续阶段
> （0815 批 pyasc/hccl 丢 testing 阶段、分数失真 20 分的根因）。

### 2.6 工具执行边界

executor **不直接执行工具**：只产出 `tool_request`，由 LangGraph tools 节点执行
（沙箱/超时在 tool_registry 与 tools 节点内实现），结果写回
`state["last_tool_result"]`（tool/args/success/result/error/task_step_id/
tool_metadata）并追加 `task_tool_history[step_id]`（含 result_for_prompt）。

---

## 3. Advisor：执行器的大脑

文件：`src/workers/executor/advisor.py`（1449 行）

### 3.1 构造（行 87–96）

`TaskAdvisor(llm, skill_loader, tool_registry)`；缓存字段：`_skill_context`、
`_tools_schema`（懒加载 OpenAI tools schema，注入必填 `thought` 字段，非 discovery
阶段过滤 `web_search`，行 137–188）、`_judge_feedback_instruction`、
`_judge_feedback_lessons`。

返回类型 `AdviceResult`（行 57–81）：`need_tool_call/tool_request/task_completed/
success/raw_output/observations/pain_points/no_issue_to_report/content/parse_failed/
search_hit…`。

### 3.2 `plan_tool_call`（行 190–326）——规划下一工具调用

1. guard：无 llm → 失败 AdviceResult。
2. skill 上下文（discovery 返回空）。
3. 拼上一任务最后命令提示（PREV_TASK_LAST_COMMAND_HINT）。
4. 拼任务完成指标（success/failure indicators）。
5. discovery 任务描述追加固定关键词白名单（`_build_discovery_task_description`）。
6. **按阶段选 prompt 模板**：discovery / contrib_discover / contrib_search /
   contrib_submit / 通用（TASK_ANALYSIS_PROMPT*），占位符统一
   `preamble/task_description/task_indicators/important_reminders`。
7. `_invoke_and_parse(valid_tool_names=…)`。

### 3.3 `handle_tool_result`（行 328–621）——工具结果分析 + 任务完成判定

1. prior_history = `tool_history[:-1]`（当前结果单独展示）。
2. discovery 阶段把 `expected_repo_url` 追加进指标（含"严禁用于构造搜索词"约束）。
3. 当前结果段格式化：成功/失败两态（失败优先展示 result，error 截 300 字符）；
   web_search 追加结构化结果段（前 10 条 rank/title/url/snippet）；
   discovery 追加首页命中分析段。
4. **Context 超限重试（滑动窗口 + ContextPack 降级，行 536–616）**：
   - 有 fact_ledger：两轮尝试 full → compact（`build_executor_context(state, phase,
     budget, focus, exclude_last_n)`，compact 聚焦最近失败+当前任务）；
   - 无 ledger 兜底：最多 4 轮，指数退避删历史（`drop = min(2^attempt, count)`
     保留尾部），清空仍超限 → 强制任务失败；
   - 仅 `_is_context_overflow_error` / `_is_timeout_error` 触发重试，其他异常上抛。
5. 兜底：advice 为 None → 失败 AdviceResult。

### 3.4 `_invoke_and_parse`（行 656–782）——两级解析重试

循环最多 `max_retries+1`（默认 4 次）：

| 情形 | 处理 |
|---|---|
| `task_completed=True` | 直接返回 |
| `need_tool_call=True` 且工具名在白名单 | 返回 |
| 工具名不在白名单 | 追加 `_INVALID_TOOL_HINT`（可用工具列表）重试 |
| 两者皆 False（格式不符） | 追加 `_FORMAT_CORRECTION_HINT` 重试 |
| context overflow | `reraise_overflow=True` 上抛给滑动窗口；否则转 `task_completed+success=False` |
| 重试耗尽 | `parse_failed=True`；若是工具名无效，强制清除 `need_tool_call` 防执行无效工具 |

### 3.5 `_invoke_llm`（行 784–864）——实际调用 + 截断

1. **judge 反馈前置注入**（行 793–799）：`_judge_feedback_instruction/lessons` 任一
   非空 → `inject_judge_feedback(prompt, …)` **置于 prompt 最前**（截断之前，保证不被裁）。
2. **Prompt 截断（保头尾压中间，行 801–821）**：超 `_PROMPT_SAFE_LIMIT=96_000` 字符时
   保头 32_000 + 尾 63_880（尾部有格式要求/judge 建议/任务目标，不能截）。
3. 调用：`[SystemMessage(EXECUTOR_SYSTEM_PROMPT), HumanMessage(prompt)]` +
   `llm.invoke(messages, tools=tools_schema)`（native function calling）。
4. token 统计回写 `state["total_tokens"]`。

### 3.6 响应解析三级降级 `_parse_response`（行 866–979）

1. **Native function calling**（`response.tool_calls`，兼容旧 additional_kwargs）→
   `need_tool_call=True`；thought 从 `args["thought"]→content→兜底` 三级提取后移出 args。
2. **文本工具调用**：`extract_tool_call(text)`。
3. **任务结论 JSON**：`extract_execution_result(text)`：
   `{"status": "success"|"failure", "no_issue_to_report", "content", "search_hit",
   "hit_rank", "matched_url", "match_reason"}`；`partial` → 未完成（解析 fallback）。

### 3.7 工具历史裁剪 `_build_tool_history_text`（行 1216–1274）

- 最近 6 条（`_RECENT_FULL_KEEP`）完整保留；
- 更早的成功调用摘要化（结果截 120 字符 + `[已摘要]`）；
- **失败调用全量保留**（命令+exit_code+error+输出；输出超 2000 字符保首 1200+尾 800）。

### 3.8 关键常量

| 常量 | 值 | 用途 |
|---|---|---|
| `_PROMPT_SAFE_LIMIT` | 96_000 | prompt 字符上限 |
| `_RECENT_FULL_KEEP` | 6 | 完整保留的近期历史条数 |
| `_FAILURE_OUT_THRESHOLD/_HEAD/_TAIL` | 2000/1200/800 | 失败输出截断 |
| `_HISTORY_RESULT_LIMIT` | 0（禁用） | 历史二级截断 |
| `_DISCOVERY_DEFAULT_KEYWORDS` | doc/repo 两桶 | 固定搜索关键词模板 |

---

## 4. Judge：裁判节点

文件：`src/workflow/nodes.py` `create_judge_node`（行 593–2146）

### 4.1 闭包状态

| 变量 | 用途 |
|---|---|
| `_judge_logger` | JudgeLogger（异步包装），**惰性初始化**（首评时取 project.name/run_id） |
| `_repro_dispatchers` | `{run_id: ReproDispatcher}` 复现调度器缓存 |
| `_judged_call_keys` | `{task_step_id: {duplicate_key}}` 同任务判定去重（跨任务清 stale） |
| `_skill_loaded/_skill_source` | skill 加载诊断 |
| `worker` | judge strategy 实例（默认 doc_conformance） |

### 4.2 入口分流（行 1747–1756）

```python
judge_node(state):
  if state["judge_stage"] == "pain":  → _review_pains(state)   # 痛点质控阶段
  else:                              → 执行复盘（本节）
```

### 4.3 执行复盘完整链路

```
① trigger 判定（行 1783–1791）
② skip_phases 检查（行 1793–1802）：只跳普通/抽样评判，强制理由（失败/证据/归因/危险）
   不可跳
③ should_judge=False → 返回 {}（行 1803–1808）
④ 去重键登记（行 1810–1816）：_judged_call_keys.setdefault(task).add(duplicate_key)
   并清理其它任务的键集
⑤ retry 闭环检测（行 1828–1836）：judge_retry_active pending 且 task 匹配 →
   classify_retry_outcome(before_tool, last_tool_result) → 写 judge_retry_history、
   清 judge_retry_active
⑥ 上下文构造（行 1839–1882）：
   - doc_evidence：_extract_doc_evidence(task_tool_history)——file_read/browser_content/
     browser_navigate 的读取结果
   - tool_history_text：_format_history_for_judge 序列化（序号.tool 成功/失败 args）
   - judge_history_text：前序判定 reason 摘要（每条截 100 字符，总截 judge_history_max=2000）
   - context：项目/路径/分支/运行环境
⑦ skill 加载（行 1851–1862）：worker.get_skill_body()（JUDGE_SKILL.md，改文件即改行为）
⑧ strategy.judge_decision(ctx)（行 1885–1900）→ verdict
⑨ 连续失败兜底（行 1902–1921）：同任务连续失败 ≥ repeat_fail.threshold(4) →
   强制 correct=False（防死胡同）
⑩ action 分发（见 4.4）
⑪ 落盘 _log_judge（行 788–827 → judge_logger.record）
⑫ judge_history 追加（行 1955–1963，保留 50 条）
```

### 4.4 Action 分发

**Pass（verdict.correct=True，行 1970–1999）**
- `refact_advice` 非空 → 写 `judge_retry_instruction`（标注"非强制优化建议"，
  **不加 retry_count**）；
- `result_anomaly=True`（伪成功/真失败被放行）→ 同通道温和提醒；
- `judge_retry_contract=None`。

**Retry（correct=False 且 retry_count < max_retries，行 2001–2118）**

retry_instruction 四段结构（行 2010–2024）：
1. 被否决操作（工具+参数+结果+错误）；
2. 否决原因（verdict.reason）；
3. 禁止事项：禁止同 tool+args 重跑 / 禁止换工具保留错误参数 / 参数值逐字复制；
4. 修正方向（verdict.retry_suggestion）。

安装脚本确定性重试合同（行 2028–2068）：`has_install_skip_signal`（交互跳过类伪成功）
时提取 bounded_answer → `build_bounded_retry_command` 生成确定性命令（printf 单次输入，
禁无界 `yes |`）；无有界答案 → `retry_blocked`，不设 judge_retry_active（防误消费）。

State 更新：`last_tool_result=None`（强制 executor 重新规划）、
`judge_retry_instruction`、`judge_retry_contract`、`judge_retry_count+1`、
`judge_retry_active={retry_id, task_id, attempt, status:"pending", before_tool}`。

**强制放行（retry_count ≥ max_retries，行 2119–2131）**：清 instruction/contract，
`metrics["judge"]["force_pass"] += 1`。

**Retry 闭环**：打回后 executor 执行的第一条新工具结果回到 judge 时（步骤⑤），
`classify_retry_outcome` 确定性比较 before/after（工具指纹 SHA256 + 失败类型）：
`resolved`（变了且成功）/ `partial`（变了仍败但类型变）/ `ineffective`（原样重复）。

---

## 5. 触发策略 trigger_policy

文件：`src/workers/judge/trigger_policy.py`（231 行），纯函数可测。

`should_judge_execution(tool_result, cfg, *, task_failed, task_history, judged_keys)`
（行 169）按序判定：

1. **失败调用** → `tool_failed`（必审，**不走去重**——同命令结果翻转本身是信号）；
2. **重复短路**（行 194–196）：非失败且 `duplicate_key ∈ judged_keys` →
   `(False, ("duplicate_call",))`。`duplicate_key`（行 148–166）=
   `tool|md5(规范化 cmd/command/path/file_path/url/query)[:12]`；
3. 任务未达成 → `task_failed_full_history` + `task_history_available`；
4. `always_judge_tools`（git_clone/browser_navigate/browser_content）；
5. `dangerous_patterns` 正则；
6. `classify_command ∈ {destructive, state_changing, build_or_run}`；
7. 证据类工具读文档失败/搜索无结果 → `affects_evidence`；
8. `which/--version/npu-smi/make/python/cmake` 等特征 → `affects_attribution`。

任一命中 → 必审。否则：只读工具 `safe_read_only` 放行；其余按 `sample_rate` 抽样。

命令分类（行 76–89）优先级：destructive > state_changing > read_only >
build_or_run/evidence_producing > unknown。

---

## 6. 判定策略 doc_conformance.judge_decision

文件：`src/workers/judge/strategies/doc_conformance.py`（行 277–425）

### 6.1 上下文构造

- `doc_evidence`：节点层提取的 agent 已读文档原文；
- `tool_signal_summary`（行 293–294）：`_compute_tool_signal`（行 107–208）**零成本
  客观信号**——退出状态 + 安装跳过信号 + 输出行数/耗时 + error/warning/fatal 关键词
  频率 + 错误行预览（前 3 行），附系统失败分类 `classify_failure`；
- prompt 模板 `JUDGE_DECISION_PROMPT` 14 个占位符 + 条件段（judge_history_section、
  skill_section）。

### 6.2 `_llm_judge_loop`（行 323–425）

- 消息：`[SystemMessage(sys_prompt), HumanMessage(prompt)]`；
- ThreadPoolExecutor 超时兜底（超时 → `_pass_verdict_obj(timeout=True)` 保守放行）；
- JSON 解析失败 → 追加 `_FORMAT_RETRY_HINT` 重试（默认 2 次）；
- HTTP 4xx/5xx → 直接放行；全部失败 → `_pass_verdict_obj()`；
- 解析后处理：refact_advice 仅配置启用时采纳；responsibility 归一化枚举；
  `aggregate_dimensions` 按 cfg.dimensions 聚合；`_annotate_result_anomaly`（行
  427–446）标注 failure_class，伪成功/真项目失败 → `result_anomaly=True`
  （**不改 correct**）。

### 6.3 correct 与 responsibility 的关系

**相互独立**：responsibility 是归因标签（PROJECT/AGENT/ENVIRONMENT/USER_INPUT/
UNKNOWN），不影响是否打回。语义注意：execution_review 的 PROJECT=操作正常（正向）；
pain_review 的 project=项目缺陷（负向），**勿混用**。

---

## 7. 痛点质控 faithfulness

文件：`src/workers/judge/faithfulness.py`（343 行）

### 7.1 `review_pain_by_claims`（行 170–343）

入口降级：无 LLM → `_pass("no_llm")`；空内容 → `_pass("empty_input")`。

prompt 7 占位符：task_desc/pain_content/task_summary/judge_evidence/doc_evidence/
knowledge_anchors/runtime_context。

调用重试：最多 2 次；超时→`_pass("timeout")`；HTTP→`_pass("http_error")`；
解析失败重试一次→`_pass("parse_failed")`（保留 raw_response 5000 字符）。

claims 规整（行 299–309）：`{claim, supported(True/False/"partial"), reason,
source}`，source 优先级 source > source_ref > evidence_ref。

**硬绑定**（`_normalize_pain_decision` 行 148–167）：
- `agent_fault/env_issue/unsupported/unverified` → 强制 `is_valid=False`
- `none/inaccurate` → 强制 `is_valid=True`

### 7.2 `validate_pain_attribution`（行 56–127）——排除性归因证据闸

仅对 `_EXCLUSION_TYPES = {agent_fault, env_issue, unsupported}` 生效。

两个必要条件**同时满足**才允许剔除：
1. ≥1 条受支持 claim（supported=True/"partial" 且 claim/reason 非空）；
2. ≥1 个来源引用——claim 级（source/source_ref/evidence_ref/tool_id/tool_ids）
   **或**继承自 evaluator 结构化引用（pain.tool_id/tool_ids/input_refs/
   matched_call_ordinals/source_call_ids）。

不满足 → 强制转 `unverified` + `unverified_cause="insufficient_attribution_evidence"`，
`attribution_validation` 落盘四个计数。`record` 本身不算来源（仅审计提示）。

---

## 8. Pain Review 阶段与复现提交

### 8.1 评审链（`_review_pains`）

```
evaluator 提取 pain_points（三条件：执行结果支撑 + 根源在文档/项目 + 体验断点）
  ↓
⓪ 签名预拦截（doc_conformance._kb_signature_prefilter）
   命中人工策划签名（taskId 精确 + allOf/anyOf 关键词，旁挂 <kb>.signatures.json）
   → 直接 discard + environment 责任 + 审计标记 kb_signature_prefilter，不耗 LLM
  ↓
① 逐条/批量 LLM 评审（review_pain_points / review_pain_points_batch）
   → review_pain_by_claims + validate_pain_attribution
  ↓
② 证据升级复核（nodes.py 行 1016–1102）：unverified+unknown 但
   _actionable_failure_evidence 有关联失败证据 → 带 claim_type/序号/签名重审一次；
   通过条件 keep|revise 且 responsibility=project
  ↓
③ 批准过滤（行 1167–1203）：decision∈{keep,revise} + responsibility=project +
   evidence 非空 → approved（pain_id=_pain_id(run_id,phase,task,original)，
   repro_case_id=make_repro_case_id(run_id,pain_id)）
  ↓
④ 同任务去重（行 1171–1211）：_dup_content——ASCII 词+单汉字 token；
   最短 8 token 门槛（短泛化痛点只精确匹配）；重叠 ≥0.55 或包含 → 跳过批准
   （裁决行保留供审计，报告"去重 N 条"口径）
```

### 8.2 复现提交（行 1220–1548）

每条 approved pain：
1. `created` 审计事件 → 资格检查 `classify_verification_eligibility`
   （静态主张切 counterfactual + 允许成功读取作 baseline）；
2. 失败 shell 提取原命令；`unsafe_patterns` 命中且 replay 模式 → `unsafe` 终态；
3. 阶段预算（max_jobs_per_phase，兼容旧 max_per_phase）→ `deferred_budget`；
4. **阶段基线快照**：`dispatcher.prepare_phase_baseline`（一次冻结，后续复用）；
5. `dispatcher.submit(ReproJob(...))` 异步执行；
6. 报告前屏障 `dispatcher.await_all` + `_merge_repro_results` 合并。

单任务 `run_repro_job → verify_pains_by_repro`（pain_verifier.py）：
规划（replay/counterfactual/remediation，preflight 预检 + unsafe 拦截含命令摘录）→
隔离副本执行（路径重绑定）→ LLM 判定 → **四道闸门**：
① replay 环境偏离（unset/export 对称差）→ `replay_env_modified` unverified；
② 证据充分性（counterfactual 假设信号 / 失败特征匹配 / 有意义输出）→
  `repro_evidence_insufficient`；
③ flaky 稳定性（replay overturn 候选同命令重跑，两次矛盾）→ `flaky_result`；
④ unknown 归因 → 保留 PainReview 原判。
终态路由：confirmed/overturned 均需 `repro_executed + repro_logged` 双条件；
审计事件流 `repro_audit`（一 case 一 terminal、重提撤销旧终态）。

---

## 9. State 字段读写总表

| 字段 | 写入 | 读取 | 说明 |
|---|---|---|---|
| pending_tasks | worker.py 1230/1422/1053/1484/1595 | 887 | 任务队列 |
| pending_tool_call | 1257–1260 | tools 节点 | 待执行工具调用 |
| last_tool_result | 1596/1486/1642 + tools 节点 | 918/judge | 最近工具结果 |
| judge_retry_instruction | 1491/1599 + judge 2010+ | 1100/432/515 | 打回修正建议 |
| judge_retry_count | 1490/1598 + judge | judge | 打回轮次 |
| judge_retry_active | judge 2101–2107 | 1831 | retry 闭环追踪 |
| judge_retry_history | judge 1936–1938 | — | 闭环结果（50 条） |
| judge_history | judge 1955–1963 | 1075/1843 | 跨任务教训（50 条） |
| judge_stage | 1641 + judge | judge 入口 | execution/pain 分流 |
| phase_retry_state | 1494/1648 | 1357 | 阶段重试预算 |
| pending_retry_request | 1631/1639/1661 | 1616 | 阶段重试事件 |
| task_retry_active | 1496–1503/1249 | 925 | 任务重跑追踪 |
| task_retry_history | 1246–1249 | — | 重跑结果 |
| task_failure_fingerprints | 1495 | 1381–1384 | 同根因去重 |
| early_exit / early_exit_reason | 1589/1547/1632 | reporter | 早退标记 |
| skipped_tasks | 1058/1331/1571/1554 | reporter | 跳过任务 |
| phase_status | 1546/1628/1630 | reporter | failed/skipped |
| search_history | 932–964 | 1095 | 搜索记录 |
| task_tool_history | tools 节点追加；清 1427/1602 | 486/1379/judge | 任务工具史 |
| contrib_*_findings | 1270/1289/1055 | 993–1099 | 贡献阶段结论 |
| all_execution_results | 1037/1477/1607 | 987 | 全部结果（50） |
| total_tokens | advisor 842–848 | 全局 | token 累计 |

---

## 10. Prompt 体系

| Prompt | 文件 | 职责 |
|---|---|---|
| EXECUTOR_SYSTEM_PROMPT | executor/prompts.py | executor 角色与输出协议 |
| TASK_ANALYSIS_PROMPT* | executor/prompts.py | 按阶段 5 变体 |
| TOOL_RESULT_PROMPT* | executor/prompts.py | 结果分析（同 5 变体） |
| inject_judge_feedback | executor/prompts.py | judge 建议前置注入格式 |
| JUDGE_SYSTEM_PROMPT | judge/prompts.py:24 + configs/judge/prompts/doc_conformance/judge_system.txt | 裁判职责/依据/维度/原则/失败分类/伪成功核查/归因防漂移（txt 与常量须同步，有测试守护） |
| JUDGE_DECISION_PROMPT | judge/prompts.py:117 | 复盘 14 占位符模板 |
| JUDGE_SKILL.md | .agents/skills/ | 可热改的评判规则补充（否决场景/git_clone 兜底/维度补充/变体必查） |
| JUDGE_PAIN_REVIEW_SYSTEM_PROMPT | judge/prompts.py:162 + txt | 六分类口径/硬绑定表/责任归因矩阵/幻觉引用规则 |
| JUDGE_PAIN_REVIEW_PROMPT | judge/prompts.py:262 + txt | 复核 7 占位符 + 引用核对段 |
| REPRO_* | judge/prompts.py:285+ | 复现规划与判定 |
| _PLAN_SYSTEM_PROMPT | verification_plan.py:220 | 复现计划器（三模式/可判读命令三条约束） |

---

## 11. 日志落盘

| 日志 | 位置 | 内容 |
|---|---|---|
| judge 详细日志 | logs/json/judges/judge_<proj>_<run>.jsonl | 每次判定全链路：context/input(tool+args+result/doc_evidence/skill)/llm_request/llm_response/decision(verdict+action+retry_instruction)/trigger_reasons/command_class/retry_effectiveness |
| 痛点质控日志 | logs/json/pain_reviews/*.jsonl | 每条痛点：input(pain_content)/llm prompt+raw/decision(is_valid/misjudgment/claims/attribution_validation)；预拦截记录 llm_prompt=(kb_signature_prefilter) |
| 复现事件流 | logs/json/pain_repros/pain_repro_events_*.jsonl | 全生命周期 append-only：created→qualification→snapshot→dispatch→planning→execution→terminal（一 case 一 terminal，重提撤销旧终态） |
| 复现执行日志 | logs/json/pain_repros/pain_repro_*.jsonl | 命令/输出/判定链（截断已放宽：reason 4000/steps data 2000/output_preview 1500） |
| 运行主日志 | logs/multi_run_<ts>/devx_<proj>.log | executor/advisor/judge/evaluator 全部 INFO/DEBUG |

---

## 附：一轮典型交互时序

```
executor_node: 取任务 → advisor.plan_tool_call（judge 反馈前置注入）
  → pending_tool_call → tools 节点执行 → last_tool_result
judge_node: trigger（失败/强制类/抽样/去重）→ 上下文（doc_evidence+历史+judge_history）
  → doc_conformance.judge_decision（LLM）→ verdict
  → pass:（可选 refact_advice/result_anomaly 温和提醒）
  → retry: retry_instruction（四段）+ contract + retry_active → last_tool_result=None
executor_node: 重新规划（advisor 前置看到修正建议）→ 新工具调用
judge_node: classify_retry_outcome(before, after) → resolved/partial/ineffective 闭环
……任务终局（success/failure）→ 任务级重跑（一次，根因分析注入）→ 早退或继续
阶段完成 → evaluator 提取痛点 → judge.pain_review（签名预拦截→LLM→归因闸→升级→
去重→批准）→ 异步 repro（计划→隔离执行→四道闸门→终态）→ reporter（报告+附录D）
```
