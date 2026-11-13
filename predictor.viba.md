# 预判器

## 需求背景

真实实验是平台里最贵的资源：部署环境、构造数据状态、在对照系统上跑同一份测试、重复测量，都消耗机器与时间。预判器（predictor）是环境模型：它预测“如果这样试，会看到什么物理结果”，让调度把预算花在最有信息量的候选上。它直接预测配对差值——相同语义与环境下，目标系统相对对照系统差多少、这个差距稳不稳——而不是分别预测两侧的绝对性能，因为平台关心的从来不是单侧快慢，而是差距。

预判器最大的风险是变成硬过滤器：只跑它预判为真的候选，被否决的候选永远没有真值，假阴性无声积累，生成器逐渐迎合它的偏好，它在自己筛出来的样本上准确率越来越高，真实发现覆盖却越来越小。所以预判器输出的是统一判定接口：判定值填“会出现稳定短板的概率”，置信度由同类负载上的历史误判校准，调度权交给上层。它否决的候选里，总有一部分因为低置信探索或随机审计而被真实执行，它的错误因此可见、可改。

本篇定义预判结果 `Forecast` 与预判操作 `Predict`。根 README 的 `RunExperiment` 依赖本篇的 `Predict`，所以这篇有 public：根 README 从这里取 `Predict`。

## public.viba

```viba public
import predictor.types as predictor
import predictor.apis as predictor_api

Forecast = predictor.Forecast

Predict = predictor_api.Predict
```

## types.viba

预判结果在统一判定之外多带两个数：预测的配对差距与它的方差。统一判定给调度用，这两个数给测量与事后分析用——真值判定拿方差做显著性对照，校准拿差距检验预判器的数值质量。

```viba types
import verdict.public as verdict


Forecast =
  Object
  * $verdict (verdict.Verdict * { 统一判定：判定值是“该候选在目标上相对对照出现稳定短板的概率”，置信度由同类负载的历史判定档案校准；调度只读这个字段 })
  * $gap (float * { 预测的配对差距：相同语义与环境下目标相对对照的性能差距，方向与大致幅度；调度用它区分“高价值候选”与“高不确定候选”，测量用它与实测差距对照 })
  * $spread (float * { 预测差距的方差：对重复测量噪声的预判；真值判定拿它做显著性对照——实测差距要超过噪声才算数 })
```

## 基本操作

只有一个操作：预判。给出一个候选测试，交回统一判定与预测的配对差距。置信度不允许自报，必须调用共享的校准操作，从同类负载的历史判定档案里算出来。

## apis.viba

```viba apis
import predictor.types as predictor
import generator.public as generator
import verdict.public as verdict
import feedback.public as feedback

Predict =
     predictor.Forecast
  <- $candidate (generator.Candidate * { 候选测试：源码、负载族与生成依据 })
  <- {
    预判物理结果：预测配对差距的方向与幅度，给出方差；把“会出现稳定短板的概率”填进判定值；候选落在没见过的负载区域时，置信度必须随相似案例的减少而降低，不能靠语气自信
  }
  <- $calibrate (feedback.Calibrate * { 校准置信度：它按同类负载的历史判定档案交回本次判断应有的置信度；预判器不自报置信度，自报的置信度在没经验时也会虚高 })
```
