# Hermes 通信机制 v1.2 — 实施规范

> 2026-05-02 | E2E 验证通过 (QA/Coco/Wiki 三 agent)

## 架构

```
小艾 (PM)
  ├─ task_watcher.py (2s poll) → events.jsonl
  ├─ task_patrol.py (5min, 兜底)
  └─ tmux send-keys (单条完整消息)
         │              │              │
    wiki-agent     qa-agent     coco-agent
    v4-flash       v4-flash     v4-pro
         │              │              │
    /tmp/hermes-{agent}-{id}.status  ← agent 完成后写入
```

## 核心组件

| 组件 | 路径 | 职责 | 故障影响 |
|------|------|------|---------|
| task_watcher | `~/.hermes/scripts/task_watcher.py` | 2s 轮询 → events.jsonl | 降级 patrol 5min |
| task_patrol | `~/.hermes/scripts/task_patrol.py` | 超时 + 状态流转 | 任务超时未回收 |
| task_utils | `~/.hermes/scripts/task_utils.py` | 原子读写 + quality 统计 | 无法操作 .task |
| herm 脚本 | `~/bin/herm` | agent 生命周期 | 需手动 tmux |

## 通信规则

- ⚠️ 一条 send-keys 包全部上下文（多条会互相打断）
- Agent 完成后写 `/tmp/hermes-{name}-{task_id}.status` JSON
- PM 消费 events.jsonl → 验收 → quality=passed/rejected
- patrol 5min 兜底，watcher 崩溃不影饷

## 新增 Agent 清单

- `hermes profile create <name> --clone-from default`
- 替换 state.db, 写 SOUL.md + AGENTS.md
- AGENTS.md 含 Capabilities + 通信协议
- 添加到 `~/bin/herm`
- E2E 通信测试
- 注册到 hermes-team registry.md

## 已知风险

| 风险 | 缓解 |
|------|------|
| tmux session 退出 | patrol 超时回收 |
| send-keys 截断 | 复杂内容走临时文件 |
| watcher 崩溃 | patrol 兜底 |
| agent 不写 status | patrol + Issue Comment |

## 验证

2026-05-02: QA ✅ Coco ✅ Wiki ✅
