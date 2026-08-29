# 架构

> 中文版。英文原版见 [ARCHITECTURE.md](ARCHITECTURE.md),两版内容对应。

[README](../README.md) 说明的是**有哪些组件**。这份文档说明的是**控制流实际怎么走**——调度规则、每个分支点、以及运行时依赖的那些假设。这部分信息散落在 `_apply_follow_ups`、`_decide_next_steps`、各 agent 的 `execute`,以及一张 priority 整数表里,从头读代码很难拼出来。

- [最重要的一个概念](#最重要的一个概念)
- [主流程](#主流程)
- [下一个 agent 是怎么被选中的](#下一个-agent-是怎么被选中的)
- [各 agent 内部的分支](#各-agent-内部的分支)
- [决策点](#决策点)
- [状态存在哪里](#状态存在哪里)
- [共享可变状态与单进程假设](#共享可变状态与单进程假设)

> 想知道每个 agent 具体在要求模型做什么,见 [PROMPTS.zh-CN.md](PROMPTS.zh-CN.md) —— 14 个提示词模板的中文对照导读,含每个 `{{ 变量 }}` 由谁填什么。

## 最重要的一个概念

**没有任何 agent 会调用另一个 agent。** 每个 agent 返回一个 `TaskResult`,由 Supervisor 把这个结果翻译成 `tasks` 表里的新行。这张表是 agent 之间唯一的通道。

这让 `tasks` 表成为一条消息总线,每个 agent 成为一个消息处理器——也就是 actor 模型,只不过邮箱在 SQLite 里而不是在内存里。把邮箱放进数据库换来的就是 `co-scientist resume`:队列能在崩溃后存活,所以一个跑到一半的会话可以后续接上。

有一个推论值得记住:**调度里没有任何智能成分。** 论文把 Supervisor 描述成一个 agent,这里它是一个确定性的规则引擎加一句 SQL `ORDER BY`。唯一让模型影响控制流的地方是 `parse_goal`,它把自然语言目标解析成 `ResearchPlan`。之后的一切都是硬编码的。这是一个有意的取舍:可复现、可单测、崩溃后可恢复,代价是不如模型驱动的调度灵活。

## 主流程

`[n]` 是任务优先级(数值小的先跑),`◇` 标记分支。

```
  co-scientist run "<目标>"
        │
        ▼
  ┌────────────┐   record_research_plan
  │ parse_goal │───────────────────────────►  ResearchPlan
  └────────────┘   (一次性;唯一影响控制流的模型调用)
        │
        │ 入队 n_initial 个 Generation
        ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                  tasks 表 —— 唯一的通道                                ║
║      claim_one: ORDER BY priority ASC, created_at ASC  (原子)          ║
╚═══════════════════════════════════════════════════════════════════════╝
   ▲                                                     │ 领取
   │ 入队,由 result.kind 决定                             ▼
   │                                        ┌──────────────────────────┐
   │                                        │ 按 task.agent 分派        │
   │                                        └────────────┬─────────────┘
   │     ┌────────────┬────────────┬─────────────┬───────┴──────┬─────────────┐
   │     ▼            ▼            ▼             ▼              ▼             ▼
   │ ┌────────┐  ┌──────────┐ ┌─────────┐  ┌───────────┐  ┌──────────┐ ┌───────────┐
   │ │生成    │  │ 评审     │ │ 排序    │  │  进化     │  │ 邻近度   │ │ 元评审    │
   │ │        │  │          │ │         │  │           │  │ 无 LLM   │ │           │
   │ │ [100]  │  │  [100]   │ │[80/120] │  │  [140]    │  │  [200]   │ │ [180 / 1] │
   │ └───┬────┘  └────┬─────┘ └────┬────┘  └─────┬─────┘  └────┬─────┘ └─────┬─────┘
   │     │            │            │             │             │             │
   │  ◇撞上重复?      │        ◇verdict          │             │             │
   │   是 → noop      │         能解析?          │             │             │
   │   否 ─┤          │         否 → noop        │             │             │
   │       │          │         是 ─┤            │             │             │
   │       ▼          ▼             ▼            ▼             ▼             ▼
   │  hypothesis_  review_    tournament_   hypothesis_   proximity_   system_
   │   created    completed  match_complete   created      updated     feedback
   │       │          │             │            │             │
   └───────┴──────────┴─────────────┴────────────┘             │
              后续规则把这些结果变成下一批任务                    │
                                    ◇ 每 20 场比赛 ─────────────┘

  队列空了 ──► decide_next_steps(最快每 10 秒一次)
        ◇ 在锦标赛中的 ≥ 2                ──► 锦标赛批次   [150]
        ◇ 成熟(≥3 场)的 ≥ min_mature     ──► 进化         [140]
        ◇ 比赛数 ≥ (已有反馈数+1) × 50    ──► 系统反馈     [180]
        ◇ 一个都没排出来                   ──► StopReason.IDLE

  每轮循环检查 should_stop():
        EXTERNAL │ BUDGET │ WALL_CLOCK │ ELO_STABLE
                 └──► 取消所有 pending ──► 最终综述 [1] ──► overview.md
```

注意**进化产出的 kind 和生成完全相同,都是 `hypothesis_created`**。这就是为什么只需要四条后续规则:一个进化出来的假设会自动重新进入"评审 → 锦标赛"这条链,不需要为它再写一套。

## 下一个 agent 是怎么被选中的

四套机制叠在一起。没有一套涉及模型,也没有一套写在 agent 里面。

### 1. 静态入口

`parse_goal` 之后,Supervisor 入队 `n_initial` 个 Generation 任务,载荷是 `{"strategy": "literature", "n": 1}`。**一次调用产出一个假设**,并行度来自多个任务,而不是来自要求一次调用产出多个假设。

### 2. 后续规则 —— `_apply_follow_ups`

规则的键是刚完成的任务返回的 `TaskResult` 的 `kind` 字段,**不是**产出它的 agent:

| `result.kind` | 入队 | 优先级 |
| --- | --- | --- |
| `hypothesis_created` | 每个新假设一个 Reflection,`kind=full` | 100 |
| `review_completed` | Ranking `AddToTournament` | 80 |
| `added_to_tournament` | Ranking `RunTournamentBatch`,带 `focus=<id>` | 120 |
| `tournament_match_complete` | Proximity 重聚类,仅当 `比赛数 % full_recluster_every_matches == 0` | 200 |

### 3. 优先级 —— 真正决定"下一个跑哪个"

这个决定是一句 SQL,不在任何 agent 里:

```sql
ORDER BY priority ASC, created_at ASC LIMIT 1
```

数值小的先跑,同值按创建时间先进先出。在用的值就这些:

| 优先级 | 任务 | 为什么在这个位置 |
| --- | --- | --- |
| 1 | 最终研究综述 | 收尾时必须第一个 |
| 80 | `AddToTournament` | 纯数据库写入,不调模型,几毫秒。让它插队几乎不占资源,却能解锁整条下游链 |
| 100 | 初始 Generation、Reflection(full) | 默认档 |
| 120 | 聚焦某个新假设的 `RunTournamentBatch` | |
| 140 | Evolution(空闲补活) | 精化工作,不该抢占正在流动的主链路 |
| 150 | 锦标赛批次(空闲补活) | 同上 |
| 180 | 元评审系统反馈 | 同上 |
| 200 | Proximity 重聚类 | 批量工作,没有时限 |

优先级解决的问题是**饥饿**:一次会话的大部分时间里队列都是非空的(60 个 idea × 评审 + 进榜 + 比赛,而并发只有 4),纯先进先出会让一个刚评审完的假设排在几十个分钟级任务后面才能进入锦标赛——而在它进榜之前,它拿不到 Elo 分,也不会被 Evolution 选中。

优先级**不是抢占式的**。它只影响 `claim_one` 的排序。一个已经在 600 秒租约下运行的 Reflection 不会被新来的 priority=80 任务打断。它的粒度是"下一个空槽给谁",不是"立刻插队"——在这里够用,因为任务都是分钟级且包着一次不可中断的模型调用。

### 4. 空闲补活 —— `_decide_next_steps`

只在没有在途任务、也没有 pending 时运行,最快每 10 秒一次。三条条件规则(见上表)。如果一个任务都没排出来,会话以 `StopReason.IDLE` 退出。

它的幂等键锚定在**当前比赛计数**上,而不是新生成的任务 id。如果用新 id,每次空闲检查(可能每 10 秒一次)都会再入队一个锦标赛或进化任务,哪怕前一个还在 pending——队列会被灌满,而且同一份工作会被重复计入预算。

### 5. 收尾

`should_stop` 每轮循环检查一次,顺序是:`EXTERNAL`(用户中止)、`BUDGET`(`budget_used_usd` 列超过 `budget_usd`)、`WALL_CLOCK`、`ELO_STABLE`(最近 N 次快照里 top-K 排名没变)。命中后在途任务排空、所有 pending 任务被取消,然后一次最终元评审产出 `overview.md`。

## 各 agent 内部的分支

| agent | 分支 | 由什么决定 |
| --- | --- | --- |
| 生成 | `literature`(已实现);`debate` / `assumption` / `feedback_driven` 抛 `NotImplementedError` | `payload["strategy"]` |
| 评审 | `full` / `verification` / `observation` —— 各用不同模板和不同思考预算(`verification` 最高) | `payload["kind"]` |
| 排序 | `AddToTournament`(纯数据库写入,不调模型)/ `RunTournamentBatch` | `task.action` |
| ↳ 配对 | 指定 focus · 新 × 最近 Elo(`p_new`)· 近 Elo × 远想法(`p_close`)· 前半随机(`p_random`) | 随机数落进哪个桶 |
| ↳ 比赛模式 | `debate` / `pairwise` | 场次 `< debate_when_matches_lt` 或 \|ΔElo\| `< debate_when_elo_delta_lt` 时用辩论 |
| 进化 | `combine`(top-K 里向量距离最远的一对)/ `simplify` / `feasibility` / `out_of_box` —— 各自独立运行,一个失败不影响其它 | `payload["strategies"]` |
| 邻近度 | 只补缺失的 embedding / 补完再全量重聚类 | `payload["rebuild"]` |
| 元评审 | `GenerateSystemFeedback` / `GenerateFinalResearchOverview` | `task.action` |

## 决策点

这些分支读代码时最容易漏掉,因为它们是**静默结束一条链**,而不是抛异常:

| 位置 | 条件 | 后果 |
| --- | --- | --- |
| 生成 / 进化落库 | 最近邻余弦 ≥ `dedup_cosine_threshold` | 返回已有 id 且 `was_new=False`,于是 `TaskResult.hypothesis_ids` 是**空的**,不会入队 Reflection —— 这条链在这里断掉 |
| 生成 / 进化落库 | 行实际上没有被插入 | 向量也**不会**被加进索引,所以索引和表永远不会分歧 |
| 排序比赛 | verdict 无法解析 | 记录一场 `mode="invalid"` 的比赛,**不**更新 Elo,返回 `kind="noop"` —— 没有后续 |
| 排序比赛 | 崩溃后重试 | `round_id = task.id`,所以 `match_id` 会被算出完全相同的值,Elo 日志拒绝这次重复 |
| 工具循环(驱动模式) | 响应里出现 `record_*` 工具 | 立即结束循环且**不派发**它 —— 答案已经在 `tool_use.input` 里了 |
| 工具循环(驱动模式) | 最后一轮允许的迭代 | `tool_choice` 被强制指向记录工具 |
| 工具循环(捕获模式) | 捕获里没有记录 | 升级重试一次,去掉检索工具;再失败就抛异常 |
| 进化 | top 集合里不足 2 个假设 | `kind="noop"` |

## 状态存在哪里

两个经常被混淆的东西:

- **配置**是输入:人写的,启动时读一次,不再变化。分层是 `config/default.toml` → `~/.co-scientist/config.toml` → `./co-scientist.toml` → `--config`。密钥只从环境变量来。
- **状态**是输出:程序写的,一直在变,需要被查询、聚合,并且要能在崩溃后存活。

状态按体积和访问模式分开存放:

| 什么 | 存在哪 | 为什么 |
| --- | --- | --- |
| 假设正文、评审正文、完整 LLM transcript | `data/artifacts/<session>/` 下的 JSON 文件 | 体积大、写一次、从不按内容查询 |
| 指向那些文件的行(`artifact_path`) | SQLite | 需要查询和联表 |
| 任务队列 | SQLite | 需要原子领取和崩溃恢复 |
| Elo、比赛历史、预算计数 | SQLite | 需要原子自增和幂等更新 |
| embedding | `data/vectors/<session>/` 下的 FAISS 索引文件,加 `embeddings_meta` 一行 | 对 L2 归一化向量做 O(N) 精确检索 |
| 给 Web UI 的实时事件 | 内存里的 `EventBus`,同时镜像进 `events` 表 | 重启后 UI 从表里做快照 |

真正**必须**用数据库而不能用文件的只有四件事:并发安全的原子更新、条件查询(`top_by_elo`、`state='in_tournament'`、"哪些假设还没有 embedding")、崩溃恢复、以及增量增长到几千行。其余的本来就已经在文件里了。

## 共享可变状态与单进程假设

运行时假定**一个会话一个进程**。这个假设在今天是成立的,处理它的代码也很仔细,但**这个假设是隐式的**——没有任何机制强制它,也没有启动检查能拦住对同一个会话发起第二个 `run` 或 `resume`。

三处状态是进程内的:

| 状态 | 由什么保护 | 两个进程下会怎样 |
| --- | --- | --- |
| `TokenBudget`(per-agent 份额、预留) | `asyncio.Lock` | 每个进程各算一套计数器,实际上限被进程数放大。数据库里的 `budget_used_usd` 列是原子自增而且 `should_stop` 读它,所以全局刹车仍然有效——精度变差,但没有失控 |
| `FaissStore`(索引文件 + `ordered_ids`) | `asyncio.Lock` | **丢更新。** `save()` 已经能防损坏(先写 `.tmp` 再 `os.replace`),但两个进程各自 load、`add`、`save`,后写的会覆盖前写的向量。而查重是"读—改—写",所以重复项也会漏过去 |
| `EventBus`(每会话的 `asyncio.Queue`) | 只在进程内 | SSE 订阅者只能看到自己进程发布的事件。`events` 表有完整记录,所以这是三者里最容易补的——轮询那张表即可 |

如果你打算跑多个 worker,先读 [PITFALLS.zh-CN.md](PITFALLS.zh-CN.md#b-部分--并发与共享状态) —— 那里详细描述了这三处各自的失效方式。
