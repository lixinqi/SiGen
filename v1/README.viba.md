# SiGen 第一版接口

## 需求背景

给一个任务，生成候选测试，给每个候选取两个预测值（有效性、非冗余），再逐个测量，返回“候选、两个预测值、观测”。生成器、预判器、去重器和测量函数由调用方按当前工件绑好传进来；预算选择、记录库、更新审核还没有，等这一版跑出真实观测再逐层加。

生成器产出的是实验对象，不是配对本身；配对执行只是比较类任务里取得有效性证据的一种形式。预测不提供真值。

## public.viba

```viba public
import readme.types as readme
import readme.apis as readme_api

Candidate = readme.Candidate
Observation = readme.Observation
JudgedCandidate = readme.JudgedCandidate
ObservedCandidate = readme.ObservedCandidate
si_forward = readme_api.si_forward
si_train = readme_api.si_train
```

## types.viba

这一版只有四个类型。生成器、预判器、去重器和测量函数的具体参数（协议、工件、历史）都由调用方绑在这些函数里，接口只写它们之间传递的数据。

### Candidate

一个可运行的测试。源码与参数说明跑什么，初始状态说明从哪开始，正确性检查说明跑完必须成立什么，假设说明为什么它可能暴露差距。

```viba types
Candidate =
  Object
  * $source (str * { 测试程序源码与依赖 })
  * $parameters (dict[str, str] * { 数据规模、并发、请求分布等可复核参数 })
  * $initial_state (str * { 跑之前的数据与缓存状态 })
  * $correctness (str * { 跑完必须成立的条件 })
  * $hypothesis (str * { 为什么这个测试可能暴露差距 })
```

### Observation

一次测量的记录：这次尝试是否有效，以及原始数据、重复次数和成本。

```viba types
Observation =
  Object
  * $effective (bool * { 测量是否确认了差距；无结论和执行无效都记 false })
  * $raw (str * { 原始记录、重复测量与成本 })
```

### JudgedCandidate

测量前的候选：测试本身，加上有效性预测和非冗余预测。

```viba types
JudgedCandidate =
  Object
  * $candidate (Candidate * { 本轮生成的测试 })
  * $validity (float * { 有效性命题的预测概率，0 到 1 })
  * $redundancy (float * { 非冗余命题的预测概率，0 到 1 })
```

### ObservedCandidate

测量后的候选：测量前的部分，加上这次测量的观测。

```viba types
ObservedCandidate =
  Object
  * $judged (JudgedCandidate * { 测量前的候选与两个预测 })
  * $observation (Observation * { 这次测量的结果与原始记录 })
```

## 基本操作

`si_forward` 按任务生成候选，给每个候选取两个预测。它不测量、不删候选。

`si_train` 跑一次前向，对每个候选测量一次，把候选、预测和观测配在一起。它不做预算选择、不追加记录。

## apis.viba

```viba apis
import readme.types as readme
```

### si_forward

生成完整候选，再给每个候选取有效性预测与非冗余预测。它不测量、不删候选。

```viba apis
si_forward =
     list[readme.JudgedCandidate]
  <- $task (str * { 被测与对照环境的可比较能力、生成目标与禁止操作 })
  <- $generate ((list[readme.Candidate] <- $task str) * { 当前生成器；工件、经验、数量、覆盖依据与种子已绑在它里面 })
  <- $predict_validity ((float <- $candidate readme.Candidate) * { 当前预判器；命题、协议与历史已绑在它里面 })
  <- $assess_redundancy ((float <- $candidate readme.Candidate) * { 当前去重器；代表集合、命题、协议与历史已绑在它里面 })
  <- { 按任务生成候选，给每个候选取有效性预测值与去重预测值；不测量、不删候选 }
```

### si_train

跑一次前向，对前向交回的每个候选测量一次，返回配好的预测与观测。它不做预算选择、不追加记录。

```viba apis
si_train =
     list[readme.ObservedCandidate]
  <- $task (str * { 本轮任务，与前向收到的是同一份 })
  <- $forward ((list[readme.JudgedCandidate] <- $task str) * { 当前前向；生成器、预判器与去重器已绑好 })
  <- $measure ((readme.Observation <- $candidate readme.Candidate) * { 当前环境评估；被测、对照、协议与初始状态已绑好 })
  <- { 跑一次前向，对每个候选测量一次，返回配好的预测值与观测；不做预算选择、不追加记录 }
```
