# Hermes 通信演进 v1.2 — 借鉴 Pilot 三模式设计

> 2026-05-02 | 小艾(PM) 设计 | 基于 QA 对比分析报告

## 原则：加法不改减法

三项改造全部遵循"新增字段/新增脚本，不动现有逻辑"。现有 patrol + .task + tmux 原封不动，新东西叠加在上面。出问题回滚就是删新文件。

---

## 1. Event Stream — 秒级通知

### 问题
当前小艾每 5 分钟巡检 /tmp/hermes-*，agent 完成到 PM 感知最差差 5 分钟。

### 方案：inotify + patrol 双层
```
┌─────────────────────────────────────────────┐
│  task_watcher.py (新增, 常驻)               │
│  inotify 监听:                              │
│    /tmp/hermes-*.status   → 创建/修改       │
│    ~/.hermes/projects/*.task → 修改         │
│                                             │
│  检测到变化 → 写入 ~/.hermes/events.fifo    │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│  小艾 (已有)                                │
│  每次响应用户前:                             │
│    1. 读 events.fifo (非阻塞)               │
│    2. 有事件 → 即时处理                     │
│    3. 无事件 → 回退 patrol (5min 兜底)     │
└─────────────────────────────────────────────┘
```

### 稳定性分析
- inotify: Linux kernel 2.6.13+ 原生支持，零外部依赖
- /tmp 在 WSL 上是 ext4，inotify 完全可靠
- 极端情况 inotify 丢事件（理论上 ext4 不会丢）→ patrol 5min 兜底
- task_watcher.py 崩溃 → patrol 照常工作，系统降级到 v1.1 行为
- **不改动 patrol、不改动 .task、不改动 agent SOUL**

### 新增文件
- `~/.hermes/scripts/task_watcher.py` — inotify 监听器 (~80 行)
- `~/.hermes/events.fifo` — 命名管道 (mkfifo)

### 启动方式
```bash
python3 ~/.hermes/scripts/task_watcher.py &
```
追加到小艾启动脚本。

---

## 2. Capability Tags — 能力标签路由

### 问题
当前小艾 MEMORY 硬编码 agent 能力映射。加新 agent 或改能力要改 MEMORY。

### 方案：从 agent profile 读取能力声明
```
TASK_SPEC.md 新增字段:
  "required_capabilities": ["pdf-conversion", "markdown"]  (可选)

agent AGENTS.md 新增:
  ## Capabilities
  - pdf-conversion
  - document-organization
  - markdown-editing
```

### 分配逻辑（小艾分配任务时）
```
1. 读 .task → required_capabilities
2. 如果为空 → 回退到 MEMORY 硬编码（向后兼容）
3. 读候选 agent 的 AGENTS.md → ## Capabilities
4. required ⊆ agent_capabilities → 可以分配
5. 不匹配 → 跳过此 agent
6. 多个 agent 匹配 → 优先当前空闲的
```

### 稳定性分析
- 纯数据模型扩展，零逻辑修改
- required_capabilities 为空 → 走旧逻辑，完全兼容
- AGENTS.md 无 Capabilities 段落 → 降级到 MEMORY 硬编码
- MEMORY 硬编码保留作为最终 fallback
- **不改动 agent 行为、不改动 task_assign 协议**

### 新增/修改文件
- `TASK_SPEC.md` +3 行（新增字段说明）
- `wiki-agent/AGENTS.md` +3 行
- `qa-agent/AGENTS.md` +3 行
- `coco-agent/AGENTS.md` +3 行

---

## 3. Quality Score — 完成质量追踪

### 问题
当前不知道哪个 agent 靠谱，分配全凭经验。

### 方案：在 .task history 加质量标记
```json
"history": [
  {...},
  {"time": "...", "from": "needs_review", "to": "done",
   "by": "小艾", "quality": "passed"},
  {"time": "...", "from": "needs_review", "to": "running",
   "by": "小艾", "quality": "rejected:3,4"}
]
```

### 评分计算（小艾分配前查询）
```
对每个 agent，扫描所有 .task 文件 history:
  total = 该 agent 的 done + rejected 次数
  passed = quality=="passed" 的次数
  score = passed / total (0.0 ~ 1.0)

分配优先级: score 高 → 优先（同能力下）
首次分配: 无 history → score = null → 按 MEMORY 默认
```

### 稳定性分析
- 纯增量：history 数组多一个可选字段
- quality 字段缺失 → 不计入统计，不影响任何逻辑
- 评分只在分配时参考，不影响其他流程
- **不改动 agent 行为、不改动状态机**

### 修改文件
- `TASK_SPEC.md` +5 行（history quality 字段说明）
- `task_utils.py` +15 行（scan_quality 函数）

---

## 实施路线

| 优先级 | 模式 | 复杂度 | 预计耗时 | 风险 |
|--------|------|--------|---------|------|
| P0 | Quality Score | 低 | 15 min | 零 |
| P1 | Capability Tags | 低 | 20 min | 零 |
| P2 | Event Stream | 中 | 30 min | 极低 |

**实施顺序**：从最简单到最复杂，每步验证再下一步。

### 验收标准
- [ ] Quality: `task_utils.py --quality wiki-agent` 返回正确分数
- [ ] Capability: 创建测试 .task（含 required_capabilities），分配逻辑正确匹配
- [ ] Event: agent 写完 status 文件后 <2s 小艾收到通知
- [ ] 回归: 不带新字段的旧 .task 文件正常工作

## 相关链接

- Issue: [#4](https://github.com/duanhaoyu88/hermes-project-control/issues/4)
- QA 对比报告: [hermes-pilot-comparison.md](hermes-pilot-comparison.md)
- Obsidian 双向链接: [[hermes-pilot-comparison]]
