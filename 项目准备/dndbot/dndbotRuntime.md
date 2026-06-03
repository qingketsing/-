# DND AI BOT Agent Runtime / ReAct 面试问答

## 1. 你的 Agent Runtime 是怎么设计的？

我的 Agent Runtime 本质上是一个把**模型调用、工具执行和步骤累积**串起来的执行内核。核心结构是 `Runtime`，输入包括 `session_id`、`system_prompt`、`user_message`、`max_steps` 和 `context_limit`，输出包括最终回复和整轮执行的 `step` 记录。

从职责上看，它主要做三件事：

- 把当前上下文、历史步骤和工具列表组装成模型输入；
- 解析模型输出，判断是直接回复还是请求调用工具；
- 如果调用工具，就执行工具并把 observation 写回步骤历史，继续下一轮。

所以它不是一个单次模型包装器，而是一个**受控的 ReAct 循环执行器**。它把模型层、工具层和业务服务层隔开，方便做日志、指标、失败兜底和后续模型路由。

## 2. ReAct 在你的项目里怎么落地？

ReAct 在这个项目里不是停留在概念上，而是直接落到了 `Runtime.Run()` 的循环里。

每一轮大致是：

1. 用当前 `system_prompt + user_message + tools + previous steps` 组装模型输入；
2. 调模型拿到结构化输出；
3. 如果模型给的是最终 reply，就结束；
4. 如果模型给的是 `tool_request`，就执行工具；
5. 把这轮的 `thought / action / observation` 记到 step 里；
6. 继续下一轮，直到完成或达到 step 上限。

所以它的 ReAct 不是单纯 prompt 约定，而是有完整执行循环、工具执行器、错误处理和 step 累积的工程实现。

## 3. Agent 一次执行有哪些 step？

一次执行至少会有这几类 step：

第一步是**上下文准备**，在真正进入 Runtime 之前，系统会先拼好：

- knowledge warmup
- preloaded session context
- game state
- encounter
- session memory

第二步是**模型推理 step**，模型根据当前上下文决定：

- 直接回复
- 还是请求调用工具

第三步是**工具执行 step**。如果模型输出了 `tool_request`，Runtime 会调用对应工具，得到 observation。

第四步是**继续推理或终止**。如果 observation 还不足以形成最终回复，模型会进入下一轮；如果已经足够，就返回 reply。

所以一轮 Agent 执行，不是“只调一次模型”，而是“模型 step + 工具 step + 再次模型 step”的受控循环。

## 4. Agent 如何决定是否调用工具？

当前实现里，Agent 是否调用工具，是由**模型输出格式**决定的。

模型输出有两种合法形态：

- 直接给出 `reply`
- 给出 `tool_request`

Runtime 会先校验模型输出是否合法。如果模型已经产出最终回复，就直接结束；如果模型输出的是结构化 `tool_request`，Runtime 就进入工具执行分支。

也就是说，**是否调用工具不是由业务层硬编码判断，而是由模型在当前 step 的结构化输出决定的**。业务层做的是约束：

- 给模型可用工具列表
- 限制步数
- 提供上下文
- 处理失败和兜底

## 5. Agent 如何选择具体工具？

具体工具也是模型选的，但选择空间是被系统约束过的。

Runtime 在每一轮会把工具注册表里的 `ToolSpec` 一起传给模型，模型只能从这些已注册工具里选。比如当前工具集里有：

- `get_agent_context`
- `get_game_state`
- `search_rules`
- `search_lore`
- `resolve_attack_action`
- `apply_damage`
- `skill_check`

模型根据当前问题和工具描述输出 `tool_request.name`。然后 Runtime 通过统一的 `Executor` 去查注册表，按名称执行对应工具。

所以选择逻辑可以理解成：

- **模型负责“选哪个工具”**
- **系统负责“能选哪些工具、工具长什么样、工具怎么执行”**

## 6. 工具调用失败怎么办？

工具调用失败分两类处理。

第一类是**可继续的失败**。比如某个工具执行报错，Runtime 不会立刻整轮崩掉，而是先把这次失败记录成一个 `tool error step record`，把错误当成 observation 交回给模型，让模型有机会调整下一步。

第二类是**连续失败过多**。当前 Runtime 里有 `tool failure limit`，默认连续失败两次就终止，返回 `ErrToolFailureLimitExceeded`。这样可以避免模型陷入无意义重试。

更外层的 `AgentService` 还会根据错误类型生成用户可见的 fallback reply，比如：

- 规则检索失败
- 工具失败
- Runtime 失败

所以整体策略是：**单次工具失败尽量可恢复，连续失败则中止并兜底。**

## 7. 模型输出格式不符合预期怎么办？

当前 Runtime 对模型输出有严格校验。如果模型输出既不是合法 reply，也不是合法 `tool_request`，会触发 `ErrInvalidModelOutput`。

这类错误不会被默默吞掉，而是会：

- 记录为 runtime error
- 进入 `AgentService` 的 fallback 分支
- 返回一个保守的兜底回复

这样做的原因是，模型输出格式错误本质上是**协议错误**。如果不严格拦截，后面工具执行和状态更新都会变得不可靠。

所以我的做法是：**模型输出必须符合 Runtime 协议，不符合就视为执行失败，而不是尝试猜测和修补。**

## 8. 为什么需要 max step？

因为 ReAct 最大的问题之一就是容易失控。

如果没有 `max_steps`，模型可能会：

- 反复决定调用工具
- 在上下文不足时来回试探
- 卡在“再查一次、再想一次”的循环里

这会直接带来两个问题：

- 延迟越来越高
- 成功率反而下降

所以 `max_steps` 的作用是给 Runtime 一个硬边界。当前主路径默认上限是 8，但我后续优化方案里会进一步做模型动态路由，把简单请求限制到 0~2 步，复杂请求控制在 2~3 步。

一句话说，`max_steps` 是为了**限制最坏时延、限制成本、限制循环风险**。

## 9. Agent 陷入循环怎么办？

当前主要有三层保护。

第一层是 `max_steps`，超过上限直接终止。

第二层是 `tool failure limit`，如果模型反复调用失败工具，连续失败两次就停止，不让它一直试。

第三层是更外层的 fallback 机制。只要 Runtime 返回 step limit、tool failure 或 invalid output 这类错误，`AgentService` 就不会把错误直接暴露给用户，而是返回保守回复。

所以现在的策略不是“让模型自己无限自救”，而是：**循环一旦明显失控，就中止并降级**。

## 10. Agent 的中间状态如何持久化？

当前实现里，Agent 的**中间 step 轨迹主要保存在内存态执行上下文里**，会在一次 Runtime 运行结束前不断累积 `StepRecord`。

这些 step 不会逐步写回数据库成为 durable execution 日志；真正持久化的是：

- 用户消息
- assistant reply
- `message_job` 状态
- session / game state / encounter / memory 的最终业务状态

也就是说，目前项目是：

- **执行中的中间状态：内存中累积**
- **最终业务结果：数据库持久化**

如果后续要进一步增强恢复能力，我会把 step 级 durable execution 作为下一阶段设计，而不是当前版本默认行为。

## 11. 什么是模型动态路由？

模型动态路由就是：**不是所有请求都用同一个模型、同一种执行方式，而是根据请求类型和复杂度决定走哪条路径。**

当前项目已经落地了一个轻量版：

- 先用关键词分类器做意图识别
- 再用路由策略决定走 `fast` 还是 `primary`
- 同时给不同路径设置不同 `max_steps`

例如：

- `status_query / session_recall / character_draft` 走 fast
- 其它更复杂的请求走 primary

后续在 spec 里我进一步把它扩展成：

- 简单请求：`flash + no ReAct`
- 复杂请求：`pro + limited ReAct`

## 12. 为什么需要模型动态路由？

因为当前最主要的延迟来源不是数据库，而是**每轮都在等 LLM**。

如果所有请求都默认走重模型和多轮 ReAct，就会出现：

- 简单问题也很慢
- 每个请求成本都很高
- ReAct 轮数一多，时延线性叠加

模型动态路由的目的，就是把：

- 状态查询
- 简单规则问答
- 简单设定问答

这种问题尽量分流到更快、更轻的路径上，只把真正复杂的战斗和多步骤请求留给主模型和 ReAct。

本质上，它是在做**质量和延迟之间的结构化平衡**。

## 13. 规则裁定、设定查询、战斗处理分别是什么工具？

这三类能力在项目里对应的工具比较清晰。

规则裁定相关工具主要是：

- `search_rules`
- `roll_dice`
- `ability_check`
- `skill_check`

它们分别负责规则检索和基础判定。

设定查询主要是：

- `search_lore`

它负责检索世界观和设定知识库。

战斗处理相关工具主要是：

- `create_encounter`
- `get_encounter`
- `resolve_attack_action`
- `apply_damage`
- `heal`
- `advance_turn`
- `add_effect`
- `remove_effect`
- `can_act`

其中最关键的是 `resolve_attack_action`，它是我专门封装的高层战斗工具，用来一次性完成攻击检定、伤害结算、扣血和可选回合推进。

## 14. 为什么要封装高层工具？

因为如果只暴露低层工具，ReAct 很容易变成：

- 先查状态
- 再查规则
- 再掷骰
- 再算伤害
- 再扣血
- 再推进回合

这样 step 会很多，延迟和失败率都会上升。

所以我会把高频、稳定、业务上天然成组的动作封装成高层工具。最典型的是：

- `get_agent_context`
- `resolve_attack_action`

前者一次拿到最近消息和会话上下文，后者一次完成攻击结算。这样能显著减少工具往返次数，也能减少模型每轮重新规划的负担。

一句话说，高层工具的价值是：**把多步低层操作压缩成一步高价值业务动作。**

## 15. 你怎么把每个 step 从 15s 优化到 8s？

如果从工程角度总结，核心不是“把单个模型调用魔法般调快”，而是**减少每轮 step 的浪费**。

我主要会从四个方向优化：

第一，**上下文预加载**。当前已经落地了 `knowledge warmup + preloaded context prompt`，把 session、game state、encounter、memory 先拼进系统提示，减少 Runtime 里为拿上下文多走几轮。

第二，**高层工具替代低层工具链**。像 `resolve_attack_action` 这种工具，本质上就是把多次工具往返压成一次业务动作。

第三，**模型动态路由**。简单请求不再默认走主模型和长 ReAct，而是走 fast 路径。

第四，**限制 ReAct 轮数**。如果一个请求从 5 轮压到 2 轮，即使单轮模型速度不变，总时延也会明显下降。

所以从 15s 到 8s，不是单点优化，而是**预加载上下文 + 高层工具 + 模型分流 + step 控制**的组合结果。

## 16. 长会话成功率 70% 到 90% 是怎么定义和统计的？

这个指标不是主观感觉，而是通过 `soak eval` 场景测试定义的。

当前项目里有专门的长会话评测逻辑：

- 一个 player 模型模拟玩家连续发言
- 一个 judge 模型评估每轮 DM 回复是否成功

成功标准包括：

1. 回复必须回应本轮输入
2. 不能忘记已建立角色、场景、任务、战斗状态
3. 不能要求用户重复已经提供的信息
4. 战斗中的攻击、伤害、回合必须推进
5. 出现失败时回复必须可恢复
6. 不能和当前状态冲突
7. 状态查询必须给出明确状态

统计方式就是：

- 跑一批多轮场景
- 用 judge 给每轮打 `success=true/false`
- 用成功轮次 / 总轮次，或者成功对话数 / 总对话数来算成功率

所以这个 70% 到 90% 的口径，本质上是**长链路场景完成率**，不是单轮 HTTP 成功率。

## 17. 失败 case 主要有哪些？

当前失败 case 主要可以分四类。

第一类是**模型层失败**：

- 模型超时
- 模型服务错误
- 输出格式不合法

第二类是**工具层失败**：

- 工具参数不合法
- 工具执行报错
- 当前没有对应结构化状态，比如 encounter 不存在

第三类是**检索层失败**：

- RAG 检索服务异常
- 检索结果退化
- 检索不到足够支撑回答的内容

第四类是**Runtime 层失败**：

- step 超限
- 连续工具失败超限
- 异步执行中 session busy
- 状态写回失败

从业务上看，最典型的失败表现是：

- 回复慢到超时
- 忘记上下文
- 战斗回合没有推进
- 回复内容和已知状态冲突

## 18. 你怎么判断失败是 prompt、工具、检索还是模型问题？

我会按阶段拆分，而不是只看最终“失败了”。

第一，看 **Runtime 错误类型**：

- `ErrInvalidModelOutput` 更偏模型协议问题
- `ErrToolFailureLimitExceeded` 更偏工具链问题
- `ErrStepLimitExceeded` 更偏 prompt / 路由 / 工具粒度设计问题

第二，看 **日志和指标分段**：

- `warmup_build`
- `system_prompt_compose`
- `preloaded_context_build`
- `runtime_total`
- tool call duration
- model call duration

如果是模型慢，通常会体现在 model call 上；如果是工具慢，会体现在 tool call 上；如果是检索退化，`search_rules/search_lore` 会带 degraded 结果。

第三，看 **fallback 分类**。当前 `AgentFallbackResponder` 会把错误粗分为：

- `model`
- `rag`
- `tool`
- `runtime`

第四，看 **失败现象是否稳定复现**：

- 如果固定某类问题总失败，更可能是 prompt 或工具设计问题
- 如果随机失败，更可能是模型波动、外部依赖或超时

所以我判断失败，不是凭感觉，而是用**错误类型 + 分段耗时 + 工具日志 + fallback 分类**一起定位。
