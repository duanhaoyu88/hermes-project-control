# .task 文件规范 v1.1

> 项目管控状态机的物理载体。一个文件 = 一个任务 = 完整生命周期。
> 
> v1.0 审查: docs/TASK_SPEC_REVIEW_v1.0.md (25 issues, 6 severe)
> v1.1 修复: 全部严重+中等问题，建议项纳入改善

## 运行环境

当前运行在 WSL (Ubuntu)，Windows F: 盘通过 `/mnt/f/` 挂载：
- `.task` 文件：`~/.hermes/projects/`（WSL ext4，原子写入可靠）
- GitHub 仓库：`/mnt/f/03_Github/`（NTFS 挂载）
- 知识库：`/mnt/f/04_Obsidian/`（NTFS 挂载）
- context 路径使用 `/mnt/f/...` 格式，所有 agent 均在 WSL 内运行

## 文件结构

```
~/.hermes/projects/          ← 小艾首次启动时创建
├── autosar-compile-3.task   ← {project-slug}-{task_id}.task
├── skill-governance-1.task
└── ...
```

文件名格式：`{project-slug}-{task_id}.task`，task_id 必须等于 GitHub Issue 任务清单中的 #编号。

## JSON Schema

```json
{
  "version": "1.1",
  "project": "AUTOSAR 知识库编译",
  "issue_url": "https://github.com/duanhaoyu88/hermes-project-control/issues/1",
  "task_id": 3,
  "title": "minerU 批量转换",
  "agent": "wiki-agent",
  "priority": "normal",
  "status": "assigned",
  "assigned_at": "2026-05-01T15:00:00+08:00",
  "started_at": null,
  "timeout_minutes": 30,
  "running_timeout_minutes": 120,
  "depends_on": [2],
  
  "context": {
    "data_path": "${OBSIDIAN_RAW}/*.pdf",
    "output_path": "${OBSIDIAN_EXTRACTED}/",
    "config_path": "${PROJECT}/.hermes/agents/wiki-agent/context.md"
  },
  
  "output": null,
  
  "acceptance": [
    "全部 PDF 转为 MD",
    "无 minerU 解析错误",
    ">200 页文档已分页处理"
  ],
  "acceptance_status": null,
  
  "history": [
    {
      "time": "2026-05-01T15:00:00+08:00",
      "from": "pending",
      "to": "assigned",
      "by": "小艾",
      "note": "依赖 #2 已完成，分配"
    }
  ],
  
  "hash": "sha256:abc123..."
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| version | string | ✅ | 规范版本，用于迁移 |
| project | string | ✅ | 项目名称 |
| issue_url | string | ✅ | GitHub Issue 完整 URL |
| task_id | int | ✅ | Issue 任务清单中的 #编号 |
| title | string | ✅ | 任务简述 |
| agent | string | ✅ | 执行 agent 名 |
| priority | enum | ✅ | `urgent`, `high`, `normal`, `low`，默认 `normal` |
| status | enum | ✅ | 见状态枚举 |
| assigned_at | ISO8601 | ✅ | 最近一次分配时间 |
| started_at | ISO8601 | ❌ | agent 开始执行时间（running 时必填） |
| timeout_minutes | int | ✅ | assigned→running 超时，默认 30 |
| running_timeout_minutes | int | ✅ | running 最大时长，默认 120 |
| depends_on | int[] | ❌ | 前置任务的 task_id（必须 < 自己的 task_id） |
| context | object | ❌ | agent 专属配置，按 agent 类型有 schema |
| output | object | ❌ | 任务产出（agent 完成时填写） |
| acceptance | string[] | ✅ | 验收标准列表，至少 1 条 |
| acceptance_status | bool[] | ❌ | 逐条验收结果（验收时填写，长度=acceptance） |
| history | object[] | ✅ | 状态变更历史，只追加不修改，上限 100 条 |
| hash | string | ✅ | 文件内容 SHA256（不含 hash 字段自身） |

### Context Schema（按 agent 类型）

**wiki-agent**:
```json
{
  "data_path": "string (required)",
  "output_path": "string (required)",
  "config_path": "string (optional)",
  "verify_script": "string (optional)"
}
```

**coco-agent**:
```json
{
  "repo_path": "string (required)",
  "branch": "string (optional)",
  "base_branch": "string (optional, default: main)",
  "build_cmd": "string (optional)",
  "test_cmd": "string (optional)"
}
```

路径支持环境变量：`${OBSIDIAN_RAW}`, `${PROJECT}`, `${HOME}` 等。

### 优先级排序

分配顺序：`urgent > high > normal > low`，同优先级按 `assigned_at` 先后。

## 状态枚举

| 状态 | 含义 | 谁触发 | 出口 |
|------|------|--------|------|
| `pending` | 待分配 | 初始/超时退回/失败重试 | → assigned |
| `assigned` | 已分配 | 小艾 | → running (agent确认) / → pending (超时) |
| `running` | 执行中 | agent | → needs_review (完成) / → pending (超时) / → failed (报错) |
| `needs_review` | 待验收 | agent | → done (通过) / → running (打回) |
| `done` | 已完成 | 小艾 | (终态) |
| `failed` | 失败 | agent/小艾 | → pending (小艾手动重试) / → cancelled (放弃) |
| `cancelled` | 取消 | 小艾 | (终态，等价 done 用于依赖链) |

## 状态流转图

```
                ┌──────────┐
                │ pending  │ ← 初始 / 超时退回 / 失败重试
                └────┬─────┘
                     │ 小艾分配 (检查依赖 + 循环检测)
                     ▼
                ┌──────────┐
           ┌────│ assigned │────超时────▶ pending
           │    └────┬─────┘
           │         │ agent 确认收到
           │         ▼
           │    ┌──────────┐
           │    │ running  │────超时────▶ pending
           │    └──┬───┬───┘
           │       │   │
           │       │   └── 报错 ──▶ failed ──▶ pending (重试) / cancelled (放弃)
           │       │
           │       │ agent 写完成 Comment
           │       ▼
           │    ┌──────────────┐
     打回  │    │ needs_review │
     通知  │    └──────┬───────┘
           │       ┌───┴───┐
           │       ▼       ▼
           │   ┌──────┐ ┌────────┐
           │   │ done │ │ running│ (打回 + 未通过项编号)
           │   └──────┘ └────────┘
           │
cancel ────┴── 任意非终态 → cancelled (触发 C-c 中断 agent)
                cancelled ≡ done (从依赖链角度)
```

## 并发安全

**原子写入**：所有对 .task 文件的修改使用 write-temp-then-rename 模式：
```
1. 写 /tmp/{filename}.tmp
2. mv /tmp/{filename}.tmp ~/.hermes/projects/{filename}  (Linux 上 rename 是原子的)
```

读取方始终看到完整文件（旧版本或新版本，不会看到半截文件）。

**写入权限**：小艾有全部写入权限。agent 只能通过约定写 status=running 和 status=needs_review，不强制技术限制（依赖运维隔离）。

## 操作协议

### 初始化（小艾首次启动）

```
1. mkdir -p ~/.hermes/projects/
2. 如果已有 .task 文件 → 迁移旧版本（见版本迁移）
```

### 小艾启动时 / 定期巡检（每 5 分钟）

```
对每个 ~/.hermes/projects/*.task:

  status=assigned 且 now - assigned_at > timeout_minutes:
    → 原子写入 status=pending, 追加 history

  status=running 且 started_at 存在且 now - started_at > running_timeout_minutes:
    → 原子写入 status=pending, 追加 history: {note: "running 超时"}
    → 通知用户：「任务 #N 超时，已退回 pending」

  status=needs_review:
    → 加入验收队列

  status=done:
    → 检查是否解锁了其他 pending 任务

  status=failed:
    → 通知用户：「任务 #N 失败，等待处理」
```

### 小艾分配任务

```
前提:
  1. depends_on 中的所有 task_id 都是 done 或 cancelled 状态
  2. 拓扑检查: 遍历所有 .task 文件，从当前任务出发做 DFS，确认无环
  3. agent 当前没有 running 任务（扫描所有 .task 文件）
  4. context 通过对应 agent 的 schema 校验

步骤:
  1. 原子写入 .task 文件 (status=assigned, assigned_at=now)
  2. 构造任务消息 → tmux send-keys 发给 agent（见消息格式）
  3. 追加 history
```

### Agent 接收任务

```
1. 收到消息 → 读 .task 文件
2. 原子写入 status=running, started_at=now
3. 追加 history
4. 读 Issue → 读专属配置 → 开始执行
```

### Agent 完成任务

```
1. 原子写入 output 字段（产出路径等）
2. 写 Issue Comment（产出摘要 + 路径）
3. 原子写入 status=needs_review, acceptance_status=null
4. 追加 history
```

### Agent 报错

```
1. 原子写入 status=failed
2. 追加 history: {note: "错误详情: ..."}
3. 写 Issue Comment
```

### 小艾验收

```
1. 读 .task → 读 Issue Comment → 检查产出
2. 填写 acceptance_status: [true, true, false, ...] （逐条标注）
3. 通过 (全部 true):
   → 原子写入 status=done
   → 追加 history: {note: "验收通过"}
   → 写 Issue Comment: 「验收通过: [时间]」
4. 不通过 (部分 false):
   → 原子写入 status=running, started_at=now（重置计时器）
   → 追加 history: {note: "打回: 未通过项 #3, #4"}
   → tmux send-keys 发送打回通知给 agent
   → 写 Issue Comment: 「验收不通过: [问题列表]」
```

### 小艾取消任务

```
1. 原子写入 status=cancelled
2. 如果 agent 在 running → tmux send-keys C-c 中断
3. 追加 history: {note: "用户取消"}
4. 依赖链处理: cancelled ≡ done（下游任务不受阻）
```

### Agent 打回后重做

```
1. agent 收到打回通知 → 读 .task → 读 acceptance_status
2. 定位未通过的验收项 → 修改
3. 完成 → 重走"Agent 完成任务"流程
```

## 任务消息格式（wire format）

小艾通过 tmux send-keys 发送给 agent 的消息，JSON 格式单行：

```json
{"type":"task_assign","project":"AUTOSAR","task_id":3,"title":"minerU 批量转换","task_file":"~/.hermes/projects/autosar-compile-3.task","issue_url":"https://github.com/.../issues/1"}
```

打回通知：
```json
{"type":"task_reject","task_id":3,"failed_items":[3,4],"note":"输出缺少交叉引用"}
```

取消通知：
```json
{"type":"task_cancel","task_id":3}
```

## 与 Issue 的关系

.task 文件是**第一手 truth**（离线自洽）。Issue 是展示层和备份。同步规则：
- .task status 变更 → 小艾负责同步更新 Issue 任务清单状态
- Issue Comment 是 public log，.task history 是 machine log
- 冲突时以 .task 文件为准

## 数据传递（任务间）

task_id=N 完成时 agent 填写 `output` 字段。小艾分配 task_id=M（depends_on 包含 N）时，将 upstream output 注入 M 的 context：

```json
// task 2 完成时的 output:
"output": {"extracted_dir": "/mnt/f/04_Obsidian/raw_extracted/", "file_count": 14}

// task 4 的 context (depends_on: [2]):
"context": {
  "data_path": "${UPSTREAM:2.output.extracted_dir}",  // 引用上游产出
  ...
}
```

## 版本迁移

v1.0 → v1.1 迁移：
- 新增字段使用默认值（priority=normal, running_timeout_minutes=120 等）
- 旧文件无 hash 字段 → 启动时计算并写入
- history 超过 100 条 → 保留最后 100 条
- 迁移在首次读取时自动执行，写入更新后的文件

## 归档策略

- done/cancelled 文件的 .task 保留 90 天，之后移到 `~/.hermes/projects/archive/`
- failed 文件保留直到处理（重试或取消）
- 对应 Issue 任务项被删除时 → 小艾移动 .task 到 archive

## 边界条件

1. Agent 正在 running → 不分配新任务（扫描所有 task 文件确认）
2. depends_on 未全 done/cancelled → pending
3. depends_on 含循环 → 小艾拒绝分配（DFS 检测）
4. assigned 超时 → pending
5. running 超时 → pending（通知用户）
6. needs_review 超时 → 无超时（验收是人工步骤）
7. 取消 → 发 C-c + status=cancelled + 依赖链不阻塞
8. .task 文件损坏 → 从 Issue 重建（hash 校验失败触发）
9. GitHub 不可用 → .task 文件自洽运行，恢复后批量同步 Issue
10. 多个 urgent 任务 → 按 assigned_at 先后

## Agent 能力注册

小艾维护 agent 能力映射（在 MEMORY 中）：

```
wiki-agent: [knowledge-base, pdf-conversion, document-organization]
coco-agent: [code-implementation, code-review, testing]
```

分配前检查任务所需能力与 agent 能力匹配。

## 信任模型

本规范不提供应用层认证。依赖运维层面的进程隔离：
- 不同 agent 在不同 tmux session 中运行
- 文件系统共享但通过原子写入避免损坏
- 恶意篡改不在防御范围内