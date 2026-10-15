# CI 失败判据匹配全流程

> 根据 ci_experience_agent 源码整理，覆盖单条 run 初判、Flaky 复判、系统性复判、截断日志复判，以及判据与知识库的分工。流程顺序按代码执行顺序描述。

## 1. 输出字段

- **cls**：一级责任归属，当前固定三类：代码失败、Action 平台失败、待定。
- **mech**：二级失败模式，例如平台限流、单元测试、构建报错不可见。
- **stage / job / jid / step / msg**：定位分类证据来自哪个阶段、任务、执行 job、失败 step 和平台消息。
- **systemic**：是否命中系统性聚集标记。
- **flaky_from / held_from**：复判前的原分类，便于追溯。

P、C、U 是代码里的局部缩写：P = Action 平台失败，C = 代码失败，U = 待定。

## 2. 总体流程图

~~~mermaid
flowchart TD
    A["raw detail"] --> B["字段投影、幻影 run 去重、恢复被遮蔽的 stage / step 名"]
    B --> C{"run 日期在观测窗口内？"}
    C -- "否" --> X["不进入 labeled"]
    C -- "是" --> D{"run.status == FAILED？"}
    D -- "否" --> S["不写 labeled；成功 run 留给 Flaky 重跑比较"]
    D -- "是" --> E["classify_run：取 status=FAILED 且有 steps 的 job"]
    E --> F{"有失败叶子 job？"}
    F -- "否" --> F1["待定 · 无失败叶子 job"]
    F -- "是" --> G["按失败 step 最早 end_time 排序，选最早失败 job"]
    G --> H["在该 job 内取 steps 顺序中的第一个 FAILED step"]
    H --> I["保存 stage / job 变体名 / jid / step / msg；计算 run 内 wide 标记"]
    I --> J{"平台 msg / job 字段匹配？"}
    J -- "未注册执行机" --> J1["Action 平台失败 · 执行机注册"]
    J -- "内部错误或任务执行错误" --> J2["Action 平台失败 · 平台内部错误"]
    J -- "任务排队超时" --> J3["Action 平台失败 · 排队超时"]
    J -- "申请资源错误或 start_time 为空" --> J4["Action 平台失败 · 资源申请"]
    J -- "都没有命中" --> K["读取所选失败 step 的日志尾"]
    K --> L{"跨阶段平台日志签名？"}
    L -- "TOO_MANY_REQUESTS" --> L1["Action 平台失败 · 平台限流"]
    L -- "覆盖率规则缺失签名" --> L2["Action 平台失败 · 覆盖率配置缺件"]
    L -- "Project not found 签名" --> L3["Action 平台失败 · 流水线配置错误"]
    L -- "插件运行异常签名" --> L4["Action 平台失败 · 平台插件异常退出"]
    L -- "无命中" --> M{"stage family == image？"}
    M -- "是" --> MI{"step == get_pr？"}
    MI -- "是" --> M1["待定 · get_pr 解析"]
    MI -- "否" --> M2{"日志含 No url found for submodule？"}
    M2 -- "是" --> M3["Action 平台失败 · 工作区残留"]
    M2 -- "否" --> M4["Action 平台失败 · 镜像/编排"]
    M -- "否" --> N{"step == download？"}
    N -- "是" --> N1{"image stage 已 COMPLETED？"}
    N1 -- "是" --> N2["Action 平台失败 · 外部依赖 OBS"]
    N1 -- "否" --> N3["待定 · 上游产物缺失"]
    N -- "否" --> O{"通用配置日志签名？"}
    O -- "是" --> O1["Action 平台失败 · 配置"]
    O -- "否" --> P{"upload 产物缺失，且此前步骤全完成？"}
    P -- "是" --> P1["Action 平台失败 · 产物路径缺失"]
    P -- "否" --> Q{"self-hosted 资源或 presmoke 阶段？"}
    Q -- "是" --> Q1{"按设备 → Pod → 产物 → 用例顺序，是否命中签名？"}
    Q1 -- "是" --> Q2["命中硬签名 → 对应平台失败；命中用例签名 → 代码失败"]
    Q1 -- "否" --> Q3["无签名：presmoke → 待定 · 真机冒烟；否则 → 待定 · self-hosted 执行机失败"]
    Q -- "否" --> R{"stage family == compile？"}
    R -- "是" --> R1["compile / verify_package、staticcheck、其他 step 分支"]
    R1 --> R2["具名平台签名 / 泛化构建错误 / 退出码 / 耗时 / pre-commit"]
    R2 --> R3["输出对应 mech；其余 compile 阶段失败落代码检查"]
    R -- "否" --> T{"stage family == ut？"}
    T -- "是" --> T1{"step 或 job 变体名为 st / st_ 前缀？"}
    T1 -- "是" --> T2["代码失败 · 系统测试"]
    T1 -- "否" --> T3["代码失败 · 单元测试"]
    T -- "否" --> U["待定 · 未归类"]
~~~

阶段家族由 **stage_fam()** 归一，例如平台改名后的 stage 仍可归到 image、compile、ut 等族。job 用变体名做失败模式判断；run 内 wide 计算则刻意使用 job 组名，避免把编译矩阵的多路执行误当成多个独立任务。

## 3. 第一轮分类：实际判断顺序

每条失败 run 只选**最早失败 job**作为主归因证据。选定 job 后，分类器读取它的第一个失败 step；日志按 repo、run_id、jid、step 拼出路径，读取该 step 的日志尾，采集侧默认保留 1200 行。

下表是 **classify_run()** 的实际顺序。某条件命中后立即返回，不再检查后续分支。

| 顺序 | 读取什么 | 命中条件 | 输出 |
|---|---|---|---|
| 1 | job.message | 完全等于“Self-hosted的执行机未注册。” | Action 平台失败 · 执行机注册 |
| 2 | job.message | 完全等于“内部错误。”或“任务执行错误” | Action 平台失败 · 平台内部错误 |
| 3 | job.message | 完全等于“任务排队超时” | Action 平台失败 · 排队超时 |
| 4 | job.message + job.start_time | 完全等于“任务申请资源错误”，或 start_time 缺失 | Action 平台失败 · 资源申请 |
| 5 | 首失败 step 日志尾 | TOO_MANY_REQUESTS | Action 平台失败 · 平台限流 |
| 6 | 同一日志尾 | classify_rule_list is empty 或 filter need cover code failed | Action 平台失败 · 覆盖率配置缺件 |
| 7 | 同一日志尾 | JSON error_message 含 Project not found: | Action 平台失败 · 流水线配置错误 |
| 8 | 同一日志尾 | ::error::插件运行异常 | Action 平台失败 · 平台插件异常退出 |
| 9 | stage family / step / 日志 | image 阶段：step 是 get_pr 则待定；日志含 No url found for submodule 则工作区残留；其余 image 失败归镜像/编排 | 三种对应 mech |
| 10 | step + stage 状态 | step 是 download：image 阶段已完成则判 OBS 外部依赖；否则判上游产物缺失 | 平台失败或待定 |
| 11 | 首失败 step 日志尾 | YAML error/invalid、invalid workflow、secret not found、workflow permission denied 等配置签名 | Action 平台失败 · 配置 |
| 12 | step + 日志 + 前序 step 状态 | step 是 upload，日志有 Artifact path not found，且此前 step 全部 COMPLETED | Action 平台失败 · 产物路径缺失 |
| 13 | resource.type / stage family + 日志 | self-hosted 或 presmoke：按设备 → Pod → 产物 → 用例顺序检查签名 | 设备、容器、产物为平台失败；冒烟用例为代码失败 |
| 14 | resource.type / stage family | 上一组没有签名：presmoke 归待定·真机冒烟；其他 self-hosted 归待定·self-hosted 执行机失败 | 对应待定 mech |
| 15 | stage family、step 和日志 | compile 阶段的 compile/verify_package step：依次检查加速下载网络、加速工具 HTTP 失败、缺包、补丁重复应用、预编译库符号缺失、泛化构建错误 | 对应平台失败、待定或代码失败 mech |
| 16 | step、日志退出码、step 耗时 | compile 阶段 staticcheck：异常退出且最后退出码为 1 → 静态检查未通过；否则 → 检查器超时。没有日志且耗时 119–122 秒也判超时 | 对应平台失败或代码失败 mech |
| 17 | compile 阶段剩余 step + 日志 | 命中 pre-commit 的格式错误与退出码双条件 | 代码失败 · pre-commit 钩子未通过；否则是代码检查 |
| 18 | stage family + step/job 变体名 | ut 阶段：规范化名称为 st 或 st_ 前缀则系统测试；否则单元测试 | 代码失败 · 系统测试或单元测试 |
| 19 | 以上均未命中 | 没有适用的 stage/step 分支 | 待定 · 未归类 |

### 日志签名清单

- **真机冒烟设备**：ACL_ERROR_RT_DEV_SETUP_ERROR、drvMemDeviceOpen failed、Setup device error
- **冒烟容器**：Pod not found after N attempts
- **冒烟产物**：cannot find -lcust_opapi，或 single/*.run 不存在
- **冒烟用例**：Run eager example failed, op_name:，或 GetWorkspaceSize failed. ERROR: 数字
- **构建加速网络**：connect timeout、socket hang up、connection refused/reset、DNS 解析失败等特定错误
- **构建加速鉴权/下载**：下载失败并输出 HTTP 三位状态码
- **编译产物缺失**：ERROR: Not find *.扩展名 in；还要求 step 为 verify_package，且同 job 的 compile_acc step 已完成
- **补丁重复应用**：Reversed (or previously applied) patch detected
- **预编译库链接错误**：路径在 lib_cache/ 下的 .a 归档中，且出现 undefined reference
- **泛化构建失败**：出现 gmake Error 或构建命令执行失败，同时没有 GCC/Clang 行列错误或 CMake Error at 这类具名错误 → 待定·构建报错不可见
- **pre-commit**：日志先有“存在部分文件格式错误”，后有 precommit.py 异常和退出码 1；中间允许其他日志，中文逗号支持半角和全角
- **静态检查退出码**：从日志中取最后一个 exit code/退出码；退出码为 1 表示检查器跑完并检出违规，否则按平台超时处理
- **通用配置**：YAML error/invalid、invalid workflow、secret not found、workflow permission denied

大多数日志签名是对日志尾执行正则 search；少数是普通子串判断。许多规则还附带阶段、step、job 状态或耗时条件，不能只看日志中是否出现某个词。

### 匹配操作的类型

| 类型 | 代码实际做法 | 例子 |
|---|---|---|
| 消息完全相等 | 对 job.message 使用 == 或 in 元组 | 未注册执行机、内部错误、排队超时 |
| step/job 名称 | 等值、子串、前缀或规范化后比较 | download、compile 子串、staticcheck、st / st_ |
| 日志签名 | 多数用预编译正则的 search；个别用字符串 in | 限流、冒烟设备、补丁重复应用 |
| 组合判据 | 签名必须与阶段、step、其他 job/step 状态同时成立 | upload 产物缺失且此前步骤全完成 |
| 结构化值 | 比较退出码、start_time、step 耗时、完成状态 | staticcheck 退出码和 119–122 秒超时墙 |
| 多 run 统计 | 按日期、PR、step、stage.identifier 聚合不同 PR 数和比例 | Flaky 双键、系统性失败 |

## 4. 第二轮：Flaky、冒烟共因和系统性失败

~~~mermaid
flowchart TD
    A["第一轮已分类的失败 run"] --> B{"同一个 PR 有更晚的 COMPLETED run？"}
    B -- "否" --> C["Flaky 不改判，继续后续系统性检查"]
    B -- "是" --> D{"head_sha 相同，作者代码 commit key 也相同？"}
    D -- "否" --> C
    D -- "是" --> E{"当前 cls 为代码失败或待定，且 mech 还不是 Flaky？"}
    E -- "否" --> C
    E -- "是" --> F{"mech 属于 NON_IDEMPOTENT_MECHS？"}
    F -- "是：pre-commit" --> C
    F -- "否" --> G["读取失败 run 自己的 step 日志"]
    G --> H{"ENV_PAT 命中且 TEST_PAT 未命中？"}
    H -- "是" --> H1["Action 平台失败 · Flaky-环境"]
    H -- "否" --> I{"TEST_PAT 命中？"}
    I -- "是" --> I1["代码失败 · Flaky-用例"]
    I -- "否" --> I2["待定 · Flaky-待分型"]
    H1 --> J["冒烟共因分组"]
    I1 --> J
    I2 --> J
    C --> J
    J --> K{"A2_smoke + manifest-management-plugin，同日同 stage.identifier 的失败 PR ≥ 5，且当前 mech=真机冒烟？"}
    K -- "是" --> K1["mech 改为真机冒烟共因失败；cls 仍待定"]
    K -- "否" --> L["通用跨 PR 系统性检查"]
    K1 --> L
    L --> M{"同日同 step：失败 PR ≥ 5，且失败 PR / 触达 PR ≥ 30%？"}
    M -- "是" --> M1["systemic=true"]
    M -- "否" --> N["保留 systemic 值"]
    M1 --> O{"systemic 且 cls=代码失败？"}
    N --> O
    O -- "否" --> P["保留 cls / mech"]
    O -- "是，pre-commit 豁免" --> P
    O -- "是，其他 mech" --> Q["cls 改为待定；mech=疑似系统性失败；held_from 留原判"]
    P --> W["写入 labeled"]
    Q --> W
~~~

- Flaky 只对原判**代码失败 / 待定**的 run 尝试；Action 平台失败不走这轮改判。
- ENV_PAT 的匹配项包括：out of memory、词边界 OOM、Killed、No space、Cannot allocate、Connection refused/reset/timed out、Temporary failure、network unreachable/error/timeout/failure、Could not resolve host、502 Bad、503 Service、curl failed/error、特定 137/139 签名、signal 9。
- TEST_PAT 的匹配项包括：[ FAILED ]、assert、tests failed、Expected ... but、gtest、run ut fail。环境和用例签名同时命中时，用例分支优先。
- 冒烟共因只改仍标为“真机冒烟”的对象；分组按日期、stage.identifier、失败 step 的 task 计算，并从符合条件的失败 PR 计数。
- 通用系统性检查按日期和 step 聚合。分子是失败的不同 PR 数，分母是触达该 step 的不同 PR 数，门槛为至少 5 个失败 PR 且比例至少 30%。
- run 内的 wide 标记另由 compile/ut 失败 job 的**不同组名**达到至少 3 种触发。它使用组名而非变体名，避免一个矩阵任务扇出多个平台变体后被误算成多个独立任务。

## 5. 第三轮：截断日志复判

截断日志复判由 **refine_truncated.py** 单独执行，安排在 classify 之后、metrics 之前：

1. 读取 labeled 对应的失败 step 日志尾。
2. 只挑出同时满足这些条件的记录：有 ASSERT FAILED: Run UT TESTCASE；有 [ PASSED ]；没有 [ FAILED ]。
3. 按 run_id 和 jid 读取**这个 job 自己**的完整日志。
4. 完整日志含 [ FAILED ]：尾部截断掩盖了失败用例，维持原判。
5. 完整日志没有 [ PASSED ]：测试块不足以证明“全通过”，维持原判。
6. 完整日志有 [ PASSED ] 且没有 [ FAILED ]：改为**待定 · 用例全通过但判失败**。
7. 完整日志缺失或无法读取：维持原判。

完整日志按 jid 精确配对，避免把同 run 下另一个 job 的失败日志算到当前 job。正常分类只读日志尾；完整日志复判只读取命中特定矛盾形态的 job。

## 6. 判据、知识库和日志签名不是一回事

- **实际匹配条件**主要写在 ci_experience_agent 的 observation/parser/classify.py 中：classify_run() 和 flaky_refine()。
- **MECHS 知识库**在 observation/parser/mechanisms.py：为 mech 提供 status、cause、evidence、guess、action、dest 等说明。分类器不是遍历 MECHS 条目来匹配；条目缺失时页面可显示待人工判读。
- **log_signature()** 在 metrics 生成问题卡时提取日志摘要，用于问题签名/聚合；它不产生 cls 或 mech。
- **propose_rules.py** 用于从待定样本中挖掘规则候选，不在日更分类瀑布中自动替 run 下结论。
- cls/mech 写进 data/labeled/{repo}.jsonl 后，截断复判还可能改写它们；之后 metrics.py 才读取最终 labeled 产生日聚合和问题卡。

## 7. 代码入口索引

以下行号对应整理本文时读取的 ci_experience_agent 工作区版本：

| 文件 | 位置 | 作用 |
|---|---:|---|
| observation/parser/classify.py | 79–106 | detail 装载、去重、叶子 job、资源类型和日志尾读取 |
| observation/parser/classify.py | 165–176 | 首个失败 step、step 耗时 |
| observation/parser/classify.py | 183–265 | 冒烟、构建加速、产物、构建失败、补丁、链接、插件、pre-commit 正则 |
| observation/parser/classify.py | 278–372 | 退出码、限流、覆盖率配置、流水线配置、系统性阈值、Flaky 正则 |
| observation/parser/classify.py | 374–557 | 单 run 分类瀑布 |
| observation/parser/classify.py | 588–605 | Flaky 复判与日志分型 |
| observation/parser/classify.py | 607–728 | 窗口裁剪、Flaky 双键搜索、冒烟共因、系统性拦截 |
| observation/parser/refine_truncated.py | 43–115 | 完整日志复判 |
| observation/run_daily.py | 44–48 | classify → 截断日志复判 → metrics 的调用顺序 |
| observation/parser/metrics.py | 246 | 问题卡使用 log_signature 摘要 |
| observation/parser/mechanisms.py | 1–7、15 起 | MECHS 知识库字段和人工审定纪律 |
