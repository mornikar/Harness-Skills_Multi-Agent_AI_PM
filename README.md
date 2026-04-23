# AI产品经理多Agent Skill体系

> v3.1 Ironforge — 三层架构（内核+能力+编排）+ 企业工程化（安全/稳定/可观测）+ Skill独立运行 + 按需组合 + 平台无关

## v3 → v3.1 核心变革

| 维度 | v2（旧） | v3 | v3.1 Ironforge（新） |
|------|---------|-----|---------------------|
| 架构模式 | 流程驱动 | 能力驱动 | 能力驱动 + 企业级工程化 |
| 独立运行 | 不支持 | 每个Skill可单独运行 | 同v3 |
| 共享层 | shared/（被动文档） | pm-core/（主动协议+平台抽象） | pm-core/ + security/ + stability/ + observability/ |
| 安全机制 | 无 | 无 | 三层权限强制执行 + 敏感信息防护 + 审计日志 |
| 故障恢复 | 无 | 文档级恢复配方 | 检查点确定性恢复 + 三级熔断 + 幂等性 |
| 可观测性 | Markdown日志 | Markdown HEARTBEAT | HEARTBEAT v2（YAML+Markdown）+ 指标 + 追踪 |
| Harness | 独立1000+行 | 独立文件 | base/公共层 + 特化层（改一处生效全局） |
| 编排方式 | 固定6阶段流水线 | 预定义编排 + 自定义 | 同v3 |
| 上下文依赖 | 必须有前置流程产出 | 三级自适应 | 同v3 + 上下文工程 |
| 平台绑定 | 绑定特定API | 行为规范层平台无关 | 同v3 |

## 三层架构

```
┌─────────────────────────────────────────────────────────┐
│                   Layer 0: pm-core（内核层）              │
│   上下文协议 + 生命周期规范 + 通信协议 + 恢复配方         │
│   平台抽象层（platform-adapter.md）                      │
│   🔒 安全：权限框架 + 敏感信息防护 + 审计日志            │
│   🛡️ 稳定：检查点回滚 + 三级熔断 + 幂等性规范            │
│   📊 可观测：HEARTBEAT v2 + 指标收集 + 分布式追踪        │
│   所有 Skill 自动继承，无需显式配置                       │
├─────────────────────────────────────────────────────────┤
│               Layer 1: Skills（能力层）                   │
│   10个独立 Skill，每个可单独运行或被委托调用              │
│   每个Skill有三模式工作流：MINIMAL/PARTIAL/FULL          │
│   pm-coder | pm-researcher | pm-writer | pm-analyst      │
│   pm-planner | pm-designer | pm-backlog-manager          │
│   pm-runner | pm-orchestrator | web-design-engineer      │
├─────────────────────────────────────────────────────────┤
│              Layer 2: Orchestrations（编排层）             │
│   预定义编排模板，按需组合 Skills                         │
│   code-only | research-only | analysis-only | doc-only   │
│   full-pipeline | custom（用户自定义）                    │
└─────────────────────────────────────────────────────────┘
```

## v3.1 Ironforge 三大支柱

### 🔒 支柱A：安全加固（Fortify）

从"告诉Agent不要做"升级为"让Agent做不到"：

| 安全层 | 机制 | 效果 |
|--------|------|------|
| 平台物理层 | spawn mode=plan/acceptEdits + blocked_tools | 工具列表里没有危险操作，想做也做不到 |
| Harness逻辑层 | 黄灯操作执行前通知 + orchestrator拦截窗口 | 允许执行但有"跟踪摄像头" |
| SKILL.md行为层 | 编码规范、禁止事项 | "道德准则"兜底 |
| 敏感信息防护 | 写入前正则扫描 + 受保护路径 | API Key/密码/私钥不会泄露到代码里 |
| 审计日志 | JSONL追加写入 + 事件定义 + trace_id查询 | 每个操作可追溯 |

### 🛡️ 支柱B：稳定性加固（Stabilize）

从"文档级恢复配方"升级为"确定性检查点恢复"：

| 机制 | 作用 | 关键设计 |
|------|------|---------|
| 检查点协议 | 每步自动保存，崩溃后精确恢复 | 3类检查点（auto/verification/handoff）+ hash校验 |
| 三级熔断 | 快速隔离故障Agent | Agent级状态机 + 任务级退避 + 项目级紧急暂停 |
| 幂等性规范 | 重复操作不破坏 | replace_in_file 0匹配=跳过 + 命令pre_check |

**每个Agent的稳定性配置不同**（orchestrator最严格：1次/15min熔断；只读Agent检查点少）。

### 📊 支柱C：可观测性加固（Observe）

从"Markdown给人看"升级为"机器可解析+人可阅读"：

| 机制 | 作用 | 关键设计 |
|------|------|---------|
| HEARTBEAT v2 | 双格式状态报告 | YAML front matter（机器监控）+ Markdown body（人类理解） |
| 指标收集 | 量化系统运行状态 | Counter/Gauge/Histogram 三类指标 |
| 分布式追踪 | 跨Agent操作关联 | trace_id贯穿审计/指标/检查点/HEARTBEAT |
| 自动化监控 | 5条规则自动告警 | 任务停滞/健康度下降/轮次超支/错误聚集/阻塞超时 |

## 使用方式

### 方式1：单Skill直接使用（最轻量）
```
用户："帮我改个Bug"  → 加载 pm-coder → 独立运行 → 交付
用户："调研下XX技术" → 加载 pm-researcher → 独立运行 → 交付
```

### 方式2：预定义编排（中等复杂度）
```
用户："梳理需求+拆解任务" → analysis-only编排 → pm-analyst → pm-planner
用户："写个PRD"          → doc-only编排 → pm-writer
```

### 方式3：全链路编排（完整项目）
```
用户："从0做个App" → full-pipeline编排 → Phase 0→1→2→3→4→5
```

### 方式4：自定义编排（最灵活）
```yaml
# orchestrations/custom/my-workflow.yaml
name: my-workflow
skills_required: [pm-analyst, pm-coder]
flow:
  phase_1: {agent: pm-analyst, input: user_requirements}
  phase_2: {agent: pm-coder, input: [goal.md, user_instruction]}
```

## 架构概览（全链路模式）

```
┌─────────────────────────────────────────────────────────────────┐
│                      orchestrator 职责分层                        │
│                                                                  │
│  【核心三件事】上下文同步 · 结果收集 · 监听                        │
│  【全局职责】阶段切换决策 · 打回路由 · 用户交互 · 复盘沉淀          │
│  【不做的事】需求澄清、任务拆解、Skills安装、Agent调度、编码、文档   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────┼────────────────────────────┐
   ▼                            ▼                            ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  Phase 0     │        │  第一阶段     │        │  第二阶段     │
│  需求池管理   │──────→ │  需求→分析→拆解│──────→ │  原型+UI定型  │
│  +MVP定义    │        │              │        │  （并行可选）  │
│ pm-backlog   │        │ pm-analyst   │        │ pm-designer  │
│ -manager     │        │ → pm-planner │        │ （方向指标）  │
└──────────────┘        └──────┬───────┘        └──────┬───────┘
                               │                       │
                               │    ┌──────────────────┘
                               ▼    ▼
                        ┌──────────────┐        ┌──────────────┐
                        │  第三阶段     │        │  第四阶段     │
                        │ pm-runner    │──────→ │  打回循环     │
                        │ → subAgents  │        │ orchestrator │
                        └──────┬───────┘        └──────┬───────┘
                               │                       │
                               ▼                       ▼
                        ┌──────────────┐        ┌──────────────┐
                        │  第五阶段     │        │  使用者       │
                        │  整合交付     │──────→ │  人工验收     │
                        └──────────────┘        └──────────────┘
```

## 核心特性

### 1. 三层解耦架构

| 层 | 名称 | 职责 | 关键文件 |
|---|------|------|---------|
| Layer 0 | pm-core | 上下文协议 + 生命周期 + 通信 + 恢复 + 平台抽象 + 🔒安全 + 🛡️稳定 + 📊可观测 | 见目录结构 |
| Layer 1 | Skills | 10个独立能力单元，可单独运行或被委托调用 | 每个Skill的 SKILL.md + standalone-prompt.md |
| Layer 2 | Orchestrations | 预定义编排模板 + 自定义编排 | orchestrations/*.yaml |

### 2. 上下文自适应（v3 新增）

每个Skill启动时自动扫描上下文，三级自适应：

| 等级 | 条件 | 行为 |
|------|------|------|
| FULL | 所有前置文档齐全 | 严格对齐，按流程验收 |
| PARTIAL | 部分文档存在 | 对齐已有，缺失从用户推导 |
| MINIMAL | 无前置文档 | 直接接收用户指令，轻量运行 |

### 3. 平台抽象层（v3 新增）

行为规范层（SKILL.md、pm-core/、orchestrations/）不绑定任何特定AI平台：

| 路径变量 | 说明 | 映射示例 |
|---------|------|---------|
| `{context_root}` | 项目上下文根目录 | `.workbuddy/`、`.cursor/`、`.cline/` |
| `{skills_root}` | Skills 安装目录 | `~/.workbuddy/skills/`、项目级 skills/ |

> 详见 `pm-core/platform-adapter.md`

### 4. 六阶段流程（全链路模式）

| 阶段 | 名称 | 主导 Agent | 关键产出 |
|------|------|-----------|---------|
| P0 | 需求池管理+MVP定义 | pm-backlog-manager | backlog.md + mvp-definition.md + batch-plan.md |
| P1 | 需求→分析→拆解 | pm-analyst → pm-planner | goal.md + modules.md + dependency-dag.md |
| P2 | 原型+UI定型（并行） | pm-designer + 使用者 | 可交互原型 + 组件树 + 页面路由 |
| P3 | 架构→开发→测试 | pm-runner → subAgents | 可运行代码 + 模块测试 + 文档 |
| P4 | 打回循环 | orchestrator（决策） | 最小回退到必要阶段 |
| P5 | 整合交付 | orchestrator | 模块化验收 + 复盘报告 |

### 5. 第二阶段并行

```
【旧流程】Phase 0 → Phase 1 → Phase 2（阻塞）→ Phase 3

【新流程】Phase 0 → Phase 1 → Phase 3（同时 Phase 2 并行进行）
                              ↓
                    Phase 2 原型完成后 → 成为 Phase 3 研发的方向指标
                    使用者在 Phase 3 执行的同时并行完善原型UI
```

**关键约束**：
- Phase 3 不等 Phase 2 完成，Phase 1 结束后立即开始
- Phase 2 原型完成后，成为 Phase 3 的方向指标
- Phase 3 初期开发以需求文档为基准，原型定型后切换为对齐原型

### 6. 三段式解耦

orchestrator 不再大包大揽，职责分发到专业 Agent：

| Agent | 独立运行 | 最低上下文 | 独立模式 | 编排模式 |
|-------|:--------:|:---------:|---------|---------|
| pm-backlog-manager | ✅ | MINIMAL | 需求排序+MVP定义 | Phase 0 守门人 |
| pm-analyst | ✅ | MINIMAL | 直接澄清需求 | Phase 1 需求分析 |
| pm-planner | ✅ | PARTIAL | 接收需求→任务拆解 | Phase 1 拆解 |
| pm-designer | ✅ | PARTIAL | 接收需求→原型 | Phase 2 方向指标 |
| pm-runner | ✅ | PARTIAL | 接收任务→调度执行 | Phase 3 调度 |
| pm-coder | ✅ | MINIMAL | 直接改代码 | Phase 3 编码 |
| pm-researcher | ✅ | MINIMAL | 直接调研 | Phase 3 调研 |
| pm-writer | ✅ | MINIMAL | 直接写文档 | Phase 3 文档 |
| pm-orchestrator | ✅ | MINIMAL | 全链路编排 | 本身就是编排者 |

### 7. 优先级排序策略（主流程优先）

```
P0 主流程核心功能 → 第一批交付给研发Agent
   - 用户核心路径上的功能
   - 阻塞其他功能开发的关键模块
   - 技术依赖的前置模块

P1 重要但可延后 → 第二批交付
   - 用户体验优化功能
   - 辅助性功能

P2 需要暗地发觉讨论 → 后续批次
   - 边缘场景
   - 需要深入讨论的需求
```

### 8. 原型作为方向指标

原型不再是开发的阻塞前提，而是研发的**方向指标**。开发先以需求文档为基准启动，原型完成后切换为对齐原型开发。

### 9. 模块化验收 + 打回循环

- 验收以模块为单位（不是任务级），更有实际意义
- 跨阶段打回由 orchestrator 决策路由，规则是"最小必要回退点"
- pm-runner 只能上报，不能自行决定打回

### 10. 自动化验收标准（Agent可执行）

```yaml
module_test_criteria:
  代码质量:
    - 单元测试覆盖率 ≥ 80%
    - 代码静态分析无高危问题
    - 依赖版本无已知漏洞
  功能验证:
    - 核心功能路径自动化测试通过
    - 边界条件测试覆盖
    - 异常处理测试通过
  集成验证:
    - 接口契约测试通过
    - 模块间依赖调用正常
```

### 11. Harness = 高解耦方向盘（v3.1 Ironforge 升级）

每个 Agent 的 Harness 采用 **inherits 模式**：继承 base/ 公共层 + 只写特化逻辑。

公共层（7个文件）定义所有Agent共享的权限/安全/审计/检查点/交接/上下文/可观测性配置。

每个 Harness 只保留特化逻辑：权限覆盖、审计事件覆盖、稳定性覆盖（检查点频率/熔断阈值/幂等规则）、上下文预算覆盖。

> 详见 `harnesses/README.md`

### 12. 上下文池（Context Pool）

全局共享的项目上下文，细粒度访问权限控制：

| 文件 | orchestrator | backlog-mgr | analyst | planner | designer | runner | subAgents |
|------|-------------|-------------|---------|---------|----------|--------|-----------|
| backlog.md | 读写 | 读写 | 只读 | 只读 | — | — | — |
| mvp-definition.md | 读写 | 读写 | 只读 | 只读 | — | — | — |
| batch-plan.md | 读写 | 读写 | 只读 | 只读 | — | — | — |
| goal.md | 读写 | — | 读写 | 只读 | 只读 | 只读 | — |
| product.md | 读写 | — | 只读 | 只读 | 只读 | — | — |
| modules.md | 读写 | — | — | 读写 | 只读 | 只读 | — |
| dependency-dag.md | 读写 | — | — | 读写 | — | 读写 | — |
| prototype/ | 读写 | — | — | — | 读写 | 只读 | 只读 |
| architecture.md | 读写 | — | — | — | — | 读写 | 只读 |
| progress/*.md | 读写 | — | — | — | — | 读写 | 读写 |

### 13. HEARTBEAT v2 双层记忆（v3.1 升级）

- **项目级**：全局进度、关键决策、风险追踪
- **任务级**：单任务状态、上下文、中间结果
- **v3.1 新增**：YAML front matter（机器可解析）+ Markdown body（人类可读）
  - 自动化监控：5条规则（任务停滞/健康度下降/轮次超支/错误聚集/阻塞超时）
  - 健康度量化：100 - errors×5 - retries×3 - turn_overuse×10 - stale×15
  - trace_id 贯穿：跨审计/指标/检查点/HEARTBEAT 统一追踪

## 目录结构

```
AI_PM_SKills/
├── README.md                    # 本文件
├── ARCHITECTURE.md              # 架构详细说明
├── ARCHITECTURE_SPEC.md         # 架构说明书
├── ENTERPRISE_FRAMEWORK.md      # Ironforge企业工程化框架文档
├── QUICKSTART.md               # 快速开始指南
│
├── pm-core/                    # 🧠 共享内核（Layer 0）
│   ├── context-protocol.md     # 上下文自适应协议
│   ├── agent-lifecycle.md      # Agent 生命周期规范
│   ├── platform-adapter.md     # 平台抽象层（v3）
│   ├── SKILL.md                # 内核入口
│   ├── security/               # 🔒 安全协议（v3.1 Ironforge）
│   │   ├── permission-framework.md   # 权限强制执行框架
│   │   ├── secrets-protection.md     # 敏感信息防护规则
│   │   └── audit-protocol.md         # 审计日志协议
│   ├── stability/              # 🛡️ 稳定性协议（v3.1 Ironforge）
│   │   ├── checkpoint-protocol.md    # 检查点与回滚
│   │   ├── circuit-breaker.md        # 三级熔断机制
│   │   └── idempotency.md            # 幂等性规范
│   ├── observability/          # 📊 可观测性协议（v3.1 Ironforge）
│   │   ├── heartbeat-v2.md           # 结构化HEARTBEAT v2
│   │   ├── metrics-protocol.md       # 指标收集协议
│   │   └── trace-protocol.md         # 分布式追踪协议
│   ├── references/             # 参考资料
│   │   ├── recovery-recipes.md
│   │   ├── default-skills.md
│   │   ├── health-check-protocols.md
│   │   ├── checkpoint-recovery.md
│   │   ├── message-protocol.md
│   │   └── lessons-learned.md
│   └── templates/              # 模板
│       └── heartbeat-template.md   # HEARTBEAT v2模板（含YAML front matter）
│
├── pm-orchestrator/            # 主控器（瘦身后）
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-backlog-manager/         # 需求池管理 Agent
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-analyst/                 # 需求澄清 Agent
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-planner/                 # 颗粒化拆解 Agent
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-designer/                # 原型设计 Agent（并行可选）
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-runner/                  # 调度执行 Agent
│   ├── SKILL.md
│   └── standalone-prompt.md
│
├── pm-coder/                   # 编程执行 Agent
│   ├── SKILL.md
│   ├── standalone-prompt.md
│   └── references/
│
├── pm-researcher/              # 信息检索 Agent
│   ├── SKILL.md
│   ├── standalone-prompt.md
│   └── references/
│
├── pm-writer/                  # 内容输出 Agent
│   ├── SKILL.md
│   ├── standalone-prompt.md
│   └── references/
│
├── web-design-engineer/        # 🎨 视觉设计工程 Skill（v3.1 新增，独立挂载）
│   ├── SKILL.md
│   ├── standalone-prompt.md
│   └── references/
│       └── advanced-patterns.md
│
├── orchestrations/             # 🔄 编排模板（Layer 2）
│   ├── README.md
│   ├── code-only.yaml
│   ├── research-only.yaml
│   ├── analysis-only.yaml
│   ├── doc-only.yaml
│   ├── full-pipeline.yaml
│   └── custom/
│       └── README.md
│
├── harnesses/                  # ⚙️ Harness 配置（平台适配器）
│   ├── README.md
│   ├── base/                   # 🆕 公共层（v3.1 Ironforge）
│   │   ├── permission-framework.md
│   │   ├── security-hooks.md
│   │   ├── audit-logging.md
│   │   ├── checkpoint-protocol.md
│   │   ├── handoff-protocol.md
│   │   ├── context-engineering.md
│   │   └── observability-config.md
│   ├── orchestrator-harness.md
│   ├── backlog-manager-harness.md
│   ├── analyst-harness.md
│   ├── planner-harness.md
│   ├── designer-harness.md
│   ├── runner-harness.md
│   ├── coder-harness.md
│   ├── researcher-harness.md
│   ├── writer-harness.md
│   └── web-design-engineer-harness.md  # 🎨 视觉设计工程 Skill（v3.1 新增）
│
└── scripts/                    # 工具脚本
```

## Agent 角色体系

| Agent | 核心职责 | 触发词 | 阶段 |
|-------|---------|--------|------|
| pm-orchestrator | 上下文同步·结果收集·阶段切换·打回路由 | 开发、实现、做一个、搭建 | 全局 |
| pm-backlog-manager | 需求池管理·优先级排序·MVP定义 | 需求池、优先级、MVP | P0 |
| pm-analyst | 需求澄清·约束提取·Goal构建 | 需求澄清、范围确认、Goal定义 | P1 |
| pm-planner | 颗粒化拆解·依赖DAG·Skills分析 | 任务拆解、模块化、规划 | P1 |
| pm-designer | 原型设计·组件树·交互流程 | 原型设计、UI设计、线框图 | P2（并行） |
| pm-runner | 调度执行·策略引擎·健康监控 | 调度、执行、运行 | P3 |
| pm-coder | 编码·调试·重构·测试 | 编码、实现、开发、调试 | P3 |
| pm-researcher | 技术调研·竞品分析·方案选型 | 调研、对比、选型、分析 | P3 |
| pm-writer | PRD·技术文档·API文档·CHANGELOG | 文档、PRD、撰写、编写 | P3 |

## 安装使用

### 1. 安装到 Skills 目录

根据你的AI平台，将 Skills 复制到对应的目录：

```powershell
# 示例：复制到用户级 Skills 目录
Copy-Item -Path "AI_PM_SKills\pm-*" `
  -Destination "{skills_root}\" -Recurse -Force
```

> 具体路径映射见 `pm-core/platform-adapter.md`

### 2. 开始使用

直接对 AI 说出你的需求：

- "帮我做一个记账App"
- "开发一个Markdown编辑器"
- "实现用户认证功能"
- "帮我改个Bug"              → pm-coder 独立运行
- "调研下XX技术"             → pm-researcher 独立运行
- "排一下需求优先级"          → pm-backlog-manager 独立运行

pm-orchestrator 会自动触发六阶段流程（全链路模式），或单个 Skill 独立运行。

## 上下文池文件结构

```
{context_root}/context_pool/
├── backlog.md               # 需求池（backlog-manager 写，其余只读）
├── mvp-definition.md        # MVP定义（backlog-manager 写，其余只读）
├── batch-plan.md            # 批次交付计划（backlog-manager 写，其余只读）
├── product.md              # 产品定义（analyst 写，其余只读）
├── goal.md                 # Goal 对象（analyst 写，其余只读）
├── requirements.md         # 需求清单（analyst 写，其余只读）
├── modules.md              # 模块清单（planner 写，其余只读）
├── dependency-dag.md       # 依赖DAG（planner 写，runner 读）
├── tasks.md                # 任务列表（planner 写，runner 读）
├── skills-needed.md        # Skills需求（planner 写，runner 读）
├── architecture.md         # 架构设计（runner 写，subAgents 读）
├── prototype/              # 原型文件（designer 写，runner/coder 读）
│   ├── wireframe.html
│   ├── component-tree.md
│   └── routes.md
├── decisions.md            # 决策记录（orchestrator 读写）
├── progress/               # 任务进度（runner/subAgents 读写）
│   ├── T001.md
│   └── ...
└── shared/                 # 共享数据
    ├── api-schema.json
    ├── db-schema.sql
    └── ui-mockups/
```

## HEARTBEAT.md 格式

```markdown
# Project Heartbeat

## 项目信息
- **项目名称**: MyApp
- **创建时间**: 2026-04-23
- **当前阶段**: P3（架构→开发→测试）
- **最后更新**: 2026-04-23 14:30

## 阶段状态

| 阶段 | 状态 | 主导Agent | 完成时间 |
|------|------|----------|---------|
| P0 | ✅ 完成 | pm-backlog-manager | 09:00 |
| P1 | ✅ 完成 | pm-analyst → pm-planner | 10:00 |
| P2 | 🔄 进行中 | pm-designer | — |
| P3 | 🔄 进行中 | pm-runner | — |
| P4 | ⏳ 待定 | orchestrator | — |
| P5 | ⏳ 待定 | orchestrator | — |

## 方向指标状态
- 原型状态: 🔄 进行中
- 开发基准: 需求文档（原型未完成时）
- 原型完成后: 切换为对齐原型开发

## 任务状态看板

| 任务ID | 模块 | Agent | 状态 | 进度 | 阻塞项 | 健康度 |
|-------|------|-------|------|------|--------|--------|
| T001 | 存储 | pm-coder | completed | 100% | — | 95 |
| T002 | 编辑器 | pm-coder | running | 60% | — | 80 |
| T003 | 搜索 | pm-coder | pending | 0% | 依赖T001 | — |

## 关键决策
- [2026-04-23] MVP范围确认：核心编辑+搜索+存储
- [2026-04-23] 选择Vue3 + Electron作为技术栈

## 风险与问题
- [MEDIUM] Electron打包体积较大 → 考虑使用Tauri替代

## 经验沉淀
- [L1] IndexedDB 异步操作需包装为 Promise
```

## 扩展开发

### 添加新的子Agent

1. 创建 `pm-{agent-name}/SKILL.md`（平台无关的行为规范）
2. 创建 `pm-{agent-name}/standalone-prompt.md`（独立运行精简prompt）
3. 在 `harnesses/` 下创建对应 `{agent-name}-harness.md`（平台适配器）
4. 在 `pm-runner/SKILL.md` 中注册子Agent配置
5. 更新 `orchestrations/` 中相关编排模板
6. 更新本 README 的 Agent 角色体系表

### 添加共享资源

1. 放入 `pm-core/references/`
2. 在相关 Agent 的 SKILL.md 中添加引用说明

### 适配新的AI平台

1. 在 `pm-core/platform-adapter.md` 中添加新平台的路径映射
2. 在 `harnesses/` 下创建适配新平台的 Harness 文件
3. SKILL.md 和 pm-core/ 无需修改（已平台无关）

## 最佳实践

1. **主流程优先**：P0核心功能优先交付，不纠结边缘场景
2. **并行开发**：Phase 2 和 Phase 3 并行，不等原型完成再开发
3. **方向指标**：原型完成后成为开发对齐基准
4. **模块粒度**：单模块功能点 ≤5，含验收标准
5. **DAG 无环**：依赖关系必须形成有向无环图
6. **最小回退**：打回时退到最小必要回退点，不退回全部
7. **策略引擎**：pm-runner 策略自动处理阻塞/恢复，不打断流程
8. **打回上报**：pm-runner 不能自行决定打回，只能上报 orchestrator
9. **追问 ≥3 轮**：pm-analyst 必须至少追问 3 轮才可结束需求澄清
10. **自动化验收**：测试验收以Agent可执行的自动化标准为准
11. **平台无关**：行为规范层不绑定特定AI平台，通过 platform-adapter.md 适配
12. **安全三铁律**（v3.1）：约束可执行、操作可追溯、故障可恢复
13. **trace_id 贯穿**（v3.1）：从项目开始到结束，trace_id 统一追踪所有操作
14. **改一处生效全局**（v3.1）：公共逻辑放 base/，特化逻辑放各 Harness

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v3.1 | 2026-04-24 | Ironforge企业工程化框架：安全加固+稳定性加固+可观测性加固+base/公共层+10个Skill（新增web-design-engineer视觉设计工程）+Harness重构 |
| v3.0 | 2026-04-23 | v3高解耦架构：能力驱动+Skill独立运行+三层架构+上下文自适应+平台无关改造+pm-core取代shared/ |
| v2.1 | 2026-04-23 | v2架构优化：Phase 0新增(pm-backlog-manager)、第二阶段并行、原型改为方向指标、自动化验收标准 |
| v2.0 | 2026-04-22 | 架构重构：五阶段流程+三段式解耦+4个新Agent+Harness方向盘 |
| v1.0 | 2026-04-20 | 初始版本，7阶段流程+4个核心Agent |

## 许可证

MIT License
