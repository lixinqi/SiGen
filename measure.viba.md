环境评估（measure）

## 需求背景

环境评估把 Generator 的产物交给真实环境执行，保存原始反馈，并按事先确定的协议判断产物是否有效。本模块实现比较型评估：同一业务语义在两个环境各执行一次，用相对差距表达有效性。单环境评估等形式只需替换环境组合与协议中的评估形式，上层命题与结论结构不变。

“没有确认”有不同含义：有效实验充分支持未达阈值，是可用的负标签；实验不稳定或证据不足，是无结论；构建失败、工作不等价和环境失效，是无效执行。三者不能合成一个 false 写进预判器经验，否则它会把工具或环境故障学成“系统没有问题”。

差距同时需要统计证据和实际意义。重复测量可能表明被测侧略慢，但差距低于业务阈值时不算本任务定义的发现。反过来，单次差距大但噪声更大时仍无结论。决定显著性的依据是实测数据和冻结协议，不读取预判器的差距或方差来调整判定门槛。

环境版本、硬件配置、数据状态与测试源码都绑定观测。相同初始状态指相同业务要求，不意味着两个环境必须有相同内部缓存结构。无法满足可比较条件时明确记录，不通过强行清缓存或隐藏初始化成本来假造公平。超时可以是性能证据，也可以是故障；是否作为删失观测使用由协议预先约定。

评估的产物是有效性结论，不是根因。确认的发现可以支撑预判器与生成器的更新，根因归类需要额外定位证据。

根 README 从这里取 `EvaluationSetup`、`Protocol`、`Observation`、`Finding` 与 `Measure`。

## public.viba

```viba public
import measure.types as measure
import measure.apis as measure_api

EvaluationSetup = measure.EvaluationSetup
Protocol = measure.Protocol
Observation = measure.Observation
Finding = measure.Finding
Measure = measure_api.Measure
```

## types.viba

观测外层区分有效测量与无效执行；有效测量内部区分确认发现、充分负证据和无结论。评估形式在协议中显式声明，比较型只是其中一种。差距与运行轨迹保留，不因最后没有进入发现集合而删除。

```viba types
EvaluationSetup =
  Object
  * $subject (str * { 被测对象及其准确版本、配置工件引用。版本变化必须重新检验历史结论，不能延用旧标签 })
  * $reference (str * { 对照或基线及版本、配置引用；单环境评估写空并说明判断依据。对照是角色，不预设成熟产品一定正确或最快 })
  * $conditions (str * { 硬件、部署、客户端、网络和资源隔离等可复核条件，说明匹配项与不可消除差异 })

Protocol =
  Object
  * $version (str * { 评估和有效性判定协议的版本，在看到该轮结果前确定。协议更新使用新版本，不改写历史标签语义 })
  * $form (EvaluationForm * { 本轮采用的评估形式，决定采样与对照方式 })
  * $metric (str * { 主指标和单位，如完成时间毫秒、每秒操作数。只有一个指标决定该轮命题，其他指标作为解释证据 })
  * $direction (MetricDirection * { 哪一侧数值增大意味着更差，显式避免吞吐与延迟符号相反 })
  * $practical_threshold (float * { 任务认定有实际意义的标准化阈值。与差距计算方法同尺度，不使用未经依据的固定百分比 })
  * $repetitions (int * { 重复测量的计划及最小样本要求引用，具体次数由任务和噪声决定，不以一次运行作为充分证据 })
  * $comparison_method (str * { 配对差距、置信区间或检验方法及多重比较、停止和复验规则。反复看结果直到显著须由序贯规则处理 })
  * $state_control (str * { 初始数据、预热、缓存条件、执行交错和漂移监测要求，两侧业务要求一致 })
  * $timeout_policy (str * { 超时、失败和删失数据的处理。不得在结果揭晓后把对己方有利的超时排除 })
  * $resource_limits (str * { 可写测试资源、时间、容量和并发上限。探索只在受控测试范围内，不使用生产数据或不可回滚资源 })

EvaluationForm =
  Oneof
  | $comparison (Comparison * { 同一业务语义在两个环境各执行一次，用相对差距表达有效性 })
  | $single_environment (SingleEnvironment * { 只有一个环境，按绝对指标与预定阈值判断；无法分离环境自身波动 })

Comparison = $pairing (str * { 两侧执行的配对、交错和可比条件，说明为何偏差可以在同一尺度上比较 })
SingleEnvironment = $threshold (str * { 绝对阈值及其来源，说明该阈值为何代表任务认可的可用水平 })

MetricDirection =
  Oneof
  | $higher_is_worse (HigherIsWorse * { 延迟或完成时间等指标，被测侧数值高是负向差距 })
  | $lower_is_worse (LowerIsWorse * { 吞吐等指标，被测侧数值低是负向差距 })
HigherIsWorse = $normalization (str * { 规定原始值到标准化 gap 的转换，gap 为正表示被测侧更差 })
LowerIsWorse = $normalization (str * { 规定原始值到标准化 gap 的转换，保持与延迟同一负向极性，不混用原始单位 })

Observation =
  Object
  * $identity (str * { 原始观测事件的唯一标识，关联候选、协议和环境版本 })
  * $candidate (str * { 完整实验对象的引用，不仅是程序名字 })
  * $setup (EvaluationSetup * { 被测、对照与条件的准确记录，支持版本漂移诊断 })
  * $protocol (Protocol * { 事先确定的评估和有效性判定规则，不受模型期望影响 })
  * $raw_evidence (list[str] * { 全部原始样本、标准输出、正确性检查、行为轨迹及资源记录引用；阴性、故障和无结论均保存 })
  * $actual_cost (float * { 实际消耗，与该任务预算同单位；初始化、重复、失败和复验按冻结口径计入 })
  * $result (MeasurementResult * { 有效观测或无效执行，不以单一布尔合并不同证据状态 })

MeasurementResult =
  Oneof
  | $valid (ValidMeasurement * { 语义和测量条件满足协议，可分析有效性与差距。有效不等于发现 })
  | $invalid (InvalidExecution * { 实验无法用于检验有效性命题，但故障记录仍可用于改进候选有效性或测量设施 })

ValidMeasurement =
  Object
  * $samples (list[Sample] * { 按协议采集的原始主指标，不只留下聚合均值；配对定义和执行顺序可复核 })
  * $gap_interval (list[float] * { 按冻结方法得到的标准化差距区间，包含两端及方法引用。正方向表示被测侧更差；区间含阈值时通常不能强行二分 })
  * $conclusion (Conclusion * { 对有效性命题的证据结论：充分支持、充分否定或尚无结论 })

Sample =
  Object
  * $subject (float * { 被测侧指标原始值，单位由协议给出 })
  * $reference (float * { 对照侧指标原始值；单环境评估写空并说明，与阈值比较 })
  * $execution_order (str * { 两侧交错顺序和实验区块引用，用于检查时间漂移与共享资源干扰 })

Conclusion =
  Oneof
  | $supported (Finding * { 有效证据支持产物成立，仍不证明根因 })
  | $not_supported (NotSupported * { 有效实验按预定规则充分支持未达阈值，才可作为预判器负标签 })
  | $inconclusive (Inconclusive * { 噪声、样本不足或临界区间导致无法判定。继续验证或保留未知，不能强制折算为零 })

Finding =
  Object
  * $candidate (str * { 暴露有效发现的完整实验对象引用，支持复现和参数边界定位 })
  * $gap (float * { 按协议计算的标准化稳定差距，正方向表示被测侧更差，原始两侧值仍可查 })
  * $conditions (str * { 结论成立的负载、版本、初始状态和环境条件。离开条件不作外推保证 })
  * $reproduction_evidence (list[str] * { 独立复验记录，不用一个可任意填 true 的字段代替复现实证 })

NotSupported = $evidence (str * { 支持未达阈值的有效测量及判定依据；未显著不必然支持否定，需遵守协议的充分证据规则 })
Inconclusive = $reason (str * { 尚不能区分阈值两侧的原因及后续验证要求，不等价于没有问题 })
InvalidExecution = $failure (ExecutionFailure * { 无效实验的具体失败类别，决定可否重跑及何种诊断任务可以使用记录 })

ExecutionFailure =
  Oneof
  | $invalid_test (InvalidTest * { 源码不可构建或两侧工作不等价、正确性失败 })
  | $invalid_environment (InvalidEnvironment * { 部署、资源或状态控制不满足实验条件 })
  | $unclassified_timeout (UnclassifiedTimeout * { 超时无法按预定规则归类为性能删失或工具故障 })
InvalidTest = $evidence (str * { 构建、执行、输出或状态校验的失败证据；不训练有效性概率 })
InvalidEnvironment = $evidence (str * { 环境或资源异常的证据及影响范围；系统版本与实验版本不能混淆 })
UnclassifiedTimeout = $evidence (str * { 超时原始日志和不能定性的原因，协议未覆盖时保留未知而非随意作负标签 })
```

## 基本操作

1. 校验产物是否可构建、两侧是否执行等价工作、测试资源及初始状态是否满足协议。失败时返回无效执行及完整原始记录。
2. 按冻结方案采集和重复测量，保存样本、状态、正确性结果、轨迹与真实成本。预判器输出不进入评估输入。
3. 按实测数据计算标准化差距与证据区间；分别返回确认、充分否定或无结论。没有显著差距并不自动构成充分否定。
4. 对确认的发现按协议复验并声明条件。行为去重和根因定位使用这些证据，但不是评估自动作出的结论。

## apis.viba

```viba apis
import measure.types as measure
import generator.public as generator

Measure =
     measure.Observation
  <- $candidate (generator.Candidate * { 源码、参数、初始状态、操作关系及正确性要求共同定义实验对象 })
  <- $setup (measure.EvaluationSetup * { 被测、对照与条件；对照为空时按单环境形式评估 })
  <- $protocol (measure.Protocol * { 在结果之前确定的评估形式、阈值、停止和失败处理规则 })
  <- {
    完成受控评估，保存全部原始记录和实际成本，依据实测噪声判断产物是否成立。返回有效观测的明确结论或未知，或无效执行的失败原因；不得读取模型预测来改变协议，不自行确定根因
  }
```
