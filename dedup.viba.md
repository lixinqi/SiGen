去重器（dedup）

## 需求背景

去重器减少没有增量信息的重复实验。测试的源码形态、业务行为与内部根因是三种不同对象：变量名变化可能没有行为变化；相同访问逻辑的参数跨过边界可能形成重要差异；不同访问逻辑也可能命中同一根因。运行前去重不负责证明根因相同。

去重位于预判后，但不只查看预判为真的候选。它对全部候选与历史代表、同批代表的关系留下记录，供调度联合选择。参考集合有版本，模型如果事后换代表集合，就改变了原判断的命题。候选不能因为去重器一句重复而永久丢失执行机会。

value 表示候选相对指定代表提供非冗余实验信息的概率；confidence 表示这类关系判断的经验支持。输出还包含关系描述：冗余重复、受控变体、语义新样本。关系描述用于解释比较设计，不等于不可撤销的标签。受控变体可能最终没有新差距，也可能揭示性能突变，不能无条件说所有单参数改动都有价值。

去重器的反馈需要额外对照。单个候选出现短板不能证明非冗余，两个候选命中同一根因也不能证明冗余。检验需候选和代表在可比条件下的实际轨迹、时序、状态变化和性能响应；轨迹不足或条件不匹配时返回无结论。行为相似度、信息增量定义及阈值应版本化，避免去重器以改标签规则提升自己的成绩。

根 README 从这里取 `Assessment`、`Validation`、`Assess` 与 `Validate`。

## public.viba

```viba public
import dedup.types as dedup
import dedup.apis as dedup_api

Assessment = dedup.Assessment
Validation = dedup.Validation
Assess = dedup_api.Assess
Validate = dedup_api.Validate
```

## types.viba

运行前的关系是预测，运行后的 `Validation` 是可用证据结论。候选相对代表的新颖性只在该比较范围内成立；不声称构造了整个程序空间上可证明的语义等价关系。

```viba types
import verdict.public as verdict

Assessment =
  Object
  * $judgment (verdict.Judgment * { 非冗余信息命题的预测记录，subject 同时绑定候选与代表集合。value 越大越支持为候选付出额外验证成本 })
  * $reference_set (str * { 比较代表集合的不可变版本，包含历史和同批代表。空集合允许冷启动，但不构成已证明新颖 })
  * $relation (Relation * { 运行前推断的逻辑关系，给调度提供聚类和成对比较依据，不能独自生成真实标签 })
  * $verification_protocol (str * { 运行后判断行为证据能否互相替代的定义、维度和阈值，结果出现后不得临时改变 })

Relation =
  Oneof
  | $redundant (Redundant * { 预期轨迹、状态和目标基本可替代，可优先执行代表并对候选保留审计机会 })
  | $controlled_variant (ControlledVariant * { 存在明确定义的基准对象和受控变化，适合成对运行验证参数响应 })
  | $novel (Novel * { 预期行为不能由参照证据替代，仍需实际运行检验 })

Redundant = $representative (str * { 提供替代证据的代表对象及观测引用；没有已验证代表时不能声称无需运行即可获得候选真值 })
ControlledVariant =
  Object
  * $base_candidate (str * { 受控对照的基准对象，要能重建其版本、参数和状态 })
  * $changed_dimension (str * { 被改变的变量及变化值，如并发、状态或数据规模。实验还应确认其他关键条件可比 })
  * $expected_boundary (str * { 希望检验的响应变化或未知边界，写作假设而非未经证明的内部路径切换 })
Novel = $difference (str * { 与最近代表的明确行为差异，说明访问、并发或持久化语义，不以代码字面差异冒充新颖性 })

Validation =
  Object
  * $assessment (Assessment * { 结果出现前的原始去重判断，比较对象与协议不被后续修改 })
  * $evidence (list[str] * { 候选与代表的可比原始观测引用，包括实际轨迹与响应。只有候选一侧证据时通常不足以检验冗余 })
  * $result (ValidationResult * { 非冗余、冗余或无结论；用于去重器校准，不冒充根因归类 })

ValidationResult =
  Oneof
  | $nonredundant (Nonredundant * { 有效行为对照支持候选提供代表无法替代的信息，可为去重命题赋正标签 })
  | $redundant (VerifiedRedundant * { 对照支持在约定维度与容差内可替代，可赋负标签；不证明所有未来条件下等价 })
  | $inconclusive (InconclusiveValidation * { 缺少代表、轨迹、可比条件或区分证据，保留未知 })
Nonredundant = $evidence (str * { 支持新增行为或参数响应信息的对照差异及适用范围，不仅是同一根因的名字不同 })
VerifiedRedundant = $evidence (str * { 支持替代性的对照证据及容差，不从一次结果相同推出普遍等价 })
InconclusiveValidation = $reason (str * { 验证不足的具体原因，决定是否需要执行代表或追加轨迹，不能填作冗余负标签 })
```

## 基本操作

1. `Assess` 从候选语义、参数与初始状态寻找历史和同批代表，输出关系预测、非冗余概率及经校准的置信度。
2. 调度按比较关系选择代表、受控变体和审计对象；成对验证需要的总成本一并计入预算。模型判冗余不删除原始候选。
3. `Validate` 根据候选与代表的实际可比观测，按冻结协议检验信息是否可替代。无法比较时返回无结论。
4. 去重器通过误合并与误拆分证据改进。根因合并或修复验证由额外定位证据处理，不能倒推运行前两个程序必然重复。

## apis.viba

```viba apis
import dedup.types as dedup
import generator.public as generator
import measure.public as measure
import verdict.public as verdict

Assess =
     dedup.Assessment
  <- $candidate (generator.Candidate * { 待比较的完整行为对象，代码与参数、并发和状态一起参与比较 })
  <- $representatives (list[generator.Candidate] * { 已有和同批代表的可复核完整内容，不只传代表名字让操作暗读隐藏状态 })
  <- $reference_version (str * { 参照集合版本，锁定该判断的比较范围 })
  <- $verification_protocol (str * { 非冗余信息的验证维度和容差，避免在物理结果后任意改定义 })
  <- $artifact (str * { 去重器工件和关系模型版本，可由后续审核更新 })
  <- $history (list[verdict.CalibrationCase] * { 去重命题的有效行为对照案例，不混入只有性能阳性却无代表证据的记录 })
  <- {
    给出候选相对代表集合的非冗余概率与关系解释，校准经验置信度。代码相似与根因相同均不能直接推导冗余；受控变体保留成对验证依据，返回原始判断而非删除动作
  }
  <- $calibrate (verdict.Calibrate * { 返回同类行为关系判断的经验可靠性，填入 confidence 并保存所用历史引用 })

Validate =
     dedup.Validation
  <- $assessment (dedup.Assessment * { 原始判断，固定比较对象、参照和协议 })
  <- $observations (list[measure.Observation] * { 候选与代表的实际可比观测；原始轨迹不足或环境不同不能硬给标签 })
  <- {
    按既定非冗余协议检验行为证据能否互相替代。输出明确正负或无结论，保存比较依据；不以同一根因或一次相同结果替代行为验证，不让未运行对象获得虚构标签
  }
```
