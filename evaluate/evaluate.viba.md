# 评估（evaluate）

## 需求背景

把一轮尝试的结果算成一个有效率：有效条数除以总条数。有效指测量确认了差距；其余都不算。

## public.viba

```viba public
import evaluate.apis as evaluate_api

Evaluate = evaluate_api.Evaluate
```

## types.viba

这一层不新增类型：输入是每次尝试是否有效，输出是 `metric.EffectiveRate`（分子与分母）。

## 基本操作

`Evaluate` 数有效条数，写成分数；空输入是程序错。

## apis.viba

```viba apis
import metric.public as metric
```

### Evaluate

数有效条数，写成分数。

```viba apis
Evaluate =
     metric.EffectiveRate
  <- $results (list[bool] * { 每次尝试是否得到有效结果，一条一个 })
  <- { 数 true 的条数作为分子，总条数作为分母；空列表是程序错；false 不算有效 }
```
