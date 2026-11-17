# 评估（evaluate）

## 需求背景

把训练各轮的有效率收成两个数：轮次平均与平均退化（相邻两轮下降量的平均）。有效指测量确认了差距；其余都不算。

## public.viba

```viba public
import evaluate.apis as evaluate_api

Evaluate = evaluate_api.Evaluate
```

## types.viba

这一层不新增类型：输入是各轮的 `metric.EffectiveRate`，输出是 `metric.RoundEvaluation`。

## 基本操作

`Evaluate` 收各轮有效率，返回轮次平均与平均退化；空列表是程序错，单轮的平均退化记 0。

## apis.viba

```viba apis
import metric.public as metric
```

### Evaluate

算轮次平均与平均退化。

```viba apis
Evaluate =
     metric.RoundEvaluation
  <- $rates (list[metric.EffectiveRate] * { 每轮训练后的有效率，按轮次排 })
  <- { 平均：各轮相加除以轮数；平均退化：相邻两轮下降量的平均，relu(r_{i-1} - r_i)（负数取 0，只算下降），单轮记 0；空列表是程序错 }
```
