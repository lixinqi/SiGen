# 自进化实验 harness

## 需求背景

这是一个自进化实验 harness 的设计：它以配对真实执行为真值，发现目标系统 A 相对对照系统 B 的稳定短板。A 与 B 是一对角色，不是两个具体产品——任何能等价执行同一份测试、给出客观可重复结果的系统对比问题，都落进这个框架，harness 本身不绑定领域。它要能在运行中改进自己：改进生成测试的方式、改进预判物理结果的方式、改进审核更新的方式，包括改进审核自身的方式。平台里所有做判断的组件——预判器、去重器、审核器——共用同一个判定接口：输出判定值与置信度两个浮点数。组件只负责判断，把判断变成动作是上层的统一调度：置信度高时按判定值执行或拒绝，置信度低时提高探索性执行的概率，让被否决的对象仍有机会获得真值。物理真值逐条回流进经验库，校准各组件的下一次判断。这样没有任何一个组件是不可申诉的硬门：它的误判，无论误放还是误杀，都会变成自己的修正材料。

总输入是一份实验配置：目标系统 A 及其版本、对照系统 B 及其版本、本轮真实执行预算上限、经验库路径。操作分两类。

第一类是实验闭环（快循环）：生成候选测试，逐个预判物理结果并做语义去重，把每个候选的判定值与置信度交给统一调度做出执行决定，对选中的候选做配对物理测量，把真值逐条回流进经验库，返回本轮报告。

第二类是更新审核（慢循环）：任一组件提出一次对 harness 自身的修改（目标组件、基础版本、完整 diff 与声称要解决的问题），审核器给出判定值与置信度，上层按置信度决定接受、拒绝或探索性试行；更新经过一个观察窗口后评估系统发现能力，给这次更新分配信用，返回更新报告。

示例：本仓库就是这套设计的规范——根 public 导出 `RunExperiment` 与 `ReviewUpdate` 两个操作，输入分别对应上述两类操作；`verdict.viba.md` 定义统一判定接口，预判、去重、审核三篇文档的判定操作都交回它定义的 `Verdict`。

## public.viba

```viba public
import readme.types as readme
import readme.apis as readme_api

ExperimentConfig = readme.ExperimentConfig

ExperimentReport = readme.ExperimentReport

UpdateReport = readme.UpdateReport

RunExperiment = readme_api.RunExperiment

ReviewUpdate = readme_api.ReviewUpdate
```

## types.viba

两类操作共用一套词汇：实验配置说明跟谁比、花多少预算、经验库在哪；实验报告记下确认了什么短板、每条判定当时怎么判的；更新报告记下审的是什么、怎么决定的、窗口结束后信用归给谁。

```viba types
import verdict.public as verdict
import measure.public as measure
import critic.public as critic
import feedback.public as feedback


ExperimentConfig =
  Object
  * $target (str * { 目标系统 A 及其版本：要发掘短板的在研产品，精确到提交号；真值绑定版本，版本变了旧结论要重新验证 })
  * $baseline (str * { 对照系统 B 及其版本：成熟的竞对，精确到发布号；所有差距都是相对它测出来的 })
  * $budget (int * { 本轮允许的真实配对执行次数上限；真实执行是最贵的资源，生成、预判、去重、调度的一切节省都以它为分母 })
  * $store (str * { 经验库路径：历次判定档案、物理真值与更新信用都写在这里；组件校准置信度只读它，不自报 })

ExperimentReport =
  Object
  * $generated (int * { 本轮生成的候选测试数量 })
  * $executed (int * { 实际占用真实执行预算的候选数量；包含调度选中的、探索性执行的和随机审计的 })
  * $findings (list[measure.Finding] * { 本轮确认的独立短板清单：差距超过噪声、可复现、按根因去重后的结果；未确认的候选不出现在这里，但它们的真值同样回流了 })
  * $judgments (list[verdict.Judgment] * { 本轮全部判定档案：预判、去重与调度各自的判定值、置信度、探索概率与后续真值，一条不缺；缺一条，事后的校准与归因就断了一环 })
  * $audited (int * { 本轮随机审计的样本数：从高置信否决中抽出并真实执行的数量；它不为零，漏报率才估得出来 })

UpdateReport =
  Object
  * $update (critic.Update * { 本次审核的更新：目标组件、基础版本、完整 diff 与声称要解决的问题 })
  * $verdict (verdict.Verdict * { 审核器给出的判定值与置信度 })
  * $decision (verdict.Decision * { 上层按置信度与探索预算做出的决定：接受、拒绝或探索性试行，以及当时的探索概率 })
  * $credit (feedback.Credit * { 观察窗口结束后的信用归因：这次更新对系统发现能力的净影响，以及它印证还是推翻了当时的判定 })
```

## 基本操作

两类操作，输入一个是实验配置，一个是一次待审更新。

1. 实验闭环：生成候选测试——逐个预判并去重——统一调度按判定值、置信度与预算决定执行谁——配对物理测量——真值回流——返回实验报告。
2. 更新审核：审核器判定更新——统一调度按置信度决定接受、拒绝或探索性试行——进入观察窗口——评估系统发现能力——给更新分配信用——返回更新报告。

## apis.viba

```viba apis
import readme.types as readme
import verdict.public as verdict
import generator.public as generator
import predictor.public as predictor
import dedup.public as dedup
import measure.public as measure
import critic.public as critic
import feedback.public as feedback

RunExperiment =
     readme.ExperimentReport
  <- $config (readme.ExperimentConfig * { 实验配置：目标与对照、预算上限与经验库路径 })
  <- {
    跑一轮实验闭环：先生成候选，再对每个候选做预判与语义去重，把判定值与置信度交给统一调度；调度选中的候选做配对物理测量；每条判定连同真值写入经验库；报告给出确认的短板清单与全部判定档案
  }
  <- $generate (generator.Generate * { 生成候选：交回一批候选测试及生成依据；它参考经验库里的物理结果选题，本操作不把预判器分数递给它迎合 })
  <- $predict (predictor.Predict * { 预判：给每个候选交回判定值、置信度与预测的配对差距；本操作拿它排序，不拿它过滤 })
  <- $assess (dedup.Assess * { 去重：交回候选的语义归类与判定；冗余重复折叠到逻辑簇代表，受控变体被认出并保留 })
  <- $decide (verdict.Decide * { 统一调度：把判定值、置信度与探索预算变成执行决定；低置信候选与高置信否决里的审计样本，都在这里获得执行机会 })
  <- $measure (measure.Measure * { 物理测量：在目标与对照上配对执行，交回差距、噪声与是否成立；未确认的结果同样作为真值回流，预判器靠它们知道自己的假阴性 })
  <- $record (feedback.Record * { 回流：把每条判定档案写进经验库；之后的校准与信用归因只认这里写下的记录 })

ReviewUpdate =
     readme.UpdateReport
  <- $update (critic.Update * { 一次待审更新：目标组件、基础版本、完整 diff 与声称要解决的问题 })
  <- $window (int * { 观察窗口长度：更新被接受或试行后，允许运行多少轮实验再做评估 })
  <- {
    审核一次 harness 更新：审核器给出判定值与置信度，上层按置信度决定接受、拒绝或探索性试行；试行与接受一样进入观察窗口；窗口结束后评估系统发现能力，给更新分配信用；审核判定、决定与信用结论写入经验库，成为审核器校准自己的材料
  }
  <- $review (critic.Review * { 审核：交回更新的净收益判定与置信度；它参考自己的误判史，包括被它否决之后证明有价值的更新 })
  <- $decide (verdict.Decide * { 决定：按置信度与探索预算接受、拒绝或探索性试行；低置信度的拒绝不会天然成为终局 })
  <- $evaluate (feedback.Evaluate * { 窗口评估：交回窗口内主指标与同预算对照的差距，以及旧能力是否退化；信用分配以它为准 })
  <- $record (feedback.Record * { 回流：把审核判定、决定与信用结论写进经验库；审核器的下一次置信度从这些记录里校准 })
```
