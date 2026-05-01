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