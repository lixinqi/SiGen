# 度量（metric）

## 需求背景

训练每轮得到一个有效率：有效结果数除以尝试数。一轮训练的好坏看两个数：轮次平均，和平均退化（相邻两轮下降量的平均，越小越稳）。

## public.viba

```viba public
import metric.types as metric

EffectiveRate = metric.EffectiveRate
Rational = metric.Rational
RoundEvaluation = metric.RoundEvaluation
```

## types.viba

### EffectiveRate

一轮尝试的有效率，写成分数：`effective` 是分子，`attempts` 是分母。

```viba types
EffectiveRate =
  Object
  * $effective (int * { 有效结果数，0 到 attempts })
  * $attempts (int * { 尝试数，分母；大于零 })
```

### Rational

一个精确的有理数；轮次平均与平均退化都用它，不用 float。

```viba types
Rational =
  Object
  * $numerator (int * { 分子，可负 })
  * $denominator (int * { 分母；大于零 })
```

### RoundEvaluation

训练各轮有效率的两个读数：轮次平均，与平均退化（越小越稳）。

```viba types
RoundEvaluation =
  Object
  * $average (Rational * { 各轮有效率的算术平均 })
  * $degradation (Rational * { 相邻两轮有效率下降量的平均：平均 relu(r_{i-1} - r_i)，负数取 0，只算下降；越小越稳 })
```

## 基本操作

这一层只有三个类型，没有操作；算法在 [evaluate](../evaluate/evaluate.viba.md) 的 `Evaluate` 里。
