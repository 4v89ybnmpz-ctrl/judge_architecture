# Pipeline 判据融入 CI 分类器：ops-nn 局部试验与方案

> **状态：试验记录与方案草稿，不是已批准的生产口径。**
> 本文记录 2026-10-15 对 ops-nn 的小样本影子分析。试验只读取现有数据，没有改 CI 分类代码、`MECHS`、`labeled` 或 `daily`。新增 mech 和生产判定仍需人工审定。

## 1. 要解决的问题

`pipeline-skill` 的 `rules.json` 包含日志正则、规则描述和 pipeline 自身的大类/小类。CI 项目已有自己的 `cls`（责任/问题大类）、`mech`（失败模式键）和 `MECHS`（人工维护的解释与处置知识库）。两边的类别含义并不完全相同，因此不能把 pipeline 类别直接当成 CI 的 `cls`，也不能把规则描述自动写成已核实根因。

本次试验要回答：pipeline 日志判据能否为 CI 的待定记录提供有用线索；哪些判据需要结合 stage/job/step 才能安全映射；是否出现值得人工审定的新 mech 候选。

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

有一条 CI 平台失败样本位于 `PreSmoke / A2_pre_smoke / smoke executor`，日志包含 `cannot find -lcust_opapi`。pipeline 将该日志命中为 `code/compile_error`，而 CI 现有规则归为“冒烟产物缺件”。这说明日志正则适合作为线索，但**单靠命中规则就覆盖 CI 分类会有误归风险**；映射必须读取失败位置上下文并遵守 CI 自身的规则优先级。

## 4. 新 mech 候选及其字段来源

候选名称：**平台插件下载失败**。此前讨论中也用过“执行机插件下载/加载失败”这一拟名；“执行机”是从 pipeline 的 `Runner机器异常` 小类推得，不是日志原文。为避免把分类推论写进名称，当前更贴近原始判据的候选名是“平台插件下载失败”。它仍只是候选，未加入 CI `MECHS`。

对应 pipeline 判据：

| pipeline 字段 | 内容 |
|---|---|
| 大类 | `env` / 环境问题 |
| 小类 | `runner` / Runner机器异常 |
| `pattern` | `Failed to download and load the plugin` |
| `desc` | `平台插件下载失败(初始化环境异常)` |

ops-nn 日志/失败记录中可见：`PreSmoke / ST_test_950 / 初始化步骤`，失败消息包含 `Failed to download and load the plugin`。本次 60 条样本中该规则命中 3 条；这些记录在 CI 里仍是既有标签（如“待定 / 真机冒烟”），并未因试验而改写。

| 拟议的 MECHS 字段 | 当前可填写内容及来源 |
|---|---|
| `status` | 建议 `pending`。这是根据目前证据状态提出的建议，不是 pipeline 或日志中的字段。 |
| `cause` | 暂无已核实根因。日志只证明初始化步骤报告插件下载/加载失败，不能单独证明具体原因或责任方。 |
| `evidence` | 可记录失败记录的 stage/job/step、原始消息/日志命中行，以及对应的 pipeline `pattern` 和 `desc`。来源是 CI 数据与 pipeline 判据两者。 |
| `guess` | 暂不填；pipeline 类别不能直接当成已核实推测。 |
| `action`、`dest`、`recheck` | pipeline 判据和日志没有提供这些字段的审定值，留待人工判读和定向。 |

`MECHS` 的根因文字受 CI 项目人工审定规则约束。pipeline `desc` 可以作为规则说明/证据来源，但不能自动复制成 CI 的 `cause`。

## 5. 建议的融合方式

### 5.1 先做影子映射

在独立的试验输出中记录 pipeline 命中的 `category`、`subcategory`、规则文本、证据行，以及 CI 的 `stage/job/step/cls/mech`。先比较，不覆盖 CI 当前输出，也不回写 `labeled` 或日更数据。这样可以看到规则覆盖和冲突，而不改变现有口径。

### 5.2 类别单独映射，不强制相等

为每条获准接入的 pipeline 规则定义映射：

`pipeline category/subcategory/pattern → CI cls（可选）+ CI mech（可选）+ 上下文条件`

一个 pipeline 大类可映射到多个 CI 类别；也可以只提供证据而不直接决定 `cls`。映射须看 CI 的责任边界与实际失败位置，不能按名称相似度自动合并。

### 5.3 匹配必须带位置上下文

除日志签名外，检查失败 job 的 stage、job、step；对已知碰撞规则加明确的上下文条件。规则应放在 CI 的强判据和宽泛兜底之间，优先级由 CI 分类器维护，不能由 pipeline 文件的排列顺序隐式决定。Flaky 和截断日志复判仍按 CI 现有后处理顺序运行。

### 5.4 规则证据与根因知识分离

pipeline 正则负责“日志出现了什么签名”；CI `MECHS` 负责“已核实的失败链、证据、推测、建议动作和去向”。新签名可以先产生待人工判读的 mech 候选；没有人工核实前，不生成根因文本、不把候选写入正式知识库。

## 6. 后续验收建议

进入生产分类器前，先由人工审定拟映射规则，再按规则逐条检查：

1. 在目标样本中是否命中预期记录；
2. 与 CI 既有判据的重叠、优先级和反向误伤；
3. 非失败步骤/不相关 stage-job 的阴性对照；
4. 命中规则但上下文不符时是否保持原 CI 判定；
5. 改判前后 `cls`、`mech`、Flaky 结果和历史日指标的差异。

本次 60 条试验不足以批准生产规则；尤其平台失败样本中存在 pipeline 将平台问题报为编译错误的实例。若以后采用规则，应按 CI 项目的裁决流程完成决策日志、字典、代码和最终产物的同步。

## 7. 当前状态

- 已完成：ops-nn 60 条有限样本的只读匹配与冲突检查。
- 未完成：人工批准规则映射、正式新增 mech、分类器实现、全量重跑和指标变更审计。
- 当前建议：把 pipeline 结果作为候选证据层；先审定上下文映射，再决定是否进入 CI 决策树。
