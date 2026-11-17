# 度量（metric）

## 需求背景

一轮尝试的有效率等于有效结果数除以尝试数。有效指测量确认了差距；其余都不算。

## public.viba

```viba public
import metric.types as metric

EffectiveRate = metric.EffectiveRate
```

## types.viba

### EffectiveRate

有效结果数除以尝试数，写成分数：`effective` 是分子，`attempts` 是分母。

```viba types
EffectiveRate =
  Object
  * $effective (int * { 有效结果数，分子 })
  * $attempts (int * { 尝试数，分母；大于零 })
```

## 基本操作

这一层只有一个类型，没有操作；算法在 [evaluate](../evaluate/evaluate.viba.md) 的 `Evaluate` 里。
