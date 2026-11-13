# SiGen 根设计

## 需求背景

本设计描述一个自进化实验 harness：Generator 产出实验对象，环境反馈判定产物是否有效，系统据此改进生成、预判与去重，并审核自身更新。被评估的对象是产物，不是配对本身；配对执行只是比较类任务中取得有效性证据的一种形式。任何预测组件都不提供真值。模型判断与执行动作分离：组件给出命题概率和经验可靠性，调度在预算内决定是否试验，结果按各自命题回流。

总体输入是一份实验配置：评估环境与协议、预算与调度策略、任务描述与覆盖依据、经验库位置及随机种子。对外有两个操作。

第一个是实验闭环 `RunExperiment`：生成完整候选，预判短板命题与去重命题，在统一预算内选择利用、探索与审计对象，执行环境评估，对需要比较的候选与代表做行为对照，追加判定记录与真值，返回本轮报告。延期对象没有物理标签。

第二个是更新生命周期 `ReviewUpdate`：保存候选快照与完整差异，由旧版本审核器预测是否满足收益与保留条件，调度决定延后或有限试行，候选与基础版本进行预算匹配的窗口对照，得到信用与晋升、恢复或待决决定，追加记录并返回更新报告。试行不等于晋升，候选版本不能改写本轮证据或成功口径。

两类操作共享同一个判定接口、经验库和版本可追溯要求。实验对象与更新分别使用自己的真值命题，测试差距不直接冒充更新收益。根仓库只描述接口与证据边界，不预设建模算法。

示例：本仓库本身按此结构组织，根 public 导出两个操作与三类报告数据；`verdict.viba.md` 定义判定与选择，`measure.viba.md` 定义环境评估，`critic.viba.md` 与 `feedback.viba.md` 定义更新判定与信用。

根 README 从这里取 `ExperimentConfig`、`ExperimentReport`、`UpdateReport`、`RunExperiment` 与 `ReviewUpdate`。

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

配置固定本轮比较的不是“版本号大于等于”，而是具体快照与协议。报告区分执行、延期、确认发现与未确认行为簇，避免把执行数量当作发现数量。更新报告只在窗口可用时给出信用，否则明确处于待决。

```viba types
import verdict.public as verdict
import measure.public as measure
import dedup.public as dedup
import critic.public as critic
import feedback.public as feedback
import generator.public as generator

ExperimentConfig =
  Object
  * $round (str * { 本轮唯一标识，绑定后续全部记录与花费 })
  * $task (str * { 任务目标、可比较能力、测量目标与禁止操作。不能把未提供的内部实现当作已知事实 })
  * $setup (measure.EvaluationSetup * { 被测对象、对照或基线及条件的准确版本与差异 })
  * $protocol (measure.Protocol * { 结果前冻结的测量与真值规则，包含实际意义阈值和失败处理 })
  * $budget (verdict.Budget * { 本轮总成本、探索与审计份额。实验和更新批次各自独立使用预算 })
  * $policy (verdict.Policy * { 抽样策略与阈值，须能计算最终入选概率 })
  * $candidate_count (int * { 请求生成的候选数量，与成本预算分开 })
  * $coverage_basis (str * { 独立于判定偏好的覆盖来源和参数范围，用于识别搜索空间收缩 })
  * $representatives (list[generator.Candidate] * { 本轮可比对的代表对象，可为空，但空集合不构成已证明新颖 })
  * $store (str * { 经验库位置，保存不可变记录、原始观测及工件版本引用 })
  * $seed (str * { 可复核随机种子，覆盖生成、选择与审计抽样 })

ExperimentReport =
  Object
  * $round (str * { 本轮标识 })
  * $selections (list[verdict.Selection] * { 全部对象的最终动作与入选概率，延期原因一并保留 })
  * $observations (list[measure.Observation] * { 所有实际执行记录，含无效执行与无结论；未执行对象不在这里占位 })
  * $validations (list[dedup.Validation] * { 本次能完成的去重行为对照，缺代表时可为空 })
  * $records (list[str] * { 已追加的判定记录标识，真值未到的记录也在其中 })
  * $findings (list[measure.Finding] * { 有定位证据的确认短板，未确认行为簇不并入 })
  * $unconfirmed_clusters (float * { 观察到差异但根因未确认的簇数，与确认发现分开报告 })
  * $actual_cost (float * { 本轮实际总成本，含未确认执行、复验与失败重跑 })
  * $deferred (dict[str, str] * { 未执行对象及原因。延期不是负标签，也不能从本轮估计其漏报 })

UpdateReport =
  Object
  * $update (critic.Update * { 本次审核的更新及冻结的对照协议 })
  * $review (verdict.Judgment * { 旧版本审核器给出的原始判断与经验来源 })
  * $selection (verdict.Selection * { 试行、延后及审计路径，含最终入选概率 })
  * $settlement (Settlement * { 完成对照时的信用与晋升决定，或证据不足时的待决状态 })
  * $records (list[str] * { 追加的审核与信用记录标识 })
  * $actual_cost (float * { 本轮试行与对照的实际成本 })

Settlement =
  Oneof
  | $settled (Settled * { 窗口对照完成，信用结论与版本决定都已落盘 })
  | $pending (PendingSettlement * { 未完成对照或证据不足，基础版本保持有效 })

Settled =
  Object
  * $credit (feedback.Credit * { 标准化净效应与收益、损失、无结论或对照无效的结论 })
  * $promotion (feedback.PromotionDecision * { 晋升、恢复或待决的实际决定 })
PendingSettlement = $reason (str * { 待决原因与下一次可比窗口条件，不能据此宣称更新安全 })
```

## 基本操作

`RunExperiment` 一轮做六件事：生成、预判与去重、选择、测量、行为对照、回流。

1. 用本轮配置生成完整候选，记录覆盖来源与假设；候选数量与执行成本分开核算。
2. 对每个候选取短板命题与去重命题的原始判断，两个判断都绑定版本、协议与参照集合。
3. 在总成本内联合选择利用、探索与审计对象；审计可越过任一模型软否决，缺失成本的对象明确延期。
4. 对入选对象做环境评估，保存原始样本、真实成本与确认、否定、无结论或无效执行的结论。
5. 对同时具备候选与代表观测的对象做去重行为对照；证据不足返回无结论。
6. 追加判定记录与真值，汇总确认发现、未确认簇、延期与成本。

`ReviewUpdate` 一次更新做五件事：记录候选快照与差异、旧版审核预测、调度决定延后或试行、预算匹配窗口对照、信用与版本决定。任何一步证据不足都停在待决，不用局部指标替代对照。

## apis.viba

```viba apis
import readme.types as readme
import critic.public as critic
import feedback.public as feedback
import generator.public as generator
import dedup.public as dedup
import measure.public as measure
import predictor.public as predictor
import verdict.public as verdict

RunExperiment =
     readme.ExperimentReport
  <- $config (readme.ExperimentConfig * { 本轮任务、评估环境、协议、预算、策略、覆盖依据与经验库 })
  <- {
    跑一轮实验闭环：生成候选，预判并评估去重关系，在总预算内选择利用、探索与审计对象，执行环境评估，按可比观测检验去重结论，追加记录并汇总。未入选对象保留判定与延期原因，不生成物理标签
  }
  <- $generate (generator.Generate * { 生成完整候选：它交回带参数、状态与正确性要求的实验对象，这里逐个送去做判断 })
  <- $predict (predictor.Predict * { 预判短板命题：它交回原始判定与差距预报，这里收进判定记录 })
  <- $assess (dedup.Assess * { 评估去重关系：它交回非冗余概率与关系解释，这里连同预判一起交给调度 })
  <- $select (verdict.Select * { 分配预算：它交回每个对象的动作与最终入选概率，这里只执行入选对象 })
  <- $measure (measure.Measure * { 环境评估：它交回有效结论或无效执行，这里把有效观测作为真值 })
  <- $validate (dedup.Validate * { 行为对照：它交回冗余、非冗余或无结论，这里修正去重命题的标签 })
  <- $history (feedback.Fetch * { 取回历史：它交回可用案例，这里交给预判、去重与审核去校准 })
  <- $record (feedback.Record * { 追加记录：它交回记录标识，这里汇总本轮报告 })

ReviewUpdate =
     readme.UpdateReport
  <- $update (critic.Update * { 基础与候选快照、完整差异、声明与冻结的对照协议 })
  <- $window (str * { 允许的观察窗口、预算和任务范围，须与基础版本对照匹配 })
  <- $store (str * { 经验库位置，保存原始审核判断、试行路径、窗口证据与版本切换 })
  <- {
    审核并试行一次 harness 更新：旧版本审核器预测收益与保留条件，调度决定延后或有限试行，候选与基础版本做预算匹配的窗口对照，得到信用和晋升、恢复或待决决定，追加记录。试行不自动晋升，证据不足返回待决
  }
  <- $review (critic.Review * { 旧版审核：它交回原始判定，这里保留而不让候选版重写 })
  <- $select (verdict.Select * { 决定延后或试行：它交回入选概率与路径，审计可以越过审核器的软否决 })
  <- $evaluate (feedback.Evaluate * { 窗口对照：它交回两版本同预算的发现、覆盖与成本，对照失效时明确返回 })
  <- $attribute (feedback.Attribute * { 分配信用：它交回净效应与结论，区分预测错误与探索代价 })
  <- $promote (feedback.Promote * { 处理版本：它交回晋升、恢复或待决，并记录版本切换 })
  <- $history (feedback.Fetch * { 取回历史：它交回与更新命题同协议的案例，供审核器校准 })
  <- $record (feedback.Record * { 追加记录：它交回记录标识，这里汇总更新报告 })
```
