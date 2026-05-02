# Hermes Project Control

> 以 GitHub Issues 为多 Agent 项目状态机，微信一句话管理项目进度。
>
> 📄 知识库: `/mnt/f/04_Obsidian/ai-agent/projects/hermes-project-control.md`
> 📋 核心规范: [TASK_SPEC.md](./TASK_SPEC.md) (v1.1, 已通过审查)

## 架构

```
用户微信 → 小艾（PM/API层）
              ├─ 读 .task 文件（状态机）
              ├─ 读写 GitHub Issues（看板+日志）
              ├─ tmux send-keys → wiki-agent
              ├─ delegate_task → coco-agent
              └─ Obsidian（项目设计文档）
```

## 当前状态

| 阶段 | 内容 | 状态 |
|------|------|------|
| Phase 1 | project-control skill v2.3（7 操作 + 验收标准 + 启动协议） | ✅ |
| Phase 2 | .task 规范 v1.1（状态机 + 并发安全 + 异常恢复） | ✅ 通过审查 |
| Phase 3 | 通信演进 v1.2（借鉴 Pilot 三模式, Issue #4） | ⏳ |
| Phase 4 | 首个实战项目（AUTOSAR 编译, Issue #1） | ⏳ |
| Phase 5 | 长期运营（按月审查、agent 能力注册） | ⏳ |

## 活跃 Issues

| # | 标题 | 状态 |
|---|------|------|
| [#1](https://github.com/duanhaoyu88/hermes-project-control/issues/1) | AUTOSAR CP 知识库增量编译 | ⏳ 阻塞 (#3 未完成) |
| [#2](https://github.com/duanhaoyu88/hermes-project-control/issues/2) | Hermes 团队管理 | ✅ 4/4 |
| [#3](https://github.com/duanhaoyu88/hermes-project-control/issues/3) | project-control 自举 | ✅ 5/5 |
| [#4](https://github.com/duanhaoyu88/hermes-project-control/issues/4) | 通信方案演进 v1.2 | ⏳ 0/3 |

## 核心文件

| 文件 | 用途 |
|------|------|
| [TASK_SPEC.md](./TASK_SPEC.md) | .task 文件规范 v1.1 |
| [docs/TASK_SPEC_REVIEW_v1.0.md](./docs/TASK_SPEC_REVIEW_v1.0.md) | v1.0 审查报告（25 issues） |
| [docs/TASK_SPEC_REVIEW_v1.1.md](./docs/TASK_SPEC_REVIEW_v1.1.md) | v1.1 复审报告（通过） |
| [docs/hermes-pilot-comparison.md](./docs/hermes-pilot-comparison.md) | Pilot Protocol 对比评估 |
| [docs/hermes-communication-evolution-v1.2.md](./docs/hermes-communication-evolution-v1.2.md) | 通信演进 v1.2 设计 |
| `~/.hermes/skills/project-control/SKILL.md` | 操作 skill（agent 加载） |
| `~/.hermes/projects/` | .task 文件运行时目录 |

## Agent 角色

| Agent | 职责 | 触发方式 |
|-------|------|---------|
| 小艾 | PM：需求分析、任务分配、验收 | 微信/CLI |
| wiki-agent | 知识库操作、文档处理 | tmux + 任务消息 |
| coco-agent | 编码实现、代码审查 | delegate_task |
| QA agent | 独立审查、方案评估 | tmux + 任务消息 |

## 关联

- 治理体系: [hermes-skill-governance](https://github.com/duanhaoyu88/hermes-skill-governance)
- 团队管理: [hermes-team](https://github.com/duanhaoyu88/hermes-team)
- 知识库: `/mnt/f/04_Obsidian/`
