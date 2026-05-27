# DND AI BOT RabbitMQ / Outbox / Redis Lock 面试问答

## RabbitMQ / message_jobs / Outbox

## 1. 为什么模型调用和回复生成要异步化？

因为 DND 场景里的回复不是单次文本生成，往往要经历：

- 读取 session 历史
- 加载 game state / encounter / memory
- 检索 rules / lore
- 多轮 ReAct
- 工具调用

如果把这些都放在同步 HTTP 请求里，单次请求很容易到几十秒，用户体验会很差，而且连接占用和超时风险也会明显上升。

所以异步化的目标不是“炫技”，而是把系统改成：

- 先可靠接单
- 后台慢慢处理
- 前端可轮询状态

这样更适合长链路 Agent 执行。

## 2. `message_jobs` 任务状态机是什么？

`message_jobs` 是项目里对“单条用户消息异步处理过程”的结构化建模。

它把一条消息从进入系统到最终完成，拆成一个可追踪的生命周期，而不是只靠日志判断任务到了哪一步。

也就是说，`message_jobs` 不是简单队列表，而是**任务状态机**：每条消息对应一条 job，job 会按固定状态迁移，系统通过它判断：

- 是否已经发布到队列
- 是否正在处理
- 是否已经完成
- 是否可重试失败

## 3. `message_jobs` 有哪些状态？

当前表结构里定义了这些状态：

- `queued`
- `published`
- `processing`
- `completed`
- `retryable_failed`
- `failed`
- `cancelled`

含义分别是：

- `queued`：已接单，尚未确认发布到消息通道
- `published`：已发布，可被 worker 消费
- `processing`：worker 已开始处理
- `completed`：任务成功完成
- `retryable_failed`：本次失败，但可恢复
- `failed`：不可恢复失败
- `cancelled`：任务被取消

## 4. 为什么需要任务状态机？

因为异步系统里，不能只靠“有没有回复”来判断任务是否成功。

比如一条消息可能出现：

- 已写入数据库但还没发到 MQ
- 已发到 MQ 但 worker 还没开始
- worker 已经开始但还没完成
- 处理中失败但可以重试
- 处理失败且不该再试

如果没有状态机，就很难做：

- 前端状态查询
- 重试
- 幂等处理
- 失败定位
- 监控统计

所以状态机的价值是让异步链路**可观测、可恢复、可统计**。

## 5. `outbox_events` 是什么？

`outbox_events` 是数据库里的“待发布事件表”。

在这个项目里，它主要记录：

- 哪个聚合对象要发事件
- 事件类型是什么
- 要发到 MQ 的 payload 是什么
- 当前发布状态
- 发布失败次数和最后错误

对于异步消息场景来说，一条典型 outbox event 就表示：

**这条 `message_job` 需要被可靠发布到 RabbitMQ。**

## 6. 什么是 Outbox 模式？

Outbox 模式的核心思想是：

**不要在业务事务里直接强依赖 MQ 发布成功，而是先把“待发布事件”写进数据库，再由后台异步分发。**

在这个项目里，就是把：

- 用户消息
- `message_job`
- `outbox_event`

放到同一个 PostgreSQL 事务里写入。事务提交成功后，说明这条消息已经被系统可靠接住；后续即使 RabbitMQ 暂时不可用，后台也还能继续补发。

## 7. 为什么要用 Outbox 模式？

因为如果不这么做，会有一个很典型的故障窗口：

1. 用户消息已经写入数据库
2. `message_job` 也写入了
3. RabbitMQ publish 失败

结果就是：

- 数据库里已经有消息
- 但这条任务没有进入异步执行链路
- 任务悬空，没人处理

Outbox 的价值就在于把这个窗口堵住。

## 8. Outbox 解决了什么一致性问题？

它解决的是：

**“数据库事务已成功提交，但 MQ 发布失败，导致任务没进入队列”的一致性问题。**

更准确地说，Outbox 保证的是：

- 业务写入和“待发布事件”是原子的
- RabbitMQ 暂时失败时，任务不会丢

它不解决所有分布式一致性问题，但解决了最关键的**入队一致性问题**。

## 9. PostgreSQL 作为状态真相源是什么意思？

意思是：**系统最终以 PostgreSQL 里的状态为准。**

在这套架构里：

- `message_jobs` 记录任务状态真相
- `outbox_events` 记录发布状态真相
- `session_messages / game_states / encounters / session_memories` 记录业务状态真相

RabbitMQ 只是分发通道，Redis 只是缓存和锁，不承担最终业务真相。

所以出问题时，恢复扫描、前端状态查询、排障定位，最终都应该回到 PostgreSQL 看状态。

## 10. RabbitMQ 在项目里负责什么？

RabbitMQ 当前主要负责**异步消息分发**。

具体来说：

- `OutboxDispatcher` 扫描 `outbox_events`
- 把 `message_job` 的 payload 发布到 RabbitMQ

所以 RabbitMQ 在这套系统里的职责是：

- 解耦接单和处理
- 承接后台消息执行链路
- 支持后续独立 worker 消费

要注意一点：当前项目的 publish 侧已经接好了，但完整独立 consumer/worker 的生产闭环还在继续推进中。

## 11. Worker 负责什么？

Worker 的职责是**真正执行一条异步消息任务**。

当前 `MessageJobProcessor` 已经定义了 worker 的主要工作：

1. 读取 `message_job`
2. 获取 session lock
3. 把 job 标成 `processing`
4. 启动锁续约 heartbeat
5. 读取 session 和用户消息
6. 调用 `AgentService.Reply(...)`
7. 把 assistant reply 写回 session
8. 最后把 job 标成 `completed`

也就是说，RabbitMQ 负责分发任务，Worker 负责真正消费任务并生成回复。

## 12. 如何保证消息不丢？

当前主要靠三层。

第一层是**事务化写入**：

- 用户消息
- `message_job`
- `outbox_event`

在同一个 PostgreSQL 事务里提交。

第二层是**OutboxDispatcher 重扫机制**：

- 会持续扫描 `pending` 和 `failed` 的 outbox 事件
- 所以 RabbitMQ 短时失败不会直接丢任务

第三层是**状态真相留在 PostgreSQL**：

- 即使 MQ 出现抖动
- 数据库里仍然能看到这条任务和事件

所以它的思路不是“保证网络永不出错”，而是“即使中间失败，任务仍可恢复”。

## 13. 如何处理重复消费？

重复消费是按幂等思路处理的。

当前有几层保护：

第一层，worker 开始时会先查 job，如果已经是 `completed`，直接返回，不重复处理。

第二层，assistant reply 使用显式关联字段：

- `reply_to_message_id`
- `source_job_id`

第三层，数据库加了唯一约束：

- assistant 对同一 `reply_to_message_id` 只能有一条回复
- assistant 对同一 `source_job_id` 只能有一条回复

第四层，PostgreSQL 唯一约束冲突会被映射成“幂等成功”，而不是普通失败。

所以重复消费不会简单靠 MQ 保证“不重复”，而是靠**消费重复了也不会写乱结果**。

## 14. 如何处理消费失败？

当前按失败类型分流。

如果是可恢复失败，比如：

- session busy
- 模型调用失败
- session 保存失败
- 锁续约失败

会把 job 标成：

- `retryable_failed`

如果是不可恢复失败，比如：

- 消息在 session 里找不到

会标成：

- `failed`

也就是说，消费失败不会都落成一个笼统的失败状态，而是区分：

- 可重试
- 不可重试

## 15. 如果 RabbitMQ 挂了怎么办？

当前策略是：**不丢任务，但发布会延迟。**

因为业务入口不会直接依赖 RabbitMQ 成功，而是先写：

- `message_job`
- `outbox_event`

如果 RabbitMQ 挂了：

- `OutboxDispatcher` 发布失败
- 对应 `outbox_event` 会标成 `failed`
- `attempt_count` 增加，`last_error` 记录错误

只要 RabbitMQ 恢复，dispatcher 后续还能继续扫描这条 outbox event 再发。

所以 RabbitMQ 宕机影响的是**处理时效**，不是**接单可靠性**。

## 16. 如果 Worker 挂了怎么办？

当前实现里，worker 挂掉后的表现取决于挂掉时机。

如果 worker 在拿锁前就挂了，任务还停在 `published`，后续可以重新消费。

如果 worker 已经进入 `processing`，但中途挂了：

- Redis 锁最终会因为 TTL 到期释放
- job 会停在 `processing`

当前项目已经做了锁续约和状态机，但**完整的 stale processing recovery scanner 还没有完全落地成最终闭环**。所以现在的基础是：

- 锁会释放
- 状态会保留

后续要靠恢复扫描把卡死的 `processing` job 拉回 `retryable_failed`。

## 17. 如果模型调用超时，job 状态怎么变化？

当前 `MessageJobProcessor` 里，如果 `agentService.Reply(...)` 返回错误，并且不是锁续约问题，就会：

- `MarkRetryableFailed(...)`
- `error_code = agent_reply_failed`

也就是说，模型超时当前会被视为**可恢复失败**，job 会进入：

- `retryable_failed`

这是合理的，因为模型超时通常是临时性问题，不应该直接视为永久失败。

## 18. 为什么不用同步调用模型？

因为同步模型调用会把最慢链路直接暴露给用户请求。

在这个项目里，同步调用的问题主要有：

- 长会话和复杂战斗场景下回复太慢
- HTTP 请求容易阻塞几十秒
- 容易超时
- 无法很好承接后续 worker、重试和状态查询能力

所以异步化的价值是：

- 接口先可靠返回 `202`
- 后台慢慢处理
- 前端通过 job 状态感知处理进度

对于这种长链路 Agent 系统，这种交互方式更合理。

## 19. 异步化后用户如何获取回复？

当前方式是**状态轮询**。

用户发消息后，接口会返回：

- `message_id`
- `job_id`
- `status`

前端后续调用：

- `GET /messages/{message_id}`

就可以看到：

- `queued`
- `processing`
- `completed`
- `failed`

如果已经完成，响应里还会带上 `assistant_reply`。所以前端不需要等长连接，而是通过任务状态轮询拿到结果。

## 20. 如何监控队列积压和 job 失败率？

当前项目已经具备部分基础观测能力，但完整监控体系还可以继续加强。

就现有数据结构来说，可以直接从 PostgreSQL 统计：

- `message_jobs` 中 `queued/published/processing/retryable_failed/failed` 的数量
- `outbox_events` 中 `pending/failed` 的数量

这些指标可以反映：

- 队列积压
- 发布失败率
- job 失败率
- 平均延迟

更进一步，Runtime、tool、RAG 已经接了 Prometheus 观测；异步链路后续可以继续补：

- outbox backlog gauge
- job failure rate
- retryable_failed total
- processing stale count

所以当前的方向是：**数据库状态做离线统计，Prometheus 指标做运行时监控。**

---

## Redis session lock / 幂等

## 1. 为什么需要 Redis session lock？

因为同一个 DND 会话是强状态的。

一条消息的处理可能会读写：

- session history
- game state
- encounter
- session memory

如果同一个 session 的两条消息同时处理，就可能同时推进状态，导致回合、血量、任务进度和剧情上下文互相覆盖。

所以要用 Redis session lock 保证：

**同一个 session 任意时刻只允许一个 worker 处理。**

## 2. 为什么同一会话消息要串行处理？

因为 DND 会话天然有强顺序依赖。

比如：

1. 玩家先说“我攻击地精”
2. 再说“我捡起掉落物”

第二条消息必须建立在第一条已经结算完的基础上。如果两条消息并行执行，就可能出现：

- 还没结算攻击，就先捡掉落
- 或者回合推进顺序错乱

所以同一会话必须串行处理，这不是实现偏好，而是业务要求。

## 3. 如果同一会话并发处理，会出现什么问题？

最典型的问题有：

- 战斗回合推进错乱
- HP 被重复扣减
- 任务状态被覆盖
- 背包和金币变化互相覆盖
- 回复内容引用了错误上下文
- assistant reply 顺序和真实状态不一致

本质上是**状态竞争和乱序执行**。所以 session 级串行是必须的。

## 4. session lock 的 key 如何设计？

当前实现的 key 是：

```text
session:{session_id}:processing_lock
```

也就是把锁粒度明确放在 session 维度。

这个设计很直接，优点是：

- 语义清楚
- 不同 session 可以天然并发
- 同一 session 会互斥

## 5. 锁过期时间怎么设置？

当前默认 TTL 是：

```text
180s
```

这是在 `session_lock.go` 里定义的默认值。

设置思路是：

- 不能太短，否则长时间模型调用时容易自然过期
- 也不能太长，否则 worker 崩溃后释放太慢

所以第一版取了一个相对保守的中间值，再配合 heartbeat 续约。

## 6. 为什么需要 heartbeat 锁续约？

因为 LLM 调用和 Agent 处理时间可能远超几十秒。

如果只在开始时拿锁一次，不续约，就会出现：

- 任务还在正常执行
- 锁先过期了
- 第二个 worker 又拿到锁开始处理

这样就会导致同一 session 并发执行。

所以 heartbeat 的作用就是：**只要 worker 还活着，就持续告诉 Redis“这把锁还归我”。**

## 7. Worker 崩溃后锁如何释放？

当前策略是依赖 TTL 自动释放。

如果 worker 崩溃：

- 它不会再续约 heartbeat
- 锁会在 TTL 到期后自然失效

所以这套设计不依赖“进程一定优雅退出”，而是允许异常退出后通过 TTL 回收锁。

## 8. Redis 锁会不会误删别人的锁？

当前实现里专门防了这个问题。

锁 value 不是简单写个标记，而是 JSON，里面包括：

- `job_id`
- `worker_id`

续约和释放都不是直接 `EXPIRE/DEL`，而是通过 Lua 脚本先比较当前 value 是否等于预期 value，再决定：

- `PEXPIRE`
- `DEL`

所以一个 worker 只能续约或删除**自己拥有的锁**，不会误删别人的锁。

## 9. 如何避免锁过期导致并发执行？

当前主要靠两层：

第一层是较长的默认 TTL，比如 180 秒。

第二层是 heartbeat，每 30 秒续约一次。

所以只要 worker 还活着、网络正常，它就会不断刷新 TTL，不会让锁自然过期。

当然，这不能 100% 消灭所有分布式边界问题，但已经把“正常长任务执行时锁自然过期”的风险大幅压低了。

## 10. 为什么不用数据库锁？

因为数据库锁不太适合这种**跨长时间模型调用**的场景。

如果把整轮 Agent 执行绑定在数据库锁上，会有几个问题：

- 锁持有时间太长
- 容易把数据库连接和事务拖住
- 对 PostgreSQL 压力更大
- 不适合跨服务、跨进程长时间持有

Redis 锁更适合做这种“轻量分布式互斥 + TTL 自动回收”的场景。

## 11. 为什么不用消息队列按 session 分区？

这是一种可选设计，但当前项目没有这么做，主要原因是复杂度和收益不匹配。

如果按 session 分区，需要：

- 设计稳定的分区键
- 确保消费组和扩容策略匹配
- 处理分区热点
- 兼容后续 worker 扩展

而 Redis session lock 的实现成本更低，语义也更直接：

- MQ 负责分发
- Redis 负责同 session 串行

对当前项目阶段来说，这更实用。

## 12. 数据库唯一约束如何实现幂等？

当前幂等主要落在 assistant reply 上。

系统在 `session_messages` 上加了两个唯一约束：

- assistant 对同一个 `reply_to_message_id` 只能有一条回复
- assistant 对同一个 `source_job_id` 只能有一条回复

这样即使出现重复消费、重复重试、重复投递，数据库最终也只允许写出一份结果。

所以数据库唯一约束在这里扮演的是**最终幂等锁**的角色。

## 13. 幂等 key 怎么设计？

当前有两个关键幂等标识：

- `message_id`
- `job_id`

对应的结果关联字段是：

- `reply_to_message_id`
- `source_job_id`

也就是说：

- `message_id` 标识这条用户输入
- `job_id` 标识这次异步执行
- `reply_to_message_id` 防止同一用户消息被重复回复
- `source_job_id` 防止同一 job 被重复产出结果

## 14. 什么场景会发生重复消费？

重复消费在异步系统里很常见，主要包括：

- RabbitMQ 重投
- worker 执行后 ack 之前崩溃
- recovery 重新发布
- session busy 后重新入队
- outbox 重复发布

所以设计时不能假设“每条消息只会消费一次”，而应该默认：

**同一个 job 可能会被处理多次，但最终结果必须等价于处理一次。**

## 15. Redis 挂了怎么办？

这个问题要分场景看。

对于异步消息处理来说，Redis 挂了会直接影响 session lock，因此当前系统会倾向于**fail closed**，而不是冒险继续并发处理同一 session。

也就是说：

- 拿锁失败时，worker 不应继续推进状态
- 更合理的结果是让 job 进入可恢复失败，后续再试

另外，项目里 Redis 还承担缓存和限流职责，所以 Redis 挂了还会影响：

- auth session cache
- session/game state/memory 缓存
- rate limiting

如果从系统设计角度总结，当前策略是：

- PostgreSQL 仍然是主真相源，不会因为 Redis 挂了而丢业务数据
- 但执行效率和并发安全会下降
- 对异步执行链路，宁可保守失败，也不要在没有 session lock 的情况下继续处理
