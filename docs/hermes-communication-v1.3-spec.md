# Hermes 通信机制 v1.3 — 实施规范

> 2026-05-02 | 小艾(架构师) | 引入 PM-agent 调度层

## 架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                      小艾 (架构师/决策者)                       │
│  tmux session: xiao-ai | model: deepseek-v4-pro              │
│                                                              │
│  每次响应用户前:                                               │
│    1. cat ~/.hermes/events.jsonl → 有新事件?                  │
│    2. 无 → fallback: ls /tmp/hermes-*.status                 │
│    3. 处理完成事件 → 验收 → 通知用户                            │
│                                                              │
│  task_watcher.py (2s 轮询)                                   │
│  task_patrol.py (5min 兜底)                                  │
└──────────────────────┬───────────────────────────────────────┘
                       │
                  tmux send-keys
                  (单条完整消息)
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    PM-agent (执行调度者)                        │
│  tmux session: pm-agent | model: deepseek-v4-flash           │
│                                                              │
│  收到小艾指令 → 拆分 → 分配 QA/Coco/Wiki → 协调闭环 → 汇总回报  │
│  完成后: /tmp/hermes-pm-{task_id}.status                     │
└────┬──────────────────┬──────────────────┬──────────────────┘
     │                  │                  │
tmux send-keys     tmux send-keys     tmux send-keys
     ▼                  ▼                  ▼
┌──────────┐    ┌──────────┐    ┌──────────┐
│QA-agent  │    │Coco-agent│    │Wiki-agent│
│v4-flash  │    │v4-pro    │    │v4-flash  │
│          │    │          │    │          │
│完成后:    │    │完成后:    │    │完成后:    │
│/tmp/     │    │/tmp/     │    │/tmp/     │
│hermes-qa │    │hermes-   │    │hermes-   │
│-{id}.    │    │coco-{id} │    │wiki-{id} │
│status    │    │.status   │    │.status   │
└──────────┘    └──────────┘    └──────────┘
```

## 通信链路（新）

```
小艾 ──(send-keys)──→ PM-agent ──(send-keys)──→ QA/Coco/Wiki
  ▲                       │                          │
  │                  [汇总结果]                  [各自完成]
  │                       │                          │
  └─── events.jsonl ←─ /tmp/hermes-pm-*.status ←─ /tmp/hermes-{qa|coco|wiki}-*.status
```

### 链路说明

| 环节 | 方式 | 说明 |
|------|------|------|
| 小艾 → PM-agent | tmux send-keys | 任务描述，单条完整消息 |
| PM-agent → 执行 agent | tmux send-keys | 拆分后的子任务，单条完整消息 |
| 执行 agent → PM-agent | tmux send-keys 通知 + /tmp status | 完成后通知 PM-agent |
| PM-agent → 小艾 | /tmp/hermes-pm-{task_id}.status | 汇总后写状态文件 |
| 小艾感知 | events.jsonl (watcher) → fallback ls /tmp | 同 v1.2 |

## 核心组件

| 组件 | 文件 | 职责 | 故障影响 |
|------|------|------|---------|
| task_watcher | `~/.hermes/scripts/task_watcher.py` | 2s 轮询 → events.jsonl | 降级到 patrol 5min |
| task_patrol | `~/.hermes/scripts/task_patrol.py` | 超时检测 + 状态流转 | 任务可能超时未回收 |
| task_utils | `~/.hermes/scripts/task_utils.py` | .task 原子读写 + quality 统计 | 无法创建/读取 .task |
| herm 脚本 | `~/bin/herm` | agent 生命周期管理 | 需手动 tmux 操作 |

## 通信规则（强制）

### PM → Agent
- ⚠️ **一条 send-keys 包含全部上下文**，禁止多条连续发送
- **消息必须带身份前缀**：`[sender-session] 消息内容`
  - 例：`[pm-agent] 审查任务: 检查 TASK_SPEC.md`
  - 例：`[qa-agent] 审查完成，问题 3 处，见 /tmp/hermes-qa-4.status`
- 接收方根据前缀立即知道谁发的、往哪个 session 回复
- 多行内容 → 先写临时文件，send-keys 仅传文件路径

### PM-agent → 执行 agent
- 拆分任务后，每条 send-keys 完整描述子任务
- QA↔Coco 闭环：PM 先派 QA → QA 审完通知 PM → PM 派 Coco(附 QA 问题) → Coco 修完通知 PM → PM 派 QA 复审
- 最多 3 轮，超了汇报小艾

### 执行 agent → PM-agent
- 完成后必须写 `/tmp/hermes-{agent}-{task_id}.status`
- JSON 格式：`{"agent":"qa","task_id":3,"status":"needs_review","time":"ISO8601","ttl_seconds":120,"summary":"..."}`
- 完成后立即停止，等验收指令

### PM-agent → 小艾
- 全部子任务闭环后 → 汇总写 `/tmp/hermes-pm-{task_id}.status`
- 阻塞/需决策 → 立即写 status(blocked) + tmux send-keys 通知

### 验收流程
1. 小艾通过 events.jsonl 或 patrol 感知 PM-agent 完成
2. 读 .task 文件 → 逐条验收
3. 通过 → status=done, quality=passed
4. 打回 → status=running, quality=rejected:N,M, 发修正指令

## 文件约定

| 文件 | 格式 | 说明 |
|------|------|------|
| `/tmp/hermes-{agent}.status` | JSON | 单任务 agent 通知 |
| `/tmp/hermes-{agent}-{task_id}.status` | JSON | 并行任务通知（防覆盖） |
| `/tmp/hermes-pm-{task_id}.status` | JSON | PM-agent 汇总通知 |
| `~/.hermes/events.jsonl` | JSONL | watcher 事件流，小艾消费 |
| `~/.hermes/projects/{project}-{task_id}.task` | JSON | 任务状态机 |
| `~/.hermes/profiles/{agent}/AGENTS.md` | Markdown | agent 能力 + 通信协议 |

## 新增 Agent 检查清单

- [ ] `hermes profile create <name> --clone-from default`
- [ ] 替换 state.db（防身份混淆）
- [ ] 清理 MEMORY 残留（尤其 USER.md 中的「小艾」）
- [ ] 编写 SOUL.md + AGENTS.md（含通信协议）
- [ ] AGENTS.md 含 Capabilities 段落
- [ ] 添加 `herm {name}-start|stop|ask|log|attach` 到 `~/bin/herm`
- [ ] 通信测试：send-keys → agent 写 status → watcher 检测
- [ ] 注册到 hermes-team docs/registry.md

## 已知风险

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| tmux session 意外退出 | 低 | 任务中断，需重分配 | patrol 超时回收 → pending |
| send-keys 消息截断（特殊字符） | 中 | agent 收到不完整指令 | 复杂内容走临时文件 |
| PM-agent 层阻塞 | 中 | 执行链断裂 | patrol 5min 检测 + 小艾直调兜底 |
| watcher 进程崩溃 | 低 | 通知延迟 5min | patrol 兜底 |
| events.jsonl 无限增长 | 低 | 磁盘占用 | watcher 每 10min 截断到 500 行 |
| .task 并发写入冲突 | 极低 | 数据损坏 | 原子写入（temp-then-rename） |
| agent 完成但不写 status | 中 | 不知道完成 | patrol 检测 + tmux capture-pane 兜底 |

## 变更记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.2 | 2026-05-02 | E2E 验证通过，QA/Coco/Wiki 三 agent |
| v1.3 | 2026-05-02 | 引入 PM-agent 调度层，小艾不再直调执行 agent |

## 验证记录

2026-05-02 v1.2 E2E：QA ✅ / Coco ✅ / Wiki ✅ — 三 agent 全链路正常
2026-05-02 v1.3：PM-agent 层待 E2E 验证
