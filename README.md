# SiGen

SiGen 自动生成测试，找出两个系统之间真实的性能差距，并用跑出来的结果训练生成器。

仓库按版本组织：每个 `vN/` 是一版系统的接口，`metric/` 是各版共用的读数，`evaluate/` 是各版共用的评估。加一版不影响评估，评估也不依赖任何一版。

## 怎么评估

一轮训练里，每次尝试是一个生成的测试按协议测一次；测出差距记有效。一轮的有效率是：

    有效率 = 有效结果数 / 尝试数

训练跑多轮，得到一串有效率。评估把它们收成两个数：

    轮次平均 = 各轮有效率相加 / 轮数
    平均退化 = (1 / (n-1)) * Σ relu(上一轮 - 这一轮)

- **轮次平均**：训练期间的平均水平，越高越好。
- **平均退化**：相邻两轮下降量的平均，只算下降；越低越稳。

所有读数都是两个整数的分数，不用 float：有效率是 `有效数/尝试数`，平均与平均退化也是分数。精确，不引入舍入误差。

读的时候两个一起看：平均涨了、退化也涨了，说明训练不稳；单看某一轮不够。

## 为什么能有很多版本

`metric/` 只定义读数（`EffectiveRate`、`Rational`、`TrainingSummary`），`evaluate/` 只实现两个数的计算。它们不依赖任何一版系统。

每个 `vN/` 只依赖 `metric/`：

- `v1/` 已有：`si_train` 每轮数出 `EffectiveRate`，追加进历史，`TrainRound` 把观测与历史带回来；
- 以后加 `v2/`，同样把每轮 `EffectiveRate` 交给 `evaluate`，评估不用改；
- 跨版本比较用同一套 `metric` 与 `evaluate`，不另写口径。

版本之间不互相依赖；`v1` 不知道 `v2`，也不依赖 `evaluate`。依赖只有两条：`vN → metric`，`evaluate → metric`。

## 目录

- [`metric/metric.viba.md`](metric/metric.viba.md)：共用的读数定义。
- [`evaluate/evaluate.viba.md`](evaluate/evaluate.viba.md)：把各轮有效率算成轮次平均与平均退化。
- [`v1/README.md`](v1/README.md)：第一版在做什么。
- [`v1/README.viba.md`](v1/README.viba.md)：第一版的接口。

## 加一版

1. 建 `vN/README.md` 与 `vN/README.viba.md`；接口只 import `metric`，不 import 别的版本，也不 import `evaluate`。
2. 训练步每轮产出一个 `metric.EffectiveRate`，留住各轮历史。
3. 用 `evaluate.Evaluate` 算 `TrainingSummary`。不要为本版改 `metric` 或 `evaluate`；读数定义真要变，单独说明。
