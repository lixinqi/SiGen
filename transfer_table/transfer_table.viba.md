# 状态转移表（transfer_table）

## 需求背景

从轨迹里学到的经验，最终要落成一套能导航的状态机：项目在自己的专门目录里定义 local state 与 cumulative state 的具体类型，每个 cumulative state 用 `goto_table` 指向一份泛型转移表。SiGen 不定义业务状态，只提供两个接口、两个 schema 读取、一份泛型转移表的约定，以及两条导航操作：从当前累积状态列出候选的下一步局部状态，以及选中一个局部状态后走到下一累积状态。

两个接口都不承载业务语义，只满足状态表转起来的最低要求：`LocalStateInterface` 要求状态有一个能做 `__import__` 的 `state_id`；`CumulativeStateInterface` 还要一个指向泛型转移表的 `goto_table`。项目定义的具体状态是这两个接口的子类，`state_id` 就是定义它的模块路径。

## public.viba

```viba public
import transfer_table.types as transfer_table
import transfer_table.apis as transfer_table_api

StateId = transfer_table.StateId
GenericImportId = transfer_table.GenericImportId
LocalStateInterface = transfer_table.LocalStateInterface
CumulativeStateInterface = transfer_table.CumulativeStateInterface
CumulativeStateGotoTable = transfer_table.CumulativeStateGotoTable

get_si_local_state_schema = transfer_table_api.get_si_local_state_schema
get_si_cumulative_state_schema = transfer_table_api.get_si_cumulative_state_schema
get_candidate_next_local_states = transfer_table_api.get_candidate_next_local_states
goto_next = transfer_table_api.goto_next
```

## types.viba

两个 Id 都是能做 `__import__` 的模块路径。接口只写最低要求，具体状态由项目定义成它们的子类；泛型转移表把“当前累积状态 + 选中的局部状态”映到下一累积状态。

### StateId

一个模块路径，能做 `__import__` 的参数；该模块定义具体的 `LocalState`。

```viba types
StateId = str * { 能做 __import__ 的参数；指向定义了具体 LocalState 的模块 }
```

### GenericImportId

一个模块路径，能做 `__import__` 的参数；指向泛型实现，如 `CumulativeStateGotoTable`。

```viba types
GenericImportId = str * { 能做 __import__ 的参数；指向泛型实现的模块 }
```

### LocalStateInterface

局部状态的最小约束：一个能做 `__import__` 的 `state_id`；该模块的 `LocalState` 必须是本接口的子类。

```viba types
LocalStateInterface =
     tagged[__type__, Object]
  * $state_id (StateId * { 具体状态所在模块的路径 })
  * { __import__[self.state_id].LocalState <: LocalStateInterface }
```

### CumulativeStateInterface

累积状态的最小约束：一个 `state_id`，一个指向泛型转移表的 `goto_table`；该模块的 `CumulativeState` 必须是本接口的子类。

```viba types
CumulativeStateInterface =
     tagged[__type__, Object]
  * $state_id (StateId * { 具体累积状态所在模块的路径 })
  * $goto_table (GenericImportId * { 这个累积状态的泛型转移表的导入路径 })
  * { __import__[self.state_id].CumulativeState <: CumulativeStateInterface }
```

### CumulativeStateGotoTable

泛型转移表：给定当前累积状态与选中的局部状态，回答下一累积状态。项目把它实现成一个泛型模块，`goto_table` 指向那份实现。

```viba types
CumulativeStateGotoTable[LocalState] =
     CumulativeStateInterface
  <- $local_state (LocalState * { 选中的下一步局部状态 })
```

## 基本操作

1. 每个自进化项目在自己的专门目录里定义具体状态：`LocalState` 是 `LocalStateInterface` 的子类，`state_id` 是定义它的模块路径；`CumulativeState` 是 `CumulativeStateInterface` 的子类，除 `state_id` 外还带 `goto_table`。例如：

```viba_example
LocalStateABC =
     LocalStateInterface
  * $state_id "a.b.c"

CumulativeStateDFE =
     CumulativeStateInterface
  * $state_id "d.f.e"
  * $goto_table "g.h.i"
```

2. `get_si_local_state_schema` 从 `$si_repo_path` 找到项目的 local state schema（`LocalStateInterface` 的子类），返回并缓存，作为全项目 local state 的约束检查依据。
3. `get_si_cumulative_state_schema` 同理，返回 `CumulativeStateInterface` 的子类。
4. `get_candidate_next_local_states` 用当前累积状态的 `goto_table` 找到所在泛型实现的目录；在该目录里 grep `pattern LocalState`，看它 import 来源的 `state_id`；对每个 `state_id` 用 `__import__` 取定义，作为候选下一步。
5. `goto_next` 用 `goto_table` 找到对应的泛型 `CumulativeStateGotoTable`；用选中的局部状态索引它，返回下一累积状态。

## apis.viba

```viba apis
import transfer_table.types as transfer_table
```

### get_si_local_state_schema

返回项目的 local state schema；可以缓存，用来检查具体状态是不是它的子类。

```viba apis
get_si_local_state_schema =
     transfer_table.LocalStateInterface
  <- $env Env
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- {
    在 $si_repo_path 的项目专门目录里找到 local state 的 schema（LocalStateInterface 的子类）；返回它并缓存，作为全项目 local state 的约束检查依据
  }
```

### get_si_cumulative_state_schema

返回项目的 cumulative state schema；可以缓存，用来检查具体累积状态是不是它的子类。

```viba apis
get_si_cumulative_state_schema =
     transfer_table.CumulativeStateInterface
  <- $env Env
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- {
    在 $si_repo_path 的项目专门目录里找到 cumulative state 的 schema（CumulativeStateInterface 的子类）；返回它并缓存，作为全项目 cumulative state 的约束检查依据
  }
```

### get_candidate_next_local_states

当前累积状态下，列出候选的下一步局部状态。

```viba apis
get_candidate_next_local_states =
     list[transfer_table.LocalStateInterface]
  <- $env Env
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- $cum_state (transfer_table.CumulativeStateInterface * { 当前累积状态；它的 goto_table 指向泛型实现 })
  <- {
    通过 $cum_state.goto_table 找到所在泛型实现所在的目录；在目录里 grep `pattern LocalState`，看它 import 来源的 state_id；对每个 state_id 用 __import__ << state_id 取定义，收成候选列表
  }
```

### goto_next

用选中的局部状态，从当前累积状态走到下一累积状态。

```viba apis
goto_next =
     transfer_table.CumulativeStateInterface
  <- $env Env
  <- $si_repo_path (str * { SiGen 项目的仓库路径 })
  <- $cum_state (transfer_table.CumulativeStateInterface * { 当前累积状态；它的 goto_table 指向泛型实现 })
  <- $local_state (transfer_table.LocalStateInterface * { 选中的下一步局部状态 })
  <- {
    通过 $cum_state.goto_table 找到对应的 GenericImportId；用 __import__ << gen_imp_id 取泛型 CumulativeStateGotoTable；用 $local_state 索引它，返回下一累积状态
  }
```
