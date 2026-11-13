# 回流、信用与检查点（feedback）

## 需求背景

快循环每次执行产生物理观测，慢循环需要观察窗口才能判断更新。两类证据写入同一个经验库，但用途不同：测试测量不能直接当作更新收益，更新窗口也不能替代某个负载是否成立短板的真值。回流要把这个界限保留下来。

原始证据只追加。判定、决定和真值分开保存，后续校准使用原始预测而非事后重算的分数。候选版本的更新若被允许改写旧观测或旧判定，比较就失去意义。经验库还要记录工件版本、协议、选择概率和证据引用，否则相似数量的案例会被误认为独立经验。

信用分配从版本对照实验开始，不从前后的单串指标开始。候选版本与基础版本在预算、任务和评估协议上匹配运行；主结果之外同时检查保留能力、覆盖以及成本。观察不足时输出无结论，等下一次对照，而不是让更新作者或当前审核器自行认定收益。

误放和误拒绝不是作者填写的类别。它们由原始判定、选择路径和后续版本对照推出：预测差却试行后获益，是拒绝侧的代价；预测好却试行后退化，是接受侧的代价；经探索试行仍无可用对照，则还没学到东西。这样就不必在证据不足时选一个“好/坏”标签。

校准也是回流的一部分。判定组件生产时调用 `verdict.Calibrate` 取得经验置信度，本轮结束后用有效结果检验其可靠性。短周期内的少量错误不支持立即换算法；持续的校准偏差应作为可审核的更新，而不是某个组件偷偷调整自己的分数。选择概率为零或未知的记录，在估计总体效果时不能当作随机样本。

根 README 从这里取 `JudgmentRecord`、`Credit`、`WindowEvaluation`、`PromotionDecision`、`Record`、`Fetch`、`Attribute`、`Evaluate` 与 `Promote`。

## public.viba

```viba public
import feedback.types as feedback
import feedback.apis as feedback_api

JudgmentRecord = feedback.JudgmentRecord
Credit = feedback.Credit
WindowEvaluation = feedback.WindowEvaluation
PromotionDecision = feedback.PromotionDecision
Record = feedback_api.Record
Fetch = feedback_api.Fetch
Attribute = feedback_api.Attribute
Evaluate = feedback_api.Evaluate
Promote = feedback_api.Promote
```

## types.viba

一条记录绑定一个判定、它的选择路径和它自己的真值。信用是版本对照的结论，和测试记录同属经验库却不能被互换。窗口评估同时承载主结果、成本、覆盖与保留情况，避免只用一项漂亮指标解释更新。

```viba types
import verdict.public as verdict
import measure.public as measure
import dedup.public as dedup

JudgmentRecord =
  Object
  * $identity (str * { 记录不可变标识，后续追加结果不改写原条目 })
  * $judgment (verdict.Judgment * { 结果揭晓前的原始命题、特征、预测和工件版本 })
  * $selection (verdict.Selection * { 当时调度对该对象的最终动作与入选概率；延期对象没有物理标签 })
  * $truth (RecordedTruth * { 与该命题同语义的有效结果、无结论或尚未获得；不把其他组件的标签借来填这一格 })

RecordedTruth =
  Oneof
  | $measured (measure.Observation * { 预判命题的测量观测。只有有效且结论明确时才能校准；无效执行保留供诊断 })
  | $validated (dedup.Validation * { 去重命题的行为对照结论。只有明确非冗余或冗余才校准，无结论仍是未知 })
  | $credit (Credit * { 审核命题的版本对照信用，针对更新而不是测试 })
  | $pending (PendingTruth * { 真值尚未到达，原因与后续机会写清，不能把延期算作判对 })

PendingTruth = $cause (str * { 预算或资源延期、窗口未结束、等待代表观测等。原因本身不参与任何真值统计 })

Credit =
  Object
  * $update (str * { 更新标识，绑定候选快照与基础快照 })
  * $comparison (str * { 对照记录标识，包含冻结协议、预算和窗口，而不是笼统的“上线一段时间” })
  * $original_verdict (verdict.Verdict * { 试行或晋升前旧版本审核器给出的原始判断，用于区分预测与结果 })
  * $original_selection (verdict.Selection * { 当时的试行或延期决定及入选概率，探索救回的案例才能与模型命中分开 })
  * $effect (float * { 候选相对基础版本在发现口径上的标准化净效应，正为改善，负为退化。量纲与阈值由对照协议规定，不与测试差距混算 })
  * $outcome (CreditOutcome * { 明确收益、明确损失、尚无定论或对照无效。没有充分对照就不强填好坏 })
  * $evidence (list[str] * { 窗口评估、原始发现记录和保留检查的引用，便于复核归因 })

CreditOutcome =
  Oneof
  | $gain (Gain * { 匹配对照支持超过晋升阈值，并满足保留约束 })
  | $loss (Loss * { 匹配对照支持未达阈值或造成可识别退化。退化位置必须能从证据定位 })
  | $inconclusive (InconclusiveCredit * { 成本、噪声或窗口长度不足，当前无法区分收益与退化 })
  | $invalid (InvalidComparison * { 对照本身不可用，如评估协议被候选版改写、预算或任务不匹配 })

Gain =
  Object
  * $margin (float * { 超过晋升阈值的标准化幅度 })
  * $retention (str * { 旧能力、历史探针和覆盖约束的检查结果引用，收益不能以缩窄搜索空间换取 })
Loss = $symptom (str * { 可定位的退化表现及证据引用，不把整体噪声随意归到某段差异 })
InconclusiveCredit = $reason (str * { 尚不能归因的原因，保留待下一次可比窗口 })
InvalidComparison = $evidence (str * { 对照失效的具体证据；不能据此宣布更新成功或失败 })

WindowEvaluation =
  Object
  * $comparison (str * { 对照标识，固定协议、预算、任务和窗口范围 })
  * $protocol (str * { 事先确定的发现口径、晋升阈值、保留约束和停止规则 })
  * $base (MetricSummary * { 基础版本的窗口指标，作为同预算对照 })
  * $candidate (MetricSummary * { 候选版本在相同预算与任务下的窗口指标 })
  * $retention (str * { 旧负载族、历史探针与覆盖检查的证据引用 })
  * $actual_cost (float * { 两版本实际消耗，按协议口径统计初始化、重复和失败 })

MetricSummary =
  Object
  * $confirmed_findings (float * { 有定位证据的独立确认短板数，按版本化根因归并规则计数 })
  * $unconfirmed_clusters (float * { 观察到行为差异但根因未确认的簇数，与已确认发现分别报告 })
  * $total_cost (float * { 该版本窗口实际总成本，含未被确认执行和无效实验 })
  * $family_coverage (dict[str, float] * { 各负载族的执行与确认分布，用于发现覆盖收缩 })

PromotionDecision =
  Oneof
  | $promoted (Promoted * { 信用与保留检查满足协议，候选快照成为后续基础版本 })
  | $reverted (Reverted * { 明确损失或协议失败，恢复基础快照并保留完整诊断足迹 })
  | $held (Held * { 无结论或证据不足，保持基础版本，候选延期待新一轮对照 })

Promoted = $reference (str * { 晋升记录，说明生效的基础版本和适用条件；晋升不是永不退化承诺 })
Reverted = $reference (str * { 恢复记录及原因，历史候选不删除 })
Held = $reason (str * { 待决原因和下一次对照条件，避免把未知当成默认拒绝 })
```

## 基本操作

1. `Record` 追加一条原始判定或补录真值。条目标识与判定版本固定，同一事件的结果不覆盖既有预测。
2. `Fetch` 按命题、判定工件、协议和特征取回可用历史，过滤未归因、无效和缺少入选概率的记录。判定组件用这批案例调用 `verdict.Calibrate`；缺少可比证据、迁移未经验证或校准偏差大时得到低置信。
3. `Evaluate` 在冻结协议下统计候选与基础版本的发现、覆盖和保留情况，报告实际成本。协议被候选版本改动或任务不相匹配时返回对照无效。
4. `Attribute` 将窗口证据与原始判断、选择路径对照，给出信用结论。探索救回的错误不能被记作预测正确；无结论不能折中成半个收益。
5. `Promote` 根据信用与协议决定晋升、恢复或待决，并记录版本切换。晋升不冻结物理观测；恢复也不回滚其他更新已产生的有效证据。

## apis.viba

```viba apis
import feedback.types as feedback
import verdict.public as verdict
import dedup.public as dedup

Record =
     str
  <- $store (str * { 经验库位置，包含原始证据和版本化工件引用 })
  <- $record (feedback.JudgmentRecord * { 判定、选择路径与本条自身真值。真值未到时如实写 pending，不借别组件的标签 })
  <- {
    追加不可变记录并返回标识。真值到达只新增关联记录，不覆盖原始预测；写入后要能按判定版本、协议和入选概率检索
  }

Fetch =
     list[verdict.CalibrationCase]
  <- $store (str * { 经验库位置，按判定版本、协议和特征检索原始记录 })
  <- $proposition (str * { 所需命题的完整定义，避免把三种判断的案例混成一批 })
  <- $judge_version (str * { 判定工件版本；迁移来源需另行标注已验证的适用范围 })
  <- $protocol (str * { 验证协议版本，标签语义不同就不能同批返回 })
  <- {
    返回可用历史案例，保留原始预测、入选概率与证据引用。未归因、无效执行和延期记录不进入集合；返回空列表时如实表示无经验，不补造案例
  }

Evaluate =
     feedback.WindowEvaluation
  <- $protocol (str * { 冻结的窗口口径、预算、晋升阈值和保留约束 })
  <- $base (str * { 基础快照及其窗口运行记录 })
  <- $candidate (str * { 候选快照及其同任务窗口运行记录 })
  <- {
    按匹配成本统计两版本的发现、未确认簇、覆盖和保留能力，报告实际总成本。协议或任务不匹配返回对照无效，不用总数上涨掩盖覆盖收缩
  }

Attribute =
     feedback.Credit
  <- $evaluation (feedback.WindowEvaluation * { 版本对照证据，包含原始记录引用 })
  <- $original_judgments (list[verdict.Judgment] * { 候选试行前的审核判断与选择路径，用于区分预测错误和探索代价 })
  <- {
    给出标准化净效应与信用结论。误放、误拒绝从原始判断和后续对照推出，不让作者自认；证据不足返回无结论，探索救回不计作预测命中
  }

Promote =
     feedback.PromotionDecision
  <- $credit (feedback.Credit * { 本次信用结论及其对照证据 })
  <- $repository (str * { harness 工件仓库位置，用于切换基础快照或恢复 })
  <- {
    按协议晋升候选、恢复基础版本或保留待决，并记录切换。物理观测不因版本切换被改写，恢复不删除候选差异与诊断记录
  }
```
