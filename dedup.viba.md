# 去重器

## 需求背景

没有语义去重的平台会自己骗自己：生成器发现某类测试容易确认短板，就批量改变量名、换循环结构、调表面参数，造出大量“新测试”；预判器在这些熟悉样本上保持高准确率，物理运行反复确认同一个已知短板，审核器看到发现数量上涨便认可更新。系统其实在不断重新发现同一个问题。去重器（dedup）补上的是实验空间的语义度量：两个测试“代码长得不同”与“探索了不同的物理行为”是两回事。

去重不是“相同／不同”的二元判断。代码相似的两个测试可能只差一个参数，却恰好跨过缓存冷热边界或元数据分片阈值，物理行为截然不同；反过来，两个用不同语言、不同封装写成的程序，可能产生几乎相同的文件系统访问轨迹。所以去重器把候选分成三类：冗余重复、受控变体、语义新样本，并把这个归类连同统一判定一起交回。它的判定值填“会带来新信息的概率”，方向与其他组件统一——越大越值得执行。被它折叠的候选不删除，留作复现、回归与预判器校准，只是不再计为新发现、不默认占用探索预算。

去重器自己也从物理反馈里学习：被它判为冗余的两个测试若真实运行表现出显著不同的访问路径，是误合并；被它保留为不同的测试若反复产生相同轨迹、相同瓶颈、相同根因，是误拆分。两类错误都靠抽样审计暴露，不靠它自己说了算。

本篇定义语义归类 `Class`、去重结果 `Assessment` 与去重操作 `Assess`。根 README 的 `RunExperiment` 依赖本篇的 `Assess`，所以这篇有 public：根 README 从这里取 `Assess`。

## public.viba

```viba public
import dedup.types as dedup
import dedup.apis as dedup_api

Class = dedup.Class

Assessment = dedup.Assessment

Assess = dedup_api.Assess
```

## types.viba

语义归类是个和类型：三种归类各带各的数据。冗余重复要指出它属于哪个已有逻辑簇，受控变体要指出它改了哪个变量，语义新样本要给出它与最近邻居的距离——三类各自需要的信息不同，塞进一个记录加标签是装不下的。

```viba types
import verdict.public as verdict


Class =
  Oneof
  | $redundant (Redundant * { 冗余重复：预期访问轨迹、系统状态与测量目标与某个已有样本基本相同，没有新增信息；折叠掉它，只保留代表 })
  | $controlled_variant (ControlledVariant * { 受控变体：主体逻辑与某个已有样本相同，只改了一个关键变量；这类近重复往往是最有价值的消融实验，跨过边界的突变点只能靠它们发现，不能过滤 })
  | $novel (Novel * { 语义新样本：访问模式、并发关系、持久化语义、初始状态或测量目标与已有样本确实不同，应优先保留 })

Redundant =
  Object
  * $representative (str * { 它所属逻辑簇的代表样本标识：折叠后由代表占用执行预算，它自己留作复现与校准材料 })

ControlledVariant =
  Object
  * $base (str * { 变体所基于的样本标识 })
  * $knob (str * { 被单独改变的那个关键变量：目录规模、并发度、缓存状态等；只改一个变量，差距的变化才能归因到这个变量上 })

Novel =
  Object
  * $distance (float * { 与最近已有样本的语义距离：调度拿它衡量探索价值，覆盖统计拿它发现还没人去过的区域 })

Assessment =
  Object
  * $verdict (verdict.Verdict * { 统一判定：判定值是“该候选会带来新信息”的概率，方向与其他组件统一，越大越值得执行；置信度由历史误合并与误拆分记录校准 })
  * $class (Class * { 语义归类：冗余重复、受控变体或语义新样本，各带各的数据；调度按归类决定折叠、保留对照还是优先执行 })
```

## 基本操作

只有一个操作：去重判定。给出一个候选测试与已有逻辑簇清单，交回统一判定与语义归类。置信度不允许自报，必须调用共享的校准操作，从历史误合并与误拆分记录里算出来。

## apis.viba

```viba apis
import dedup.types as dedup
import generator.public as generator
import verdict.public as verdict
import feedback.public as feedback

Assess =
     dedup.Assessment
  <- $candidate (generator.Candidate * { 候选测试：源码、负载族与生成依据 })
  <- $clusters (list[str] * { 已有逻辑簇及代表样本的标识清单：判定的参照系；判断依据包括访问轨迹、操作比例与顺序、初始状态、缓存冷热、并发时序与测量目标 })
  <- {
    判定新信息：先按访问轨迹与系统状态找邻居，再区分冗余重复、受控变体与语义新样本；只改一个关键变量的近重复必须认成受控变体保留下来；判定值填“会带来新信息”的概率，误合并与误拆分的历史决定置信度
  }
  <- $calibrate (feedback.Calibrate * { 校准置信度：它按历史误合并与误拆分记录交回本次判断应有的置信度；去重器不自报置信度，否则它可以在从没判对的区域里依然自信 })
```
