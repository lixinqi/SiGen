# 回流与评估

## 需求背景

平台里有两个时间尺度的循环。快循环每跑一次实验就有物理真值；慢循环一次更新要经历一个观察窗口才能评估，数据稀疏。两个循环的证据都汇入同一个经验库：快循环的判定档案回答“这类判断通常有多准”，慢循环的窗口评估回答“这次更新造成了什么”。本篇（feedback）定义这两条回流路径共用的设施。

校准回答的是置信度从哪来的问题。置信度不能由组件自报——一个组件完全可以在没有经验时依然输出高置信度，探索机制随即失效。校准操作从历史判定档案里统计相似案例的数量、结果一致性与历史误判率，算出本次判断应有的置信度。判定值学习“对象值不值得”，置信度学习“我关于这类对象的判断通常有多可靠”，两者都由后续真值持续修正。

信用归因回答的是更新的净影响。归因的主指标只有一个：每单位真实实验成本发现的独立、可复现、典型且可定位的短板根因数量。“没有退化”不能单独作为成功标准，因为永远不更新就能实现它；“预测准确率高”也不能，因为系统可能只在自己熟悉的一小片空间上准确。归因结论分四种：判定被证实、误放、误杀、置信度虚高。一次窗口引入多个修改时，靠回放与消融补做归因，否则学到的只是“某段时间系统变好了”，而不是“哪次更新为什么有效”。

本篇定义信用 `Credit`、窗口评估 `Evaluation`，以及三个操作：写入档案 `Record`、校准置信度 `Calibrate`、窗口评估 `Evaluate`。根 README 的 `RunExperiment` 与 `ReviewUpdate` 依赖本篇的 `Record` 与 `Evaluate`，预判、去重、审核三篇文档的判定操作都依赖本篇的 `Calibrate`，更新报告引用本篇的 `Credit`，所以这篇有 public：根 README 从这里取 `Credit`、`Evaluation`、`Record`、`Calibrate` 与 `Evaluate`。

## public.viba

```viba public
import feedback.types as feedback
import feedback.apis as feedback_api

Credit = feedback.Credit

Evaluation = feedback.Evaluation

Record = feedback_api.Record

Calibrate = feedback_api.Calibrate

Evaluate = feedback_api.Evaluate
```

## types.viba

信用结论是个和类型：证实、误放、误杀、置信度虚高是四种结局，各带各的证据。窗口评估只装三件事：主指标、同预算对照、旧能力是否退化——其他的辅助指标都服务于对这三件事的解释。

```viba types
CreditKind =
  Oneof
  | $confirmed (Confirmed * { 判定被证实：窗口评估的结果方向与当时的判定值一致 })
  | $misaccept (Misaccept * { 误放：接受了坏更新，窗口内发现能力下降或旧能力退化 })
  | $misreject (Misreject * { 误杀：拒绝了好更新，后续回放或探索性试行证明它本有收益 })
  | $miscalibrated (Miscalibrated * { 置信度虚高：判定碰巧没错，但它声称的经验支撑并不存在；这种结局不伤害本轮结果，却污染之后的探索决策 })

Confirmed =
  Object
  * $margin (float * { 判定与结果的差距：差距越小，校准越可靠；长期系统性偏差说明判定值的刻度需要修正 })

Misaccept =
  Object
  * $symptom (str * { 退化的具体表现：哪个负载族、哪项指标、从什么时候开始；复盘要能据此定位到更新里哪一段逻辑被利用 })

Misreject =
  Object
  * $evidence (str * { 证明本有收益的记录标识：哪一次抽样回放或探索性试行推翻了当时的拒绝；没有这条记录，误杀永远只是猜测 })

Miscalibrated =
  Object
  * $claimed (float * { 当时声称的置信度 })
  * $actual (float * { 同类历史档案里的实际判准率；两者的差距就是校准误差，直接修正下一次置信度 })

Credit =
  Object
  * $update (str * { 被归因的更新标识 })
  * $effect (float * { 净影响：窗口内主指标相对同预算对照的变化，折算成与判定值同刻度的数值；它是判定值的对答案 })
  * $kind (CreditKind * { 归因结论：证实、误放、误杀或置信度虚高，各带各的证据；结论回流给审核器，成为它校准自己的材料 })

Evaluation =
  Object
  * $metric (float * { 主指标：窗口内每单位真实实验成本发现的独立、可复现、典型且可定位的短板根因数量 })
  * $baseline (float * { 同预算下“生成后全部真实执行”的对照指标；主指标必须超过它，整套调度与预判才有存在价值 })
  * $regressed (bool * { 旧能力是否退化：旧负载族与历史探针上的表现是否下降；不退化是底线，不是功劳 })
```

## 基本操作

三个操作，分别对应经验库的写、读与评估。

1. 写入档案：把一条完整判定档案写进经验库，返回档案标识；之后一切校准与归因只认这里写下的记录。
2. 校准置信度：给出组件的历史判定档案与当前对象，交回本次判断应有的置信度。
3. 窗口评估：给出观察窗口，交回主指标、同预算对照与旧能力退化情况。

## apis.viba

```viba apis
import feedback.types as feedback
import verdict.public as verdict

Record =
     str
  <- $judgment (verdict.Judgment * { 一条完整判定档案：判定、决定与真值三样齐全；真值未到的先按 pending 写入，到达后补全 })
  <- $store (str * { 经验库路径 })
  <- {
    写入经验库：档案一旦写入不可改写，真值到达只追加不覆盖；返回档案标识，校准、归因与争议复核都凭它回查
  }

Calibrate =
     float
  <- $history (list[verdict.Judgment] * { 该组件在同类对象上的历史判定档案：相似案例的数量、结果一致性与历史误判率都从这里统计 })
  <- $subject (str * { 当前对象的特征摘要：负载族、代码区域或更新类型，用来圈定“同类” })
  <- {
    校准置信度：相似案例越少、历史误判越多、结果越不一致，返回值越低；当前对象落在从没判过的区域时返回接近零的值；输出供组件填进判定，组件不得自报置信度
  }

Evaluate =
     feedback.Evaluation
  <- $store (str * { 经验库路径：窗口内的判定档案、物理真值与更新记录都从这里读 })
  <- $window (int * { 观察窗口长度：统计多少轮实验 })
  <- {
    窗口评估：统计窗口内的主指标，与同预算下“生成后全部真实执行”的对照比较，并检查旧负载族与历史探针上的表现是否退化；一次窗口引入多个修改时，靠回放与消融补做归因，不把整体变化笼统记在某一次更新头上
  }
```
