预判器（predictor）

## 需求背景

预判器是对环境反馈的拟合。给定产物、系统版本与协议，它预测环境会给出什么反馈，并说明这个拟合在哪些区域还不可靠。它学习的是这些条件下的结果规律，不是无条件的物理世界模型，也不产出真值。版本改变、工作负载分布迁移或测量协议变更时，旧经验可能不再适用。

预测直接针对评估差距；比较型任务里就是配对差距，分别预测两侧绝对性能再相减可能累积误差，且任务关心的正是这个相对差距；具体是否采用联合模型由实现和对照实验决定。输出保留差距估计及区间，统一判定的 value 则对应明确的无效性命题，两种量不可互换。

经验不足与物理噪声分开处理。一个负载可以充分研究却仍有很大运行方差；另一个负载可能看起来稳定，却从未得到真实验证。confidence 表示前一种意义上的经验可靠性，预测区间表示对结果的分布估计。不能用预测方差为实测显著性设置门槛。

预判器只作判断，不对候选作最终过滤。所有合格候选的判定交给调度，低置信样本和高置信否决的审计样本获得执行机会。所观察的标签仍受到生成和选择策略影响，经验必须保留最终入选概率；无法覆盖的区域不能被报告成已证明没有漏报。

输入是完整候选、评估环境、协议、预测工件及可用历史。输出是原始判定和差距预报；差距预测若缺少尺度依据应声明未知，而不是随意填零。

根 README 从这里取 `Forecast` 与 `Predict`。

## public.viba

```viba public
import predictor.types as predictor
import predictor.apis as predictor_api

Forecast = predictor.Forecast
Predict = predictor_api.Predict
```

## types.viba

`Forecast` 包含一条完整判定档案和数值预报。判定命题将评估环境、协议与候选绑定；预测区间不参与物理真值的独立裁决。未知差距有自己的原因，不能冒充“预测无差距”。

```viba types
import verdict.public as verdict

Forecast =
  Object
  * $judgment (verdict.Judgment * { 命题为环境反馈将确认该产物有效，比较型任务即存在超过阈值的稳定相对短板；记录预测工件版本、可见特征和经验来源，value 是命题概率 })
  * $gap (GapForecast * { 标准化评估差距的数值预报或未知。与命题概率不同，正差距表示目标更差；未知不等于负预测 })

GapForecast =
  Oneof
  | $estimated (EstimatedGap * { 模型有依据给出差距分布估计，其误差由后续实测检验 })
  | $unknown (UnknownGap * { 缺少可比经验或尺度约定，保留未知，不生成伪精确数值 })

EstimatedGap =
  Object
  * $expected_gap (float * { 与评估协议同尺度的差距中心估计，比较型任务为配对差距，不能把绝对延迟与比例差距混用 })
  * $predictive_interval (list[float] * { 结果的预测区间及覆盖率由 uncertainty_model 规定。区间代表预测分布，不是实测差距置信区间，也不替代 confidence })
  * $uncertainty_model (str * { 预测区间的版本化计算依据，包括物理随机波动与模型不确定性的处理，后续检验覆盖率而非只看区间宽窄 })
UnknownGap = $reason (str * { 没有可比经验、系统版本未知或区间模型无法适用等明确原因，促使调度按经验不足处理 })
```

## 基本操作

1. 从候选源码、参数、状态和操作关系提取结果揭晓前可见的特征，绑定系统和协议版本。不得读入该候选的真实结果或根因结论。
2. 使用预测工件及可比历史估计有效性命题概率，同时形成差距预报。不同版本的证据迁移需要验证依据。
3. 调用 `Calibrate` 获得经验置信度，并保存其案例引用。冷启动或分布外对象可以有初步猜测，但不能有未经验证的高置信。
4. 返回预测档案，不执行测试也不删候选。物理结果由测量给出；预测工件更新由慢循环审核，不能在一批数据中一边读取结果一边覆盖原预测。

## apis.viba

```viba apis
import predictor.types as predictor
import generator.public as generator
import measure.public as measure
import verdict.public as verdict

Predict =
     predictor.Forecast
  <- $candidate (generator.Candidate * { 完整候选及结果揭晓前的操作、状态特征 })
  <- $setup (measure.EvaluationSetup * { 被测、对照与条件版本，确定预报适用范围 })
  <- $protocol (measure.Protocol * { 有效性命题、评估形式、阈值和差距尺度，不能在预报后改变 })
  <- $artifact (str * { 预测器代码、提示、模型及区间方法的版本化工件引用 })
  <- $history (list[verdict.CalibrationCase] * { 与有效性命题对应的有效历史标签，保留版本、相关簇、原始预测与入选概率 })
  <- {
    给出有效性成立概率和差距预报，调用校准获得经验置信度。保留预测前特征与历史来源，返回完整原始判断；没有数值预报依据时声明未知，不把模型输出作为实际测量阈值
  }
  <- $calibrate (verdict.Calibrate * { 返回该版本该命题的经验可靠性，本操作将其填入 confidence 并记录实际使用的案例来源 })
```
