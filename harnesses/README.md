# Harness 定义目录（v3.1 Ironforge 架构版）

> Harness 是子Agent的**执行载体**，定义运行环境、绑定Skills、通信配置。
> v3.1 Ironforge 核心变革：公共逻辑抽取到 base/ 层，各Agent Harness 只保留特化逻辑。
>
> **v3.1 新增（P0 安全基线 + P1 稳定性）**：
> - `base/` 公共层：权限、安全、审计、检查点、交接、上下文、可观测性
> - 每个 Harness 通过 `inherits` 声明引用公共层
> - 三层权限执行模型（平台物理层 > Harness逻辑层 > SKILL.md行为层）
> - 敏感信息写入前扫描 + 受保护路径 + 审计日志
> - 检查点与回滚（确定性恢复 + 验证点检查点 + 交接检查点）
> - 三级熔断模型（Agent级/任务级/项目级）
> - 幂等性规范（文件操作 + 命令执行 + 恢复场景保障）
>
> **v3 保留**：
> - 每个 Skill 有 `standalone-prompt.md`（独立运行时的精简prompt）
> - 编排模板定义在 `orchestrations/` 目录
> - 上下文自适应协议在 `pm-core/context-protocol.md`

## 文件说明

### 公共层（v3.1 新增）

| 文件 | 说明 |
|------|------|
| `base/README.md` | 分层架构说明 |
| `base/permission-framework.md` | 权限强制执行（三层模型 + spawn配置模板） |
| `base/security-hooks.md` | 安全钩子（敏感信息扫描 + 受保护路径 + 网络策略） |
| `base/audit-logging.md` | 审计日志（JSONL格式 + 事件定义 + 写入规范） |
| `base/checkpoint-protocol.md` | 检查点（自动保存 + 验证点 + 交接检查点 + 回滚规范 + 熔断联动） |
| `base/handoff-protocol.md` | 交接棒（触发条件 + 流程 + 文档模板） |
| `base/context-engineering.md` | 上下文工程（分层 + 预算 + 噪声过滤） |
| `base/observability-config.md` | 可观测性（HEARTBEAT v2 + 追踪传播 + 指标收集 + 通信事件 + 监控规则） |

### 特化层（各Agent专属）

| 文件 | 对应Agent | 说明 |
|------|----------|------|
| `orchestrator-harness.md` | pm-orchestrator | 瘦身后主控器：监听+同步+收集+决策 |
| `backlog-manager-harness.md` | pm-backlog-manager | 需求池管理专家 |
| `analyst-harness.md` | pm-analyst | 需求澄清专家 |
| `planner-harness.md` | pm-planner | 任务规划专家 |
| `designer-harness.md` | pm-designer | 原型设计专家（并行可选） |
| `runner-harness.md` | pm-runner | 执行调度专家 |
| `coder-harness.md` | pm-coder | 编码执行子Agent |
| `researcher-harness.md` | pm-researcher | 信息检索子Agent |
| `writer-harness.md` | pm-writer | 内容输出子Agent |

## v3.1 分层架构

```
harnesses/
├── base/                  # 公共层（7个文件，改一处生效全局）
│   ├── permission-framework.md
│   ├── security-hooks.md
│   ├── audit-logging.md
│   ├── checkpoint-protocol.md
│   ├── handoff-protocol.md
│   ├── context-engineering.md
│   └── observability-config.md
│
├── coder-harness.md       # 特化层（只写Agent独有逻辑）
├── runner-harness.md
└── ...                    # 其他7个Agent
```

**改造前后对比**：
- 改造前：coder-harness 1055行，总代码量 ~3000行，大量重复
- 改造后：base/ ~1400行 + 各特化层 ~700行 = ~2100行，零重复
- 维护成本：改安全策略只改 `base/security-hooks.md`，9个Harness自动生效

## Ironforge 三层权限执行模型

```
┌────────────────────────────────────────────────────────┐
│ Layer 1: 平台物理层 — Agent的工具列表里根本没有，想做也做不到  │
│   WorkBuddy: explore→plan模式 / blocked_tools移除       │
├────────────────────────────────────────────────────────┤
│ Layer 2: Harness 逻辑层 — 允许执行但有"跟踪摄像头"         │
│   黄灯操作：通知后执行，orchestrator可拦截                 │
├────────────────────────────────────────────────────────┤
│ Layer 3: SKILL.md 行为层 — "道德准则"，靠自觉，作为兜底    │
└────────────────────────────────────────────────────────┘

安全强度：Layer 1 > Layer 2 > Layer 3
叠加规则：越危险的操作越靠底层拦截
```

## 方向盘差异化配置

| Agent | 方向盘模式 | 核心权限 | 核心约束 |
|-------|----------|---------|---------|
| **pm-orchestrator** | default | 只读+spawn+消息 | 不编码不调度；三角验证+打回路由 |
| **pm-backlog-manager** | default | 只读+创建需求文档 | 不产出代码；MVP需用户确认 |
| **pm-analyst** | default | 只读+创建需求文档 | 不产出代码；至少3轮追问 |
| **pm-planner** | default | 只读+创建规划+Skills检查 | DAG无循环；模块≤5功能点 |
| **pm-designer** | acceptEdits | 读写原型+image_gen | 覆盖全部核心功能；语义化命名 |
| **pm-runner** | acceptEdits | 创建子Agent+消息+Skills | 不自行决定打回；对齐原型 |
| **pm-coder** | acceptEdits | 读写代码+测试+调试 | 规划优先；delete_file被物理移除 |
| **pm-researcher** | acceptEdits | 只读+搜索+创建报告 | 调研是终端操作，不可委托 |
| **pm-writer** | acceptEdits | 只读+创建文档 | 文档与上游一致；不执行命令 |

## v3.1 P1 稳定性配置总览

| Agent | 检查点频率 | 熔断阈值 | 幂等重点 |
|-------|----------|---------|---------|
| coder | every_step | 3次/30min | replace_in_file唯一性 + 命令pre_check |
| runner | every_step | 2次/20min | agent_spawn去重 + 任务派发去重 |
| analyst | verification_only | 3次/30min | write_to_file检查已存在 |
| planner | verification_only | 2次/20min | write_to_file检查已存在 |
| designer | every_step | 3次/30min | write_to_file + image_gen去重 |
| researcher | verification_only | 3次/45min | write_to_file检查已存在 |
| writer | every_step | 3次/30min | write_to_file检查已存在 |
| orchestrator | every_step | 1次/15min | 状态转换幂等 + 阶段切换去重 |
| backlog-manager | verification_only | 3次/30min | write_to_file检查已存在 |
