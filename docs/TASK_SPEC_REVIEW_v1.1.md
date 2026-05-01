# .task 文件规范 v1.1 — 复审报告

审查人: Hermes Agent (独立复审)
审查时间: 2026-05-02
审查对象: ~/.hermes/TASK_SPEC.md (v1.1, 全面重写)
参考: v1.0 审查报告 (25 issues, 6 severe)

---

## 复审结论：通过，建议投产 ✅

---

## 1. 严重问题修复验证

全部 6 个严重问题已修复：

| # | 问题 | 修复 | 验证 |
|---|------|------|------|
| 1 | failed 死胡同 | `failed → pending` (重试) / `failed → cancelled` (放弃) | ✅ 流转图自洽 |
| 2 | running 无超时 | 增加 `started_at` + `running_timeout_minutes` + 定期巡检 | ✅ |
| 3 | 并发无保护 | 原子写入 (temp + rename)，移除每变更 git commit | ✅ |
| 4 | 依赖环 | DFS 拓扑检测 + depends_on 只能引用 < 自己的 task_id | ✅ 双重防护 |
| 5 | 打回无通知 | tmux send-keys 发送 task_reject 消息 | ✅ wire format 已定义 |
| 6 | 取消无中断 | tmux C-c + status=cancelled | ✅ |

## 2. 中等问题修复验证

全部 8 个中等问题已修复：

| # | 问题 | 修复 |
|---|------|------|
| 7 | projects/ 目录缺失 | 初始化流程明确 mkdir |
| 8 | context 无约束 | 按 agent 类型定义 schema (wiki-agent/coco-agent) |
| 9 | git 过频 | 移除硬性要求，改为批量策略 |
| 10 | history 冗余 | 保留 history 但 status 作为缓存，一致性约束明确 |
| 11 | 标识双轨 | task_id 强制对齐 Issue #编号 |
| 12 | 检测仅启动时 | 增加定期巡检（每 5 分钟） |
| 13 | 无部分验收 | 增加 acceptance_status 数组，逐条标注 |
| 14 | cancelled 级联 | cancelled ≡ done（从依赖链角度不阻塞下游） |

## 3. 建议项修复验证

11 条建议全部纳入：

- 路径变量 `$OBSIDIAN_RAW`, `$PROJECT` 等
- .task 文件为第一手 truth（离线自洽），Issue 为展示层
- priority 字段 (urgent/high/normal/low)
- agent 能力注册 (MEMORY 中维护)
- hash 字段 (SHA256)
- 归档策略 (90天 → archive/)
- 版本迁移策略
- 信任模型声明
- 任务间数据传递 (`output` + `${UPSTREAM:N.output.xxx}`)
- wire format 定义 (task_assign/task_reject/task_cancel)
- history 上限 100 条

## 4. 新发现（不阻塞投产）

### [极低] #A — hash 需要 canonical JSON

`hash` 字段计算需要 JSON 的 canonical 序列化（键排序、无多余空格）。标准 JSON 库不保证序列化一致性。建议实现时使用 `json.dumps(obj, sort_keys=True, separators=(',', ':'))`。

### [极低] #B — 小艾需要确定运行模型

5 分钟巡检依赖小艾在后台持续运行。如果小艾只在用户消息触发时才执行，巡检不会生效。建议：
- 如果小艾在 tmux 中持久运行 → 用 sleep + loop 或 cron 触发巡检
- 如果小艾按需启动 → 启动时巡检，受理局限

### [极低] #C — acceptance_status 生命周期

验收通过后 `acceptance_status: [true, true, true]` 保留在 .task 中，但如果任务被打回修改后重新验收，旧值会被覆盖。这符合预期，但可以在规范中显式说明："打回 → running 时，acceptance_status 重置为 null"。

### [极低] #D — 两个超时字段命名

`timeout_minutes` (assigned→running) 和 `running_timeout_minutes` 命名不够对称。建议改为 `assign_timeout_minutes` 和 `run_timeout_minutes` 或 `exec_timeout_minutes`。

## 5. 过度修正检查

无过度修正。以下设计合理：
- 双重循环防护（DFS + task_id 约束）不是冗余，是 defense-in-depth
- hash + 原子写入方案简洁有效，未引入不必要的复杂度
- context schema 为每种 agent 定义，未过度泛化

## 6. 投产建议

**可以投产。** 建议：
1. 首批 3-5 个任务验证关键路径（pending → assigned → running → needs_review → done）
2. 特别关注原子写入和巡检的正确性
3. 验证后放量到更多项目

核心关注点：
- 实现 hash 计算时使用 canonical JSON 序列化
- 确保小艾有定期巡检机制（cron 或持久进程）
