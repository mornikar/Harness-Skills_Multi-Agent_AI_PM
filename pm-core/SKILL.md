---
name: pm-core
description: |
  AI产品经理团队的共享运行时内核（v3.1 Ironforge 架构）。
  提供上下文协议、Agent生命周期规范、通信协议、恢复配方、默认行为技能。
  v3.1 新增：安全加固、稳定性加固、可观测性加固。
  所有 Skill 自动继承 pm-core，无需显式配置。
  
  核心能力：
  - 上下文自适应协议（FULL/PARTIAL/MINIMAL 三级降级）
  - Agent 生命周期规范（启动→执行→交付）
  - 统一通信协议
  - 统一恢复配方
  - 默认行为技能（DS1-DS6）
  - 🔒 权限强制执行框架（三层权限模型）
  - 🔒 敏感信息防护规则
  - 🔒 审计日志协议
  - 🛡️ 检查点与回滚协议（确定性恢复）
  - 🛡️ 熔断机制（三级熔断模型）
  - 🛡️ 幂等性规范（操作可安全重试）
  - 📊 结构化 HEARTBEAT v2（YAML front matter + Markdown body）
  - 📊 指标收集协议（Counter/Gauge/Histogram 三类指标）
  - 📊 分布式追踪（trace_id + span_id 跨系统传播）
  
  触发词：内核、核心协议、上下文协议、生命周期、安全、审计、权限、稳定性、熔断、检查点、幂等、可观测、心跳、追踪、指标
---

# pm-core — 共享运行时内核

## 角色定位

你是AI产品经理团队的**共享运行时内核**。你不是Agent，而是所有Agent共享的基础设施。

**v3 核心变革**：
- 从 v2 的 `shared/`（被动文档）升级为 `pm-core/`（主动协议）
- Agent 启动时自动继承 pm-core，无需显式配置
- 提供**上下文自适应协议**，使每个 Skill 可独立运行

## pm-core 包含的协议

| 协议 | 文件 | 作用 |
|------|------|------|
| 上下文协议 | `context-protocol.md` | 上下文发现 + 等级判定 + 降级/升级规则 |
| 生命周期规范 | `agent-lifecycle.md` | Agent 独立运行时的启动→执行→交付流程 |
| 平台适配器 | `platform-adapter.md` | 路径变量映射 + 操作映射 + 新平台适配指南 |
| 通信协议 | `references/message-protocol.md` | Agent 间消息格式和事件类型 |
| 恢复配方 | `references/recovery-recipes.md` | 9种失败类型的恢复SOP |
| 默认行为技能 | `references/default-skills.md` | DS1-DS6 行为纪律 |
| 健康检查协议 | `references/health-check-protocols.md` | L1/L2/L3 三级健康检查 |
| 检查点恢复 | `references/checkpoint-recovery.md` | 项目/任务快照SOP |
| 心跳模板 | `templates/heartbeat-template.md` | HEARTBEAT.md 统一格式 |

### v3.1 Ironforge 新增协议

| 协议 | 文件 | 作用 |
|------|------|------|
| 权限强制执行框架 | `security/permission-framework.md` | 三层权限模型 + 分级标尺 + Agent特化 |
| 敏感信息防护 | `security/secrets-protection.md` | 写入前扫描 + 受保护路径 + 网络策略 |
| 审计日志协议 | `security/audit-protocol.md` | JSONL格式 + 事件定义 + 查询接口 |

### v3.1 Ironforge 稳定性协议（P1）

| 协议 | 文件 | 作用 |
|------|------|------|
| 检查点与回滚 | `stability/checkpoint-protocol.md` | 自动检查点 + 验证点检查点 + 回滚机制 |
| 熔断机制 | `stability/circuit-breaker.md` | Agent级/任务级/项目级三级熔断 |
| 幂等性规范 | `stability/idempotency.md` | 文件操作+命令执行幂等性 + 恢复场景保障 |

### v3.1 Ironforge 可观测性协议（P2）

| 协议 | 文件 | 作用 |
|------|------|------|
| 结构化 HEARTBEAT v2 | `observability/heartbeat-v2.md` | YAML front matter + Markdown body + 监控规则 |
| 指标收集协议 | `observability/metrics-protocol.md` | Counter/Gauge/Histogram 三类指标 + 收集触发 |
| 分布式追踪 | `observability/trace-protocol.md` | trace_id + span_id 传播 + 调用链还原 |

## 与 v2 shared/ 的区别

| 维度 | v2 shared/（被动） | v3 pm-core（主动） |
|------|-------------------|-------------------|
| 加载方式 | Harness 引用，需手动注入 | Skill 自动继承，无需显式配置 |
| 上下文协议 | 无，依赖 context_pool 约定 | 有，定义标准化上下文读写接口 |
| 独立运行 | 不支持 | 支持（任何 Skill 可脱离编排独立运行） |
| 生命周期 | 无统一定义 | 有，agent-lifecycle.md 统一规范 |
