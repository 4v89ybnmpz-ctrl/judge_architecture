# Judge 拦截成功演示

**用途**：展示 Judge 如何拦截 Agent 误操作、环境问题和外部依赖问题，避免这些问题被错误写入最终项目报告。

## 1. 演示目标

没有 Judge 时，Agent 可能把以下问题错误包装成项目缺陷：

- 没有完整遵循项目文档；
- 使用错误的命令、参数或目录；
- 环境、芯片型号、CANN 版本或外部依赖不匹配；
- 忽略前置步骤失败，继续执行后续命令。

Judge 的目标不是让所有命令都成功，而是阻止责任归属错误的问题进入正式报告。

## 2. 演示数据流

    Agent 执行操作
          ↓
    产生失败结果
          ↓
    Evaluator 候选痛点
          ↓
    Judge execution_review / pain_review
      ├─ Agent/环境/外部责任 → discard 或 unverified → 不进入正式报告
      ├─ 需要修正操作 → retry → Executor
      └─ 项目责任且证据充分 → keep/revise → 异步复现门禁

## 3. 典型拦截案例

以下案例中的“候选痛点”代表 Agent 可能提交给 Reporter 的原始归因；Judge 必须先回看工具参数、文档证据、前置步骤和环境事实，再决定是否放行。

### 案例 A：cann/shmem —— Agent 跳过文档要求的依赖安装

阶段：quickstart_doc。原始现象是运行 matmul_allreduce 时出现 ModuleNotFoundError: torch_npu。
人工审核结论：quickstart 4.4 已说明 PyTorch 和 torch_npu 安装要求，Agent 没有按要求安装。

Agent 候选痛点：

    项目缺少 torch_npu 依赖声明，导致 matmul_allreduce 示例无法运行。

Judge 判决：

    decision: discard
    responsibility: agent
    misjudgment_type: agent_fault
    reason: 文档已说明 torch_npu 安装要求，但 Agent 未执行前置依赖安装。

结果：不进入正式项目痛点。

### 案例 B：cann/sip —— Agent 未处理安装器 EULA

阶段：setup_prereq。Agent 执行 .run 安装命令后因等待 EULA 确认而失败，随后报告“项目安装脚本有问题”。
人工审核结论：该记录是非问题，安装时需要交互确认或使用 yes 管道。

Judge 判决：

    decision: discard
    responsibility: agent
    misjudgment_type: agent_fault
    reason: Agent 未处理安装器交互确认，不能将未完成安装归因于项目。
    retry_suggestion: 确认 EULA 后按文档重新安装。

### 案例 C：cann/runtime —— Agent 使用错误的 Ops 安装参数

阶段：setup_prereq。Agent 使用 --full 安装 A3 Ops 失败，并报告“Ops 包与项目不兼容”。
人工审核结论：文档实际给出了 Ops 包的 --install 安装方式。

Judge 不接受原始操作，要求 Executor 修正：

    decision: retry
    responsibility: agent
    reason: A3 Ops 应使用文档规定的 --install 参数。
    retry_suggestion: 改用文档原始命令，并核对 SoC 与 Ops 包类型。

如果修正后成功，原始失败不会形成项目痛点。

### 案例 D：cann/amct —— 芯片能力限制被误报为项目缺陷

阶段：quickstart_doc。DynamicMxQuant 在 ascend910_93 上报错：

    SoC version ascend910_93 verification failed

Judge 识别到错误明确指向 SoC 能力限制，判定为环境/硬件责任：

    decision: discard
    responsibility: environment
    misjudgment_type: unsupported
    reason: 当前 SoC 不支持该能力，不能直接归因于项目代码。

### 案例 E：cann/asc-tools —— 外部产品线工具链责任不明确

阶段：quickstart_doc/dev_run。msopgen gen 找不到 new_op_project_template，Agent 报告 asc-tools 打包缺陷。
人工审核结论：msopgen 模板由产品线工具和软链接提供，当前证据不能直接归因于 asc-tools 仓库。

Judge 采取 fail-closed 判定：

    decision: unverified
    responsibility: external
    misjudgment_type: unsupported
    reason: 缺少模板来源和安装链路证据，暂不能确认属于 asc-tools 项目。

结果：暂不进入正式项目痛点，保留审计记录。

## 4. 拦截结果汇总

| 案例 | 错误归因 | Judge 判决 | 责任归属 | 正式项目报告 |
|---|---|---|---|---|
| shmem torch_npu | 忽略已读依赖要求 | discard | agent | 否 |
| sip EULA | 未处理安装器交互 | discard | agent | 否 |
| runtime Ops 参数 | 使用错误安装参数 | retry | agent | 原始问题否 |
| amct SoC 限制 | 硬件能力不支持 | discard | environment | 否 |
| asc-tools 模板 | 外部工具链证据不足 | unverified | external | 否 |

演示性结果：

    候选痛点：5
    拦截或纠正：5
    直接进入正式项目痛点：0
    避免的错误项目归因：5

## 5. 进入正式报告的门槛

只有以下条件同时满足，痛点才允许进入正式项目报告：

    decision ∈ {keep, revise}
    responsibility == project
    evidence 非空
    若存在失败 shell_exec，则异步复现 confirmed

## 6. 扩展拦截样例

以下样例同样来自人工审核知识库，用于展示 Judge 面对不同类型候选问题时的处理边界。

| 编号 | 项目/阶段 | 事实证据 | Judge 判罚 | 拦截动作/报告结果 |
|---:|---|---|---|---|
| 6 | shmem / quickstart | -M 参数来自错误目录或错误命令上下文，审核结论为非项目问题 | discard / agent | 核对 README 工作目录和脚本帮助，不写入报告 |
| 7 | shmem / dev_run | set_env.sh 实际路径与 Agent 拼接的 shmem/latest 路径不一致，审核结论为非项目问题 | discard / agent-or-context | 打回路径选择，不把路径猜测写成项目缺陷 |
| 8 | shmem / test_unit | 错误码 -3 与无 RDMA 网卡环境同时出现，相关用例由 CI 看护 | discard / environment | 标记硬件前置条件缺失，不生成项目痛点 |
| 9 | shmem / setup_deps | torch_npu 版本必须与 CANN 配套，不能简单写入通用 requirements | discard / environment-or-policy | 按 CANN 版本安装，不误报依赖策略 |
| 10 | pypto / test_unit | UTest 崩溃意见与缺少 --forked 说明重复，审核不接受为新问题 | discard / duplicate | 合并重复证据，避免重复报告 |
| 11 | pypto / quickstart | “输出全为 0”不在实际流程中，且审核复测无法复现 | unverified / unknown | 保留审计，禁止进入正式报告 |
| 12 | asc-tools / quickstart | 按 README 步骤执行 msobjdump 无异常 | discard / agent | 要求提供实际工作目录和完整命令 |
| 13 | asc-tools / quickstart | numpy 缺失记录已被人工审核标记为非项目问题 | discard / environment | 归入环境依赖，不归因于 asc-tools |
| 14 | asc-tools / dev_run | soc_version 需要结合 npu-smi 和实际芯片确认，审核归为环境链路 | discard / environment | 先补充芯片与版本证据 |
| 15 | runtime / setup_prereq | A3 Ops 文档命令是 --install，Agent 使用 --full | retry / agent | 打回并要求使用文档原始参数 |
| 16 | runtime / quickstart | patch 缺失记录已合并到已有非项目问题 | discard / environment-or-duplicate | 不重复创建新的项目问题 |
| 17 | amct / quickstart | 链接大小写问题被审核判定为非项目问题，属于文件名/平台核对 | discard / agent-or-platform | 确认大小写敏感路径后再归因 |
| 18 | xla-npu / dev_run | compare 不支持可能由未正确安装或 JAX 版本不为 0.8.1 导致 | unverified / environment | 先复核安装和版本 |
| 19 | ops-cv / quickstart | 当前仓明确发布 ascend910b、ascend910_93、ascend950，不包含 Ascend910 | discard / unsupported-hardware | 将不支持硬件从项目缺陷列表剔除 |
| 20 | ops-cv / test_unit | 119 个测试通过但脚本返回 2，审核要求跨环境复测 | unverified / unknown | 复测确认前不进入报告 |

### 扩展样例的统一判罚格式

对每一条候选问题，Judge 都必须同时输出：

    decision
    responsibility
    misjudgment_type
    reason
    evidence
    report_action

例如“测试全通过但退出码为 2”不能直接判定为项目缺陷，应先记录为：

    decision: unverified
    responsibility: unknown
    reason: 测试通过统计与脚本退出码冲突，需要独立复测并检查构建目标退出状态。

如果复测确认脚本逻辑稳定错误，才允许改判为：

    decision: keep 或 revise
    responsibility: project

### 扩展案例的分类统计

以下分类允许重叠，一个案例可能同时属于环境责任和上下文错误。

    Agent 操作/上下文错误：7
    环境、硬件或版本责任：5
    重复或已有结论：2
    证据不足、需要复测：3
    直接进入项目正式痛点：0（本回放只选取人工审核的非项目问题）

## 7. Judge 展示时应强调的结果

本回放的核心成果不是“把所有失败都判成项目问题”，而是展示责任归因被阻断的具体位置：

    读取文档证据
        ↓
    检查 Agent 是否执行前置要求
        ↓
    核对命令、路径、版本和硬件
        ↓
    合并人工审核的非项目问题知识
        ↓
    discard / retry / unverified
        ↓
    阻止错误项目问题进入 Reporter

只有项目责任、证据充分、且需要复现的失败命令在干净环境中得到 confirmed，才允许成为正式项目痛点。
