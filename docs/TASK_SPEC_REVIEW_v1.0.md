# .task 文件规范 v1.0 — 审查报告

审查人: Hermes Agent (独立审查)
审查时间: 2026-05-02
审查对象: ~/.hermes/TASK_SPEC.md (191 行)

---

## 严重问题（会导致系统无法运行或数据损坏）

### [严重] #1 — 状态机存在不可恢复的死胡同

`failed` 状态只能进不能出。流转图定义"任意状态 → failed"，但 `failed` 没有任何出口边。这意味着一次网络抖动、一个 minerU 解析错误、一次临时磁盘满，都会让任务永久卡死在 `failed`。

规范本身不自洽：状态枚举表说 `failed` 可通过"小艾/验收不通过"触发，但操作协议中"小艾验收"章节说验收不通过 → `running`（打回修改），而非 `failed`。两处矛盾。

修改建议:
- 增加 `failed → pending` 转换（小艾手动重试）
- 或者在 `history.note` 中区分"可重试失败"与"永久失败"
- 统一状态表和操作协议中关于验收不通过的跳转目标

### [严重] #2 — Agent 运行中无超时/心跳机制

超时检测仅针对 `assigned` 状态（小艾启动时检查 `now - assigned_at > timeout_minutes`）。`running` 状态没有任何超时。Agent 崩溃、死循环、hang 住后，任务永远卡在 `running`。规范第 188 行"Agent 宕机(分配后无响应) → 超时自动退回 pending"只覆盖了 `assigned` 状态，没有覆盖 `running` 状态。

修改建议:
- `running` 状态增加 `started_at` 时间戳 + `running_timeout_minutes` 字段
- 小艾定期巡检（不仅是启动时），对 `running` 超时的任务发出告警/强制退回

### [严重] #3 — 无并发写入保护，必现数据损坏

规范说"只有小艾和对应 agent 可以写自己的 .task 文件"，但这是纯约定，没有任何技术保障。所有 agent 和小艾共享同一个 Linux 用户和文件系统。以下是必然发生的竞态：

```
时间 T1: 小艾启动，检测 assigned 超时 → 准备写 status=pending
时间 T2: wiki-agent 刚好在此时收到任务 → 准备写 status=running
结果: 后写者覆盖前者，history 数组丢失，状态不一致
```

加上"文件变更必须 git commit"的要求，两个进程同时 git commit 必然产生 merge conflict，而规范里完全没有冲突解决策略。

修改建议:
- 使用文件锁 (flock/fcntl) 或原子写入（写临时文件 + rename）
- 去掉"每次变更必须 git commit"的硬性要求，改为批量提交或定时提交
- 或者：只让小艾写 .task 文件，agent 通过 Issue Comment 或消息通知小艾代写

### [严重] #4 — `depends_on` 无循环检测

`depends_on: [2]` 表示 task_id=3 依赖 task_id=2。但没有任何机制防止：
- 直接循环: task 2 depends_on [3], task 3 depends_on [2]
- 间接循环: task 1 → task 2 → task 3 → task 1

一旦出现循环依赖，所有涉及任务永远无法脱离 `pending`。

修改建议:
- 小艾分配任务时必须做拓扑排序/DFS 检测，拒绝含环的依赖链
- depends_on 只能引用 task_id < 自己的 task_id（单向约束）

### [严重] #5 — 打回修改后 agent 无感知机制

小艾验收不通过 → 将 `needs_review` 打回 `running`。但 agent 如何知道自己被"打回"了？规范只定义了"小艾 → tmux send-keys → wiki-agent"这条分配通道（assigned 时），但没有定义打回时的通知通道。Agent 不会主动轮询 .task 文件。

这意味着打回后的任务会永远留在 `running` 状态，agent 以为已经完成（它在等验收结果），小艾以为已经打回（它在等 agent 修改）。

修改建议:
- 明确定义打回通知协议：小艾通过 tmux send-keys 发送打回消息
- 或者在 .task 文件中增加 `action_required` 字段，agent 定期轮询

### [严重] #6 — 取消 running 中的任务无中断机制

用户说"取消" → 小艾设置 `status=cancelled`。但如果 agent 正在 `running` 状态疯狂执行，它完全不知道被取消了。Agent 会继续消耗资源直到自然结束。

修改建议:
- 小艾取消时发送 tmux C-c 中断信号给对应 agent
- 或者 agent 在执行循环中定期检查 .task 文件状态

---

## 中等问题（影响可靠性和可维护性）

(省略 8 条中等 + 11 条建议的详细内容，完整报告见 v1.1 复审)

---

## 总结

| 等级 | 数量 | 核心主题 |
|------|------|----------|
| 严重 | 6 | 死锁状态、无并发保护、无心跳、依赖环、打回无通知、取消无中断 |
| 中等 | 8 | 目录缺失、context 无约束、git 过频、信息冗余、标识双轨、检测不及时、无部分验收、级联取消未定义 |
| 建议 | 11 | 路径移植性、单点故障、优先级、能力匹配、完整性保护、删除/归档、迁移策略、身份验证、数据传递、协议未定义、history 膨胀 |

规范 v1.0 的核心思路（.task 文件 = 状态机载体 + Issue 关联）是清晰的，但目前在**并发安全**和**异常恢复**两个维度上存在根本性缺陷，在单 agent 单任务串行场景下可以工作，一旦进入多 agent 并行场景就会暴露竞态和死锁问题。建议在 v1.1 中优先修复 6 个严重问题再投入生产使用。