# 物理测量

## 需求背景

物理测量（measure）是环境：它不学习、不判断，只提供最终证据。平台里一切自进化的合法性都来自它的真值，所以它的测量协议必须先做扎实，否则组件学到的只是噪声与偏差。协议包括：两侧构造相同的初始数据与缓存状态，多轮重复并交错执行顺序以均摊缓存与机器负载的漂移，控制缓存冷热与数据状态，先做正确性校验再比性能，最后按重复测量噪声判断差距是否显著。

“确实为真”不是一次运行的二值标签。一个短板要成立，需要同时满足：两侧执行的是等价工作，配置、硬件与系统状态可比较，差距超过重复测量噪声，能按相同步骤复现，并且明确它的适用条件。严格说，测量发现的是“目标系统在该环境与负载下相对对照系统的稳定短板”，不直接等于证明目标系统存在内在实现缺陷。

未确认的结果同样重要。预判为真却未确认的候选，是预判器的假阳性；预判为假却被审计执行、结果确认的候选，是预判器的假阴性。两类结果都作为真值回流，缺了哪一类，校准都是偏的。

本篇定义确认短板 `Finding`、测量结果 `Result` 与测量操作 `Measure`。根 README 的 `RunExperiment` 依赖本篇的 `Measure`，实验报告引用本篇的 `Finding`，所以这篇有 public：根 README 从这里取 `Finding`、`Result` 与 `Measure`。

## public.viba

```viba public
import measure.types as measure
import measure.apis as measure_api

Finding = measure.Finding

Result = measure.Result

Measure = measure_api.Measure
```

## types.viba

测量结果是个和类型：确认短板与未确认是两种结局，各带各的数据。确认的要写清差距、根因归属与适用条件；未确认的要写清为什么不成立——原因不同，回流的用法也不同。

```viba types
Finding =
  Object
  * $test (str * { 暴露短板的候选测试标识 })
  * $gap (float * { 配对差距：目标相对对照的稳定差距，方向与幅度；它必须超过重复测量噪声才出现在这里 })
  * $root_cause (str * { 根因族标识：与已知短板按根因去重后的归属；新根因给新标识，已知根因的新表现归到旧标识，不重复计数 })
  * $conditions (str * { 差距成立的适用条件：负载、配置、缓存状态与数据规模；离开这些条件，结论不成立 })
  * $reproduced (bool * { 是否按相同步骤重复得到同方向差距；不可复现的差距不计为短板，但记录保留，供排查环境干扰 })

NoShortboard =
  Object
  * $cause (str * { 未确认的原因：差距未超过噪声、无法复现、两侧工作不等价，或正确性校验未通过；原因决定这条真值回流的用法——不等价的执行连噪声都算不上 })

Result =
  Oneof
  | $shortboard (Finding * { 确认短板：差距稳定、可复现、两侧等价，适用条件明确 })
  | $no_shortboard (NoShortboard * { 未确认：同样作为真值回流给预判器与去重器；缺了它，组件只能看到自己的假阳性被确认，看不到假阴性 })
```

## 基本操作

只有一个操作：配对测量。给出一个候选测试与目标、对照两侧的版本，按测量协议执行，交回确认或未确认的结局。测量不做任何筛选与判断，也不读取组件的判定——它的独立性是真值可信的前提。

## apis.viba

```viba apis
import measure.types as measure
import generator.public as generator

Measure =
     measure.Result
  <- $candidate (generator.Candidate * { 候选测试：源码与负载族 })
  <- $target (str * { 目标系统 A 及其版本 })
  <- $baseline (str * { 对照系统 B 及其版本 })
  <- {
    配对测量：两侧构造相同初始状态，多轮重复并交错执行顺序，控制缓存冷热；先做正确性校验确认两侧执行的是等价工作，再比性能；交回差距、噪声与是否成立；不读取任何组件的判定，测量结果不受期望影响
  }
```
