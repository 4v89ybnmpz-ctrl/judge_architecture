# 独立 Judge 节点架构

**版本**：1.2.0  
**文档状态**：正式版  
**最后更新**：2026-08-04  
**适用范围**：当前 DevX Agent 工作流中的 Judge 节点

## 1. 文档目的

本文只描述 Judge 在当前 Agent 中的架构位置、职责边界和数据流，不重复介绍整个 DevX Agent 的通用架构，也不定义评分机制。

Judge 的最终目的只有一个：

> 防止 Agent 操作错误、运行环境问题、外部依赖问题或证据不足，被错误包装成项目问题并写入最终报告。

Judge 是工作流中的一个独立 LangGraph 节点。`PainJudge` 不是另一个节点，而是 Judge 节点内部的 `pain_review` 处理分支。
异步复现也不是额外的 LangGraph 节点：`ReproDispatcher` 是 Judge 节点拥有的运行级后台服务，结果必须回到 Judge 状态合并后才能影响 Reporter。

## 2. 在总体架构中的位置

按照 [`architecture.md`](./architecture.md) 的分层方式，Judge 属于编排层与 Worker 业务层之间的业务节点：

```text
DevXWorkflowBuilder
        │
        ▼
   StateGraph
        │
        ├── planner      生成任务
        ├── executor     决策并执行 Agent 行为
        ├── tools        执行真实工具
        ├── evaluator    生成候选观察和候选痛点
        ├── judge        独立裁决节点
        └── reporter     生成最终报告
```

Judge 不替代 Executor、Evaluator 或 Reporter：

| 组件 | 负责内容 | Judge 的关系 |
|---|---|---|
| Planner | 生成开发者旅程任务 | Judge 不重新规划任务 |
| Executor | 选择工具、组织操作、处理结果 | Judge 审查其操作是否合理 |
| Tools | 执行 Shell、文件、浏览器等工具 | Judge 使用工具事实作为证据 |
| Evaluator | 从事实生成候选观察和痛点 | Judge 审查候选痛点的真实性和责任归属 |
| Reporter | 汇总并输出报告 | Reporter 只能使用 Judge 放行的数据 |

## 3. 核心数据流

### 3.1 未启用 Judge 时的基线流程

当 `judge.enabled=false`（或工作流构建器显式关闭 Judge）时，工作流不注册 `judge` 节点，
工具执行结果直接返回 Executor，Evaluator 生成的候选痛点也直接交给后续阶段或 Reporter。
此模式保留原始 Agent 行为，但缺少执行责任审查、PainReview 和异步痛点复现门禁。

```mermaid
flowchart TD
    A[Planner] --> B[Executor]
    B --> C[Tools]
    C -->|工具结果直接返回| B
    B -->|当前阶段任务完成| D[Evaluator]
    D -->|还有后续阶段| B
    D -->|全部阶段完成或提前结束| E[Reporter]
    D -.候选观察/候选痛点.-> E
```

### 3.2 启用 Judge 时的数据流

```mermaid
flowchart TD
    A[Executor] --> B[Tools]
    B --> C[Judge 节点]
    C --> D{风险触发策略}
    D -->|跳过普通成功只读操作| A
    D -->|触发审查| E[Judge execution_review]
    E -->|pass / force_pass| A
    E -->|retry| A
    A --> F[Evaluator]
    F --> G[Judge pain_review]
    G -->|keep / revise| H[approved_pain_points]
    G -->|discard / unverified| I[仅保留审计记录]
    H --> N{满足复现条件?}
    N -->|是| J[ReproDispatcher 后台复现]
    N -->|否| K[Reporter 前有界收敛]
    J -.主 Agent 同时继续后续任务.-> A
    J -->|复现确认/推翻/未验证| K
    I --> K
    K --> L{还有后续任务?}
    L -->|是| A
    L -->|否| M[Reporter]
```

说明：Evaluator 完成一个阶段后，无论是否产生候选痛点，都会先进入统一 Judge 的
`pain_review` 阶段；没有候选痛点时 Judge 只完成阶段标记和路由，不调用无意义的逐条审查。
`ReproDispatcher` 是 Judge 拥有的后台服务，不是额外的 LangGraph 节点。

Judge 有两个处理时机，但只有一个节点实现：

| `judge_stage` | 调用时机 | 主要问题 | 结果 |
|---|---|---|---|
| `execution` | 工具调用完成后 | Agent 是否正确执行，失败由谁负责 | `pass`、`retry`、`force_pass` |
| `pain` | Evaluator 产生候选痛点后 | 候选问题是否是可归因于项目的真实痛点 | `keep`、`revise`、`discard`、`unverified` |
| `pain_reviewed` | 当前阶段痛点评审完成后 | 为后续路由标记已完成审查 | 路由到下一阶段或 Reporter |

## 4. Judge 节点的职责

### 4.1 执行审查：`execution_review`

执行审查检查最近一次工具调用及其任务上下文，主要防止 Agent 因为错误操作制造虚假问题。

Judge 重点检查：

- 工具名称、参数、路径和命令是否符合已读取的项目文档；
- 是否在前置步骤失败后盲目执行后续步骤；
- 是否重复执行相同失败操作；
- 失败是否由 Agent、项目、环境或外部依赖导致；
- 工具输出是否足以支持任务成功或失败结论；
- 是否需要让 Executor 修正操作后重试。

典型结果：

```text
正确执行或责任不属于 Agent  → pass
参数/路径/顺序错误           → retry
达到重试上限或 Judge 不可用   → force_pass，并记录风险
```

### 4.2 痛点评审：`pain_review`

痛点评审是 Judge 防止“非项目问题进入报告”的核心环节。Evaluator 只能提交候选痛点，不能直接决定最终报告内容。

Judge 必须同时检查：

1. 是否存在明确失败、阻塞或体验损失事实；
2. Agent 是否基本遵循了项目文档和任务要求；
3. 是否有项目代码、项目文档、构建配置或项目流程证据；
4. 是否排除了环境、权限、网络、驱动、硬件、第三方依赖等因素；
5. 痛点描述是否具体，影响范围是否被夸大。

判定含义：

| 判定 | 含义 | 是否进入正式报告 |
|---|---|---:|
| `keep` | 事实充分且责任归属于项目 | 是（若未处于待复现状态） |
| `revise` | 事实成立，但描述或责任归因需要修正 | 使用修订后的内容 |
| `discard` | Agent、环境、外部依赖或证据不足 | 否 |
| `unverified` | LLM 或异步复现无法形成可靠闭环 | 否 |

## 5. 责任归因模型

Judge 的核心不是判断“有没有报错”，而是判断“谁应为这个问题负责”。

```text
project    项目代码、文档、配置或流程问题
agent      Agent 命令、参数、路径或操作错误
environment 本机环境、权限、网络、驱动、硬件问题
external   第三方服务、依赖或外部组件问题
unknown    证据不足，无法可靠归因
```

只有 `project` 责任的问题可以成为正式项目痛点。`unknown` 默认不放行，但必须保留完整审计信息，便于人工复核。异步复现结束后，复现推翻初审的痛点会改为 `agent`，复现未完成或结果不一致的痛点会改为 `unknown/unverified`。

证据优先级固定为：

```text
安全/破坏性规则
  > 项目文档明确证据
  > 工具执行事实
  > 项目源码、构建脚本和配置
  > 非项目问题知识库
  > LLM 推断
```

知识库只能作为辅助证据，不能覆盖当前运行中新的项目级事实。

## 6. 输入与输出契约

### 6.1 输入

Judge 从 `WorkflowState` 读取：

- `judge_stage`：当前审查阶段；
- `last_tool_result`：最近一次工具结果；
- `tool_history`：当前任务的完整工具调用历史；
- 当前任务、阶段和项目上下文；
- Agent 实际读取到的文档和浏览器证据；
- Evaluator 产生的候选痛点；
- Judge 历史判决和任务级重试次数；
- 同一任务此前 `execution_review` 的判错证据；
- 可验证的运行环境摘要（操作系统、架构、版本、驱动和设备信息）；
- Judge 配置、策略和知识库路径。

### 6.2 输出

执行审查至少写入：

```json
{
  "judge_verdict": {
    "decision": "pass | retry | force_pass",
    "correct": true,
    "reason": "可验证的判定理由",
    "responsibility": "agent | project | environment | external | unknown",
    "evidence": ["命令输出或文档证据"],
    "confidence": 0.0
  },
  "judge_retry_instruction": "仅 retry 时填写",
  "judge_trigger_reasons": ["tool_failure"],
  "judge_trigger_command_class": "build"
}
```

痛点评审至少写入：

```json
{
  "judge_pain_decisions": [
    {
      "decision": "keep | revise | discard | unverified",
      "original_content": "Evaluator 原始候选",
      "content": "最终允许使用的内容",
      "responsibility": "project | agent | environment | external | unknown",
      "evidence": ["项目文档或运行事实"],
      "reason": "判定理由"
    }
  ],
  "judge_approved_pain_points": ["只有 keep 或 revise 的项目痛点"]
}
```

痛点评审调用 LLM 时，逐条和批量模式必须使用同一组上下文：任务事实、文档证据、此前执行审查证据、知识库锚点和运行环境摘要。不能因为批量优化而省略责任归因依据。

`judge_approved_pain_points` 是 Reporter 在 Judge 模式下唯一允许使用的正式痛点来源。列表中的痛点必须同时满足结构化裁决为 `keep/revise`、责任为 `project` 且证据非空；关联失败 `shell_exec` 的痛点还必须通过异步复现门禁。

### 6.3 异步复现状态

Judge 初审为项目责任、并关联失败 `shell_exec` 的痛点会产生稳定的 `repro_job_id`，后台任务状态写入：

```text
judge_repro_jobs       可序列化任务元数据，不包含 Future
judge_repro_results    job_id 到确定性复现结果的映射
```

后台线程不直接修改 `WorkflowState`。后续 Judge 调用非阻塞收集已完成结果；仅在 `early_exit`、当前阶段为 `report` 或没有待处理任务、即将进入 Reporter 时，最多等待 `pain_repro.final_wait_timeout` 秒并形成最终状态。普通阶段不会为单条复现任务阻塞主 Agent。

```text
pending       后台执行中，主 Agent 继续
confirmed     独立复现出现相同退出状态和同类错误，保留项目痛点并追加复现证据
overturned    独立复现成功，改判 agent 并剔除
unsafe        命中危险命令，不执行且不进入报告
failed        复现执行异常，不进入报告
timeout       最终屏障超时，不进入报告
unverified    错误不一致或无法形成确定结论，不进入报告
not_eligible  无法关联失败 shell_exec、没有可执行命令或没有匹配工具调用，不提交复现，沿用文档/源码证据裁决
```

`not_applicable` 仅作为早期状态名称兼容说明；当前实现统一写入 `not_eligible`，避免把“无需复现”和“尚未判定”混为一谈。

复现只针对以下候选提交：

```text
pain_review decision ∈ {keep, revise}
责任归属 == project
evidence 非空
存在关联且失败的 shell_exec
命令未命中 unsafe_patterns
当前阶段尚未达到 pain_repro.max_per_phase
```

痛点与工具调用的关联必须能够解析为当前任务 `tool_calls` 中的具体调用 ID。为兼容历史
Evaluator 输出，Judge 同时接受 `tool_ids`、`tool_id`、`pain_points_tool_nums` 和
`pain_points_tool_nums_flat`；数字引用既支持真实调用 ID，也支持任务内从 1 开始的调用序号。
解析后只选择 `tool == shell_exec` 且执行失败的调用。无法完成映射时记录
`repro_status=not_eligible` 和跳过原因，不会静默丢弃候选。

`ReproDispatcher` 使用稳定的 `run_id + phase + task_id + pain` 哈希生成任务 ID，并对同一任务去重。调度器持有的 `Future` 只存在于运行时注册表，不得写入 LangGraph 状态；状态中只保存任务元数据和可序列化的 `ReproResult`。

## 7. 触发与路由

### 7.1 执行审查触发

以下情况必须触发 Judge：

- 任意工具失败；
- 任务最终失败，需要回看完整工具链；
- 构建、运行、安装、下载、环境设置、修改或删除操作；
- 影响文档证据或责任归因的文件读取、搜索或浏览器操作；
- 返回空结果、未找到结果或关键参数由搜索结果决定。

普通成功只读操作可以跳过，但跳过不能覆盖失败、证据影响或责任归因审查。

`execution_review` 的 `retry` 受 `judge.max_retries` 限制（当前默认 2 次）。同一任务连续被
Judge 判错达到上限后，不再继续循环，而是记录 `force_pass` 并放行当前结果；这表示防死循环
的降级，不表示该操作已被 Judge 认可。

### 7.2 路由规则

`route_after_judge` 根据 `judge_stage` 和判决结果路由：

```text
execution + retry       → executor
execution + pass        → executor，继续处理工具结果
execution + force_pass  → executor，带风险记录继续
pain + retry 需求       → executor，重新完成任务
pain + reviewed         → 下一阶段或 reporter
```

Judge 不直接调用工具，也不直接修改项目文件。它只能通过状态中的判决和修正指令影响 Executor 的下一步行为。

## 8. Reporter 门禁

Reporter 必须先对 Judge 状态进行清洗：

1. Judge 开启时，丢弃未审核的候选痛点；
2. 只读取 `judge_approved_pain_points`；
3. `discard`、`unverified` 以及未完成复现的结果只留在审计日志和 `judge_pain_decisions`；
4. Judge 未完成痛点评审时，不得默认把候选痛点放行；
5. 保留原始事实与裁决结果的关联，支持报告追溯；
6. Reporter 再次过滤 `pending/unsafe/failed/timeout/unverified`，形成最终 fail-closed 门禁。

因此，数据流必须满足：

```text
Evaluator 候选痛点
        │
        ▼
Judge 审核/修订/丢弃
        │
        ├── judge_pain_decisions（完整审计）
        └── judge_approved_pain_points（Reporter 门禁输入）
```

## 9. 可靠性与降级

Judge 的失败不能导致 Agent 无限循环，也不能静默制造结论：

- 单任务重试次数由 `judge.max_retries` 和 `task_retry_counts` 限制；
- 后台复现由 `pain_repro.max_workers` 限制并发、`max_per_phase` 限制阶段任务数；
- 复现默认复制到临时项目快照，避免与主 Agent 同时修改工作目录；
- Reporter 前只进行一次有界等待，普通任务和阶段流转不等待单条复现；
- 复现快照是工作目录隔离和污染控制措施，不是硬安全边界；危险命令过滤和 Sandbox 策略仍是安全边界；
- 同一失败操作反复出现时，停止盲目重试并记录风险；
- Judge 超时、LLM 不可用或响应无法解析时，执行审查使用 `force_pass`；
- `judge.max_retries` 达到上限时，执行审查使用 `force_pass`，并在 Judge 日志中记录实际判错原因和重试次数；
- 痛点评审无法验证时，候选痛点不得自动升级为正式项目问题，应标记为未验证或丢弃；
- 所有降级都要写入 `judge_metrics`、审计日志和轨迹。

## 10. 可观测性

每次 Judge 调用至少记录：

- 审查阶段和任务 ID；
- 触发原因与命令分类；
- 输入摘要、工具历史和证据来源；
- LLM 请求、响应解析状态和耗时；
- 最终判决、责任归属、置信度和重试次数；
- 痛点评审的原始内容、修订内容或丢弃原因。

`judge_trajectory` 使用 action-observation 形式记录 Agent 操作和工具观察，支持复盘“Agent 做了什么、工具返回了什么、Judge 为什么这样判”。

## 11. 实现映射

| 架构职责 | 当前实现 |
|---|---|
| 节点创建与统一调度 | `src/workflow/nodes.py:create_judge_node` |
| 工作流注册与边 | `src/workflow/workflow.py` |
| 执行后路由 | `src/workflow/routing.py:route_after_judge` |
| 执行触发策略 | `src/workers/judge/trigger_policy.py` |
| 判定策略接口 | `src/workers/judge/strategies/base.py` |
| 文档一致性策略 | `src/workers/judge/strategies/doc_conformance.py` |
| 非项目问题知识库 | `src/workers/judge/knowledge_base.py` |
| 轨迹构建 | `src/workers/judge/trajectory.py` |
| 异步复现调度 | `src/workers/judge/repro_dispatcher.py` |
| 确定性命令复现 | `src/workers/evaluator/pain_verifier.py` |
| Reporter 门禁 | `src/workers/reporter/worker.py` |
| 状态字段定义 | `src/workflow/workflow.py:WorkflowState` |

关键运行状态字段：

| 字段 | 用途 | 约束 |
|---|---|---|
| `judge_approved_pain_points` | Reporter 的正式痛点输入 | 只含结构化裁决放行项，最终屏障后不得含未验证复现项 |
| `judge_pain_decisions` | 每条候选的完整裁决和复现改判记录 | 保留 discard/unverified，支持审计 |
| `judge_repro_jobs` | 复现任务元数据 | 可序列化，不含 Future、线程或客户端对象 |
| `judge_repro_results` | `job_id -> ReproResult` 序列化映射 | 结果必须可写入日志和报告上下文 |

## 12. 设计边界

Judge 明确不负责：

- 生成完整评估计划；
- 替代 Executor 进行工具选择；
- 修复项目代码或环境；
- 直接替代 Executor 重做普通工具调用；仅允许 ReproDispatcher 在隔离快照中执行受限复现；
- 计算或调整项目评分；
- 编造不存在的项目证据；
- 将证据不足的问题强行归因给项目。

Judge 只负责裁决、归因、给出必要的重试指令，以及为 Reporter 提供经过审核的结果。

## 13. 变更同步要求

任何以下变更都必须同步更新本文档、`docs/judge_node_design.md`、相关测试和实现：

- Judge 触发条件或路由；
- `WorkflowState` 中的 Judge 字段；
- 判决枚举、责任归因或降级规则；
- Reporter 的数据门禁；
- 审计日志、轨迹或知识库输入；
- Judge 策略、提示词和配置。

完成标准是：架构文档、实现、测试和真实运行日志对 Judge 行为的描述一致。
