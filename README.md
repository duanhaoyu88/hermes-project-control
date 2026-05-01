# Hermes Project Control

> 以 GitHub Issues 为多 Agent 项目状态机，微信一句话管理项目进度。
>
> 📄 知识库: `/mnt/f/04_Obsidian/ai-agent/projects/hermes-project-control.md`

## 架构

```
用户微信 → 小艾（API层）→ GitHub Issues（状态机）← 子 Agent
                ↓
           Obsidian（知识库 + 项目文档）
```

## 原始 Skill

整合自三个自建 skill：
- `github-issues-project-tracking`（80行）— GitHub Issues 状态机 + 微信指令
- `project-management`（41行）— Phase 1-4 通用流程
- `qmd`（116行）— FlowState-QMD 外部工具

## 路线图

### Phase 1: 整合设计
- [ ] 提取三个 skill 的核心逻辑
- [ ] 设计统一 project-control skill 结构

### Phase 2: 实现
- [ ] 重写为单一 skill（四层标准）
- [ ] 完整指令映射表
- [ ] 多 Agent 协调协议

### Phase 3: 基础设施
- [ ] Obsidian 项目页面
- [ ] 示例 Issue 测试

### Phase 4: 长期运营
- [ ] 新项目强制走 project-control
- [ ] 按月审查活跃项目

## 关联

- 治理体系: [hermes-skill-governance](https://github.com/duanhaoyu88/hermes-skill-governance)
- Agent 协作: wiki-agent, coco-agent
- 知识库: `/mnt/f/04_Obsidian/`
