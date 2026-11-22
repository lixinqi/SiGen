# 轨迹与驾驶（trajectory）

## 需求背景

Agent 的每次经历留下一条轨迹：提示词、回答、环境结果。运行时用 [transfer_table](../transfer_table/transfer_table.viba.md) 的状态机导航：从当前累积状态列出候选局部状态，让 ACP 会话从中选一个，把这一步的做法写进临时 wiki，发提示词；回答落进轨迹后，用 `goto_next` 走到下一累积状态。

轨迹本身只记录发生过的步骤；状态的类型、转移表和导航都在 transfer_table 里。这个模块只负责“选下一步、发出去、把回答收回来、走一步”。

## public.viba

```viba public
import trajectory.types as trajectory
import trajectory.apis as trajectory_api

Prompt = trajectory.Prompt
Answer = trajectory.Answer
EnvResult = trajectory.EnvResult
Trajectory = trajectory.Trajectory
NextStep = trajectory.NextStep

get_next_step_tmp_wiki_and_prompt = trajectory_api.get_next_step_tmp_wiki_and_prompt
drive_agent_like_human_by_send_prompt_and_tell_tmp_wiki_path = trajectory_api.drive_agent_like_human_by_send_prompt_and_tell_tmp_wiki_path
```

## types.viba

轨迹的三种步骤用各自的 tag 区分；`NextStep` 把“下一步的临时 wiki、提示词、选中的局部状态”绑在一起，驾驶时三样一起用。

```viba types
import transfer_table.public as transfer_table
```

### Prompt

发给 agent 的提示词。

```viba types
Prompt =
     tagged[__type__, Object]
  * $text (str * { 提示词原文 })
```

### Answer

agent 对一条提示词的回答。

```viba types
Answer =
     tagged[__type__, Object]
  * $text (str * { 回答原文 })
```

### EnvResult

环境在一次操作后返回的结果。

```viba types
EnvResult =
     tagged[__type__, Object]
  * $text (str * { 环境返回的原始结果 })
```

### Trajectory

一次经历：按时间顺序排列的提示词、回答与环境结果。

```viba types
Trajectory = list[Prompt | Answer | EnvResult]
```

### NextStep

下一步的三样东西：临时 wiki 的路径、发给 agent 的提示词、选中的局部状态。

```viba types
NextStep =
  Object
  * $tmp_wiki_file_path (str * { 这一步的临时 wiki 路径 })
  * $prompt (str * { 下一步发给 agent 的提示词 })
  * $local_state (transfer_table.LocalStateInterface * { 选中的局部状态；下一步用它走转移表 })
```

## 基本操作

1. `get_next_step_tmp_wiki_and_prompt` 用 `transfer_table.get_candidate_next_local_states` 从当前累积状态拿候选局部状态；调用 `judge_candidate_local_states` 让 ACP 会话给候选打分；选置信度最高的候选，把它的做法写进临时 wiki，生成一条提示词，返回 `NextStep`。
2. `drive_agent_like_human_by_send_prompt_and_tell_tmp_wiki_path` 先取 `NextStep`；把临时 wiki 的路径并进提示词，调用 `viba_acp.send_prompt` 发给会话；回答由 `$on_update` 收进轨迹；再用 `transfer_table.goto_next` 走一步，返回新的累积状态。

## apis.viba

```viba apis
import trajectory.types as trajectory
import transfer_table.public as transfer_table
import viba_acp.public as viba_acp
```

### judge_candidate_local_states

让 ACP 会话给候选局部状态打分。

```viba apis
judge_candidate_local_states =
     list[(transfer_table.LocalStateInterface, $confidence float)]
  <- $env Env
  <- $acp_session (viba_acp.Session * { 当前会话，轨迹从中读取 })
  <- $candidates (list[transfer_table.LocalStateInterface] * { get_candidate_next_local_states 交回的候选 })
  <- {
    把候选与当前轨迹交给 agent，让它给出每个候选的置信度；用 viba_acp.send_prompt 提问并收回答，候选按置信度从高到低排
  }
```

### get_next_step_tmp_wiki_and_prompt

从当前累积状态选出下一步，写出临时 wiki 与提示词。

```viba apis
get_next_step_tmp_wiki_and_prompt =
     trajectory.NextStep
  <- $env Env
  <- $acp_session (viba_acp.Session * { 当前会话，用来读候选并让 agent 打分 })
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- $cum_state (transfer_table.CumulativeStateInterface * { 当前累积状态 })
  <- {
    用 transfer_table.get_candidate_next_local_states 拿候选；调用 judge_candidate_local_states 让 agent 打分；选置信度最高的候选，把它对应的做法写进临时 wiki 文件，并给出一条下一步提示词；返回 NextStep
  }
```

### drive_agent_like_human_by_send_prompt_and_tell_tmp_wiki_path

像人一样驱动 agent：发提示词，并告诉它临时 wiki 的路径。

```viba apis
drive_agent_like_human_by_send_prompt_and_tell_tmp_wiki_path =
     transfer_table.CumulativeStateInterface
  <- $env Env
  <- $acp_session (viba_acp.Session * { 要驱动的 ACP 会话 })
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- $cum_state (transfer_table.CumulativeStateInterface * { 当前累积状态 })
  <- $on_update (void <- $env Env <- $notification (viba_acp.SessionNotification * { 会话通知 }))
  <- $sub_env_prefix_on_update (str * { 通知回调的子环境名前缀 })
  <- {
    先向 get_next_step_tmp_wiki_and_prompt 要下一步；把临时 wiki 的路径并进提示词，调用 viba_acp.send_prompt 发给会话；回答由 $on_update 收进轨迹；再用 transfer_table.goto_next 走一步，返回新的累积状态
  }
```
