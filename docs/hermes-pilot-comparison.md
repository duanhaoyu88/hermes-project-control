# Hermes vs Pilot Protocol — 通信方案对比评估

审查时间: 2026-05-02
审查团队: 小艾(PM) + QA(审查)

## 结论: 不切换。Hermes 胜出 (42 vs 40, 不计无关维度 40 vs 30)

Pilot Protocol 是为全球多机 P2P 设计的，Hermes 是 WSL 单机 4 agent 设计的。Pilot 80% 的复杂度在 Hermes 场景下是零价值的。

## 六维对比

| 维度 | Hermes | Pilot | 胜方 |
|------|--------|-------|------|
| 可靠性 | 9/10 | 7/10 | Hermes |
| 实时性 | 7/10 | 7/10 | 平局 |
| 部署成本 | 10/10 | 3/10 | Hermes |
| 扩展性 | 2/10 | 10/10 | Pilot |
| 故障恢复 | 5/10 | 8/10 | Pilot |
| 运维复杂度 | 9/10 | 5/10 | Hermes |

- 计入全部 6 维: Hermes 42 vs Pilot 40
- 不计扩展性 (Hermes 不需要): Hermes 40 vs Pilot 30

## 细节

### 可靠性 (Hermes ✅)
Hermes 无 daemon、无网络依赖、无加密隧道——少一个组件少一类故障。
Pilot 依赖 daemon + registry + STUN beacon + 加密协商，故障面更大。

### 实时性 (平局)
- Hermes 直连更快 (tmux send-keys <100ms vs 网络往返 >50ms)
- Pilot 异步通知更实时 (push vs poll)
- 对 Hermes 场景，task_assign 是高频操作，直连延迟更关键

### 部署成本 (Hermes ✅ 压倒性)
Pilot 需要: curl 安装 pilotctl + registry 服务 + STUN beacon + daemon + N×(N-1) 次握手
Hermes 仅需: 已有 tmux + profile 目录

### 扩展性 (Pilot ✅ 但无关)
Pilot 原生支持多机、动态注册、能力标签、自动路由。
但 Hermes 定位就是单机 4 agent，多机扩展不是需求。

### 故障恢复 (Pilot ✅)
Pilot 有结构化超时/自动重试/watchdog/webhook 通知。
这是唯一真实差距，需要弥补。

### 运维复杂度 (Hermes ✅)
文件系统操作 >> 网络协议栈排查。
出问题时 cat .task + ls /tmp 秒排查，Pilot 需要查网络/daemon/registry/信任。

## 核心洞察

### 设计目标错位

| | Pilot Protocol | Hermes |
|---|---|---|
| 设计目标 | 全球多机 P2P | WSL 单机 4 agent |
| 核心复杂度 | NAT穿透+加密+注册+握手 | 文件系统+tmux |
| 在 Hermes 场景 | 80% 功能零价值 | 100% 功能条条有用 |

### 状态机同构
两者的 task 状态机几乎一致:
- Pilot: NEW → ACCEPTED → EXECUTING → SUCCEEDED
- Hermes: assigned → running → needs_review → done

验证了 Hermes 的设计方向正确。

### 三个值得借鉴的模式

1. **Event Stream 代替文件轮询**
2. **Capability Tags 代替硬编码映射**
3. **Polo Score 引入质量追踪**

## 决策记录

- ✅ 不切换到 Pilot Protocol
- ✅ 保持当前 /tmp + .task + tmux 方案
- 📋 借鉴 3 个模式纳入演进路线 (Issue #4)

## 相关链接

- QA 原始输出: `/tmp/hermes-qa.status`
- Pilot 源码: `https://github.com/TeoSlayer/pilot-skills`
- Obsidian 双向链接: [[hermes-communication-evolution-v1.2]]
