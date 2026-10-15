# Pipeline 判据融入 CI 分类器：ops-nn 局部试验与方案

> **状态：试验记录与方案草稿，不是已批准的生产口径。**
> 本文记录 2026-10-15 对 ops-nn 的小样本并行分析。试验只读取现有数据，没有改 CI 分类代码、`MECHS`、`labeled` 或 `daily`。新增 mech 和生产判定尚未审定。

## 1. 要解决的问题

`pipeline-skill` 的 `rules.json` 包含日志正则、规则描述和 pipeline 自身的大类/小类。CI 项目已有自己的 `cls`（责任/问题大类）、`mech`（失败模式键）和 `MECHS`（人工维护的解释与处置知识库）。两边类别含义并不完全相同，因此需要通过映射层把 pipeline 命中与 CI 现有责任边界、失败位置关联起来；pipeline 规则描述也不能直接充当已核实根因。

本文记录有限样本中的命中情况、类别冲突和候选 mech，并描述一套将 pipeline 规则接入 CI 判定流程的方案。

## 2. 数据与规则来源

- pipeline 判据库：`pipeline-skill/gate-failure-analysis/references/rules.json`，试验时版本为 v7，包含 3 个大类、31 个小类、229 条正则。
- pipeline 日志分析器：`pipeline-skill/gate-failure-analysis/scripts/analyze_log.py`。它对日志错误行做正则搜索，并从日志尾部方向回溯；它本身不负责 CI 的 run/job 权威失败选择。
- CI 分类器：`ci_experience_agent/observation/parser/classify.py`，先按 CI 规则选择失败 job 并分类；Flaky 复判在后续处理。
- CI 根因知识库：`ci_experience_agent/observation/parser/mechanisms.py`。它按 `mech` 键提供人工审定的 `status`、`cause`、`evidence`、`guess`、`action`、`dest`，以及部分条目使用的 `recheck`。
- 样本：ops-nn 已有 `labeled` 中有对应日志的记录。按 `cls` 分成“待定”“代码失败”“Action 平台失败”，每类抽 20 条，共 60 条；随机种子为 `20261015`。这是局部样本，不代表全量分布。

## 3. 局部试验结果

将 60 条样本的日志交给 pipeline 分析器做只读匹配：

| CI 原分类 | 样本数 | pipeline 命中 | 命中概况 |
|---|---:|---:|---|
| 待定 | 20 | 17 | `code_check` 6、`compile_error` 5、`env/runner` 3、`llt_fail` 2、`env/network` 1；另有 2 条 `unknown/unclassified`、1 条未命中 |
| 代码失败 | 20 | 20 | `compile_error` 10、`llt_fail` 9、`env/device` 1 |
| Action 平台失败 | 20 | 4 | 命中涉及 runner、compile、network、PR conflict |
| **合计** | **60** | **41** | 命中率 41/60；仅描述本次样本 |

### 发现的类别冲突

有一条 CI 平台失败样本位于 `PreSmoke / A2_pre_smoke / smoke executor`，日志包含 `cannot find -lcust_opapi`。pipeline 将该日志命中为 `code/compile_error`，而 CI 现有规则归为“冒烟产物缺件”。这说明日志命中需要结合失败位置上下文，并遵循 CI 分类器的优先级。

## 4. 新 mech 候选及其字段来源

候选名称：**平台插件下载失败**。此前讨论中也出现过“执行机插件下载/加载失败”这一拟名；“执行机”来自 pipeline 的 `Runner机器异常` 小类，不是日志原文。当前名称贴近 pipeline 判据描述，仍未加入 CI `MECHS`。

对应 pipeline 判据：

| pipeline 字段 | 内容 |
|---|---|
| 大类 | `env` / 环境问题 |
| 小类 | `runner` / Runner机器异常 |
| `pattern` | `Failed to download and load the plugin` |
| `desc` | `平台插件下载失败(初始化环境异常)` |

ops-nn 日志/失败记录中可见：`PreSmoke / ST_test_950 / 初始化步骤`，失败消息包含 `Failed to download and load the plugin`。本次 60 条样本中该规则命中 3 条；这些记录在 CI 里仍是既有标签（如“待定 / 真机冒烟”），并未因试验而改写。

| MECHS 字段 | 当前证据状态与来源 |
|---|---|
| `status` | 草案状态为 `pending`；这是根据当前证据作的状态标记，不是 pipeline 或日志里的字段。 |
| `cause` | 暂无已核实根因。日志证明初始化步骤报告插件下载/加载失败，不能单独证明具体原因或责任方。 |
| `evidence` | 可核对的 stage/job/step、原始消息/日志命中行，以及对应的 pipeline `pattern` 和 `desc`；来源是 CI 数据与 pipeline 判据。 |
| `guess` | 当前未填写。 |
| `action`、`dest`、`recheck` | 当前没有经审定的值。 |

`MECHS` 的根因文字由人工审定。pipeline `desc` 是规则说明，可作为证据来源；它不会自动成为 CI 的 `cause`。

## 5. 融合方案

### 5.1 并行对照阶段

试验输出同时保留 pipeline 命中的 `category`、`subcategory`、规则文本、证据行，以及 CI 的 `stage/job/step/cls/mech`。该阶段只比较两套判定，不覆盖 CI 当前输出，也不回写 `labeled` 或日更数据。

### 5.2 映射结构

每条接入的 pipeline 规则关联以下信息：

`pipeline category/subcategory/pattern → CI cls（可选）+ CI mech（可选）+ 上下文条件`

pipeline 大类与 CI 大类之间不要求名称或数量相等。同一 pipeline 类别可以按 CI 责任边界映射到不同结果；一条规则也可以只提供证据，不直接决定 `cls`。

### 5.3 位置上下文与规则顺序

判定依据包含日志签名以及 CI 所选失败 job 的 stage、job、step。CI 分类器负责 run 内失败 job 选择和规则优先级；pipeline 分析器提供日志签名和规则来源。接入位置处于 CI 强判据之后、宽泛兜底之前。Flaky、冒烟共因和系统性复判继续按 CI 现有后处理顺序运行。

### 5.4 判据与根因知识分层

pipeline 正则识别“日志出现了什么签名”；CI `MECHS` 记录已核实的失败事实、证据、推测、处置动作和去向。pipeline 命中可以关联到已有 mech，也可以形成待人工判读的新 mech 候选。人工审定之前，不生成根因文本，不把候选写入正式知识库。

## 6. 当前状态与边界

- 已完成：ops-nn 60 条有限样本的只读匹配与类别冲突检查。
- 尚未发生：规则映射审定、正式新增 mech、分类器实现、全量重跑和指标变更审计。
- 本文是试验记录和融合方案说明，不代表 pipeline 判据已进入 CI 生产分类流程。
