# AI PM Skills v3.1 架构说明书

> 版本: v3.1 Ironforge  
> 日期: 2026-04-24  
> 状态: 已实施  
> 作者: AI PM Team

---

## 1. 架构总览

### 1.1 设计哲学

AI PM Skills v3 的核心设计哲学是 **"高解耦 + 可插拔 + 上下文自适应 + 平台无关"**：

1. **高解耦**：从流程驱动→能力驱动，每个 Skill 可独立运行，不依赖前置流程
2. **可插拔**：编排是可选的组合层，用户按需选择 code-only / full-pipeline / 自定义
3. **上下文自适应**：FULL/PARTIAL/MINIMAL 三级降级，有文档对齐，没有就轻量运行
4. **平台无关**：行为规范层（SKILL.md、pm-core/、orchestrations/）不绑定任何特定AI平台
5. **向后兼容**：v2 的六阶段全链路流程仍是可用编排（full-pipeline）之一
6. **企业级工程化**（v3.1 Ironforge）：安全强制执行 + 确定性恢复 + 自动化可观测

### 1.2 三层架构

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
├─────────────────────────────────────────────────────────┤
│              Layer 2: Orchestrations（编排层）             │
│   预定义编排模板 + 自定义编排                             │
└─────────────────────────────────────────────────────────┘
```

### 1.3 六阶段流程（全链路编排模式）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     pm-orchestrator（瘦身后：只做监听+同步+收集+决策）         │
│                                                                              │
│  【核心三件事】上下文同步 · 结果收集 · 监听                                    │
│  【全局职责】阶段切换决策 · 打回路由 · 用户交互 · 复盘沉淀                      │
│  【不做的事】需求澄清、任务拆解、Skills安装、Agent调度、编码、文档               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       │                            │                            │
 Phase 0 守门           Phase 1 团队              Phase 2 + Phase 3 并行
       │                            │                            │
       ▼                            ▼                            ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ pm-backlog-mgr  │      │  pm-analyst     │      │  pm-designer    │
│ 需求池管理       │─────→│  需求澄清        │      │  原型设计        │
│ 优先级排序       │      │  Goal构建       │      │  线框图/组件树   │
│ MVP定义         │      │      ↓          │      │  使用者并行完善  │
│ 批次交付计划     │      │  pm-planner     │      └────────┬────────┘
│                 │      │  颗粒化拆解      │               │
└─────────────────┘      │  依赖DAG        │               │ 原型完成后
                         │  Skills分析     │               │ 成为方向指标
                         └────────┬────────┘               │
                                  │                        │
                                  └────────────┬───────────┘
                                               │
                                               ▼
                                  ┌─────────────────┐
                                  │  pm-runner      │
                                  │  执行调度        │
                                  │  架构讨论        │
                                  │  Skills管理      │
                                  │  子Agent调度     │
                                  │  ↓  ↓  ↓        │
                                  │  coder×N       │
                                  │  researcher    │
                                  │  writer        │
                                  └─────────────────┘
                                               │
                    ┌──────────────────────────────────┐
                    │  Phase 4: 打回循环                 │
                    │  模块化验收 + 最小必要回退          │
                    │  打回决策权归 orchestrator          │
                    └──────────────────────────────────┘
                                    │
                    ┌──────────────────────────────────┐
                    │  Phase 5: 整合交付                 │
                    │  模块化讨论验收 + 复盘               │
                    │  协作奖励评估 + 经验沉淀             │
                    └──────────────────────────────────┘
                                    │
                    ┌──────────────────────────────────┐
                    │  使用者人工验收                     │
                    └──────────────────────────────────┘
```

### 1.4 核心变化：第二阶段并行

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

### 1.5 平台抽象层

v3 新增 `pm-core/platform-adapter.md`，定义平台无关的路径变量和操作映射：

| 路径变量 | 说明 | 示例映射 |
|---------|------|---------|
| `{context_root}` | 项目上下文根目录 | `.workbuddy/`、`.cursor/`、`.cline/` |
| `{skills_root}` | Skills 安装目录 | `~/.workbuddy/skills/`、项目级 skills/ |

> 所有 SKILL.md 和 pm-core 文件使用 `{context_root}`、`{skills_root}` 等抽象变量，不硬编码任何平台特定路径。

---

## 2. 八大 Agent 角色体系

### 2.1 Agent 一览

| Agent | 层级 | 核心职责 | 独立运行 | 最低上下文 |
|-------|------|---------|---------|-----------|
| pm-orchestrator | 主控 | 监听+同步+收集+决策+打回路由+复盘 | ✅ MINIMAL | 直接编排 |
| pm-backlog-manager | Phase 0 | 需求池管理+优先级排序+MVP定义 | ✅ MINIMAL | 需求排序+MVP |
| pm-analyst | Phase 1 | 需求澄清 + Goal构建 | ✅ MINIMAL | 直接澄清需求 |
| pm-planner | Phase 1 | 颗粒化拆解 + 依赖DAG + Skills分析 | ✅ PARTIAL | 接收需求→拆解 |
| pm-designer | Phase 2 | 原型设计 + 组件树 + 多方案 | ✅ PARTIAL | 接收需求→原型 |
| pm-runner | Phase 3 | 架构讨论 + Skills管理 + 子Agent调度 | ✅ PARTIAL | 接收任务→调度 |
| pm-coder | Phase 3 子Agent | 编码执行 + 测试 + 调试 | ✅ MINIMAL | 直接改代码 |
| pm-researcher | Phase 3 子Agent | 信息检索 + 竞品分析 | ✅ MINIMAL | 直接调研 |
| pm-writer | Phase 3 子Agent | 文档输出 + PRD | ✅ MINIMAL | 直接写文档 |

### 2.2 委托关系

```
orchestrator
    ├──→ pm-backlog-manager（Phase 0）
    ├──→ pm-analyst（Phase 1）
    ├──→ pm-planner（Phase 1）
    ├──→ pm-designer（Phase 2，与 Phase 3 并行）
    └──→ pm-runner（Phase 3）
              ├──→ pm-coder × N
              │       └──→ pm-researcher（允许：编码时需调研）
              ├──→ pm-researcher
              └──→ pm-writer
                      └──→ pm-researcher（允许：撰写时需补充信息）

禁止：pm-researcher 不再向下委托
禁止：嵌套委托（A→B→C 不允许）
```

### 2.3 Skill vs Harness

| 维度 | Skill | Harness |
|------|-------|---------|
| 本质 | 知识包（SKILL.md + references/） | 执行载体（运行环境配置） |
| 类比 | SOP 手册 | 带工具箱的工作台 |
| 定义内容 | 角色定位、工作流程、输出模板、禁止事项 | spawn配置、工具绑定、通信协议、Skill加载策略 |
| 文件位置 | `{skills_root}/pm-{name}/SKILL.md` | `harnesses/{name}-harness.md` |
| 生命周期 | 项目级（不随任务变化） | 任务级（每次创建子Agent重新配置） |
| 平台绑定 | **无**（v3 平台无关） | **有**（适配特定AI平台的执行环境） |

---

## 3. 核心创新点

### 3.1 Phase 0 需求池管理 + 优先级排序（v2.1 新增）

> 设计参考：产品经理的需求池管理最佳实践

在需求进入开发流程前，先经过需求池管理和优先级排序：

**优先级排序策略**：
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

**MVP定义**：从P0需求中识别MVP范围，定义核心功能边界和验收标准。

### 3.2 第二阶段并行优化（v3 核心变化）

> 设计参考：敏捷开发中的并行工作流

原型设计不再是开发的阻塞前提，而是与开发并行进行：

1. Phase 1 完成后，Phase 2 和 Phase 3 **同时开始**
2. Phase 3 初期以需求文档为基准开发
3. Phase 2 原型完成后，成为 Phase 3 的**方向指标**
4. 使用者在 Phase 3 执行的同时并行完善原型UI

**优势**：减少等待时间，开发与原型设计并行推进。

### 3.3 三模式工作流（v3 新增）

每个 Skill 都有三种运行模式，根据上下文自动适配：

| 模式 | 条件 | 行为 |
|------|------|------|
| **MINIMAL** | 无前置文档 | 直接接收用户指令，轻量运行 |
| **PARTIAL** | 部分文档存在 | 对齐已有，缺失从用户推导 |
| **FULL** | 所有前置文档齐全 | 严格对齐流程，按流程验收 |

### 3.4 平台无关改造（v3 新增）

**核心原则**：行为规范层（SKILL.md、pm-core/、orchestrations/）不绑定任何特定AI平台。

- SKILL.md 中使用 `{context_root}`、`{skills_root}` 等抽象路径变量
- 操作描述化：`read_file("path")` → `读取文件 {path}`
- 通信描述化：`send_message(...)` → `向...发送消息`
- 平台抽象层：`pm-core/platform-adapter.md` 定义路径变量映射和操作映射
- harnesses/ 保留平台绑定（它们是特定AI平台的适配器）

### 3.5 推理验证点（Reasoning Checkpoints）

> 设计参考：DeepSeek-R1 的"推理链检查点"思想

在关键决策节点，Agent 必须暂停推理、输出显式的验证步骤，然后才能继续。防止"想当然"导致的错误沿链路放大。

**验证点清单：**

| ID | 名称 | Phase | 触发时机 | 执行者 |
|----|------|-------|---------|--------|
| RC-P0-01 | MVP范围验证 | Phase 0 | backlog-manager 完成MVP定义后 | backlog-manager |
| RC-P1-01 | 需求理解验证 | Phase 1 | analyst 完成追问后 | analyst |
| RC-P1-01b | 拆解完整性验证 | Phase 1 | planner 完成拆解后 | planner |
| RC-P2-01 | 原型覆盖度验证 | Phase 2 | designer 完成原型后 | designer |
| RC-P2-02 | 原型用户确认 | Phase 2 | 覆盖度验证通过后 | orchestrator + 用户 |
| RC-P3-01 | 架构方案可行性验证 | Phase 3 | runner 收集多方案后 | runner |
| RC-P3-02 | 模块开发对齐验证 | Phase 3 | 每个模块开发完成后 | runner |
| RC-P3-02a | 代码-原型一致性 | Phase 3 | 模块开发完成时 | coder |
| RC-P4-01 | 打回根因验证 | Phase 4 | 打回循环触发前 | orchestrator |
| RC-P5-01 | 整合一致性验证 | Phase 5 | 模块化讨论验收时 | orchestrator |

### 3.6 自动化验收标准（v2.1 新增）

> 面向 Agent 的自动化验收，取代传统 DoD 清单

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
    - 数据流完整
```

### 3.7 协作奖励模型（Collaborative Reward Model）

> 设计参考：协作 RL 的"团队奖励"思想

评估 Agent 贡献时，不只看个体产出质量，还看其对团队协作的正外部性。

**奖励信号构成：**

| 类别 | 信号 | 权重 | 说明 |
|------|------|------|------|
| 个体质量 | 交付物质量 | 0.3 | 三角验证置信度 |
| 个体质量 | 约束合规 | 0.2 | 约束违规次数（反向） |
| 协作外部性 | 下游便利度 | 0.2 | 下游 Agent 是否无需额外澄清 |
| 协作外部性 | 信息同步及时性 | 0.1 | HEARTBEAT 更新及时率 |
| 协作外部性 | 可复用性 | 0.1 | 产出物是否可被其他模块/项目复用 |
| 协作外部性 | 冲突避免 | 0.1 | 是否主动避免与其他 Agent 的文件/资源冲突 |
| 惩罚 | 下游阻塞惩罚 | -0.3 | 因本 Agent 产出问题导致下游阻塞 |
| 惩罚 | 打回连锁惩罚 | -0.2 | 因本 Agent 产出问题导致跨阶段打回 |

### 3.8 树搜索架构讨论（Architecture Tree Search）

> 设计参考：论文中"树搜索"思想

在架构阶段探索多棵子树，基于评估剪枝，而非仅比较两个方案。

**流程：**
1. 识别架构决策点（状态管理、数据层、路由架构等）
2. 为每个决策点生成 2-3 个候选方案（创建多个 coder Agent）
3. 局部评估 + 剪枝（原型覆盖度 0.3 + 技术可行性 0.25 + 开发效率 0.2 + 可维护性 0.15 + 扩展性 0.1）
4. 全局组合优化（各决策点 Top 方案组合，检查一致性）
5. 上报 orchestrator 最终决策

### 3.9 多方案设计并行（Design Tree Search）

在原型阶段，当核心交互流程有 ≥2 种可行方案时，pm-designer 必须输出多方案对比：
- 方案A：保守方案（成熟模式）
- 方案B：创新方案（可能更好但有风险）
- 方案C：简化方案（快速实现）

### 3.10 Ironforge 安全加固（v3.1 新增）

> 从"告诉Agent不要做"到"让Agent做不到"

**三层权限执行模型**：

| 层级 | 机制 | 安全强度 | 示例 |
|------|------|---------|------|
| 平台物理层 | spawn mode + blocked_tools | 最高（不可绕过） | coder的delete_file直接移除 |
| Harness逻辑层 | 黄灯通知 + 拦截窗口 | 中（有监督） | replace_in_file前通知orchestrator |
| SKILL.md行为层 | 编码规范 + 禁止事项 | 低（靠自觉） | "不要硬编码密码" |

**敏感信息防护**：写入前正则扫描（API Key/密码/私钥）+ 受保护路径（.env/credentials/）。

**审计日志**：JSONL追加写入 + trace_id + span_id + 事件类型 + 风险等级。

> 详见：`pm-core/security/` 和 `ENTERPRISE_FRAMEWORK.md`

### 3.11 Ironforge 稳定性加固（v3.1 新增）

> 从"文档级恢复配方"到"确定性检查点恢复"

**检查点协议**：3类检查点（auto/verification/handoff）+ hash校验 + 5步回滚流程。

**三级熔断模型**：

| 级别 | 触发条件 | 动作 |
|------|---------|------|
| Agent级 | 同类Agent连续N次失败 | closed→open→half_open状态机 |
| 任务级 | 同一任务重试耗尽 | 指数退避 + 上报orchestrator |
| 项目级 | 轮次>80%且完成率<30% | 全局暂停 + 紧急通知用户 |

**幂等性规范**：replace_in_file 0匹配=跳过 + 命令pre_check + 文件操作安全策略。

**每个Agent的稳定性配置不同**：

| Agent | 熔断阈值 | 检查点 | 原因 |
|-------|---------|--------|------|
| orchestrator | 1次/15min | every_step | 失败影响全局 |
| runner | 2次/20min | every_step | 调度失败影响大 |
| 只读Agent | 3次/30min | verification_only | 无文件修改 |

> 详见：`pm-core/stability/` 和 `ENTERPRISE_FRAMEWORK.md`

### 3.12 Ironforge 可观测性加固（v3.1 新增）

> 从"Markdown给人看"到"机器可解析+人可阅读"

**HEARTBEAT v2**：YAML front matter（机器解析+自动化监控）+ Markdown body（人类阅读理解进度）。

**指标收集**：Counter（只增，如agent_spawn_total）+ Gauge（可增减，如health_score）+ Histogram（分布，如task_duration_turns）。

**分布式追踪**：trace_id（proj-YYYYMMDD-xxxx）贯穿审计/指标/检查点/HEARTBEAT，span_id精确定位Agent+任务。

**5条自动化监控规则**：任务停滞/健康度下降/轮次超支/错误聚集/阻塞超时。

> 详见：`pm-core/observability/` 和 `ENTERPRISE_FRAMEWORK.md`

---

## 4. 核心机制详解

### 4.1 三角验证模型

| 验证层 | 执行者 | 内容 | 特点 |
|--------|--------|------|------|
| 确定性规则 | 子Agent自验证 | 格式、编译、关键词、测试 | 零歧义、快速 |
| 语义评估 | orchestrator | 对比 Goal success_criteria 加权打分 | 灵活但有幻觉风险 |
| 人工判断 | 用户（HITL） | 原型确认、MVP交付、架构决策 | 最高权威 |

**验收算法：**
```
置信度 ≥ 80% → COMPLETED
置信度 < 80% → 打回循环（附反馈）
硬约束违规 → ESCALATE
```

### 4.2 打回路由

| 触发条件 | 最小回退目标 | 动作 |
|---------|------------|------|
| 原型不通过 | Phase 1 中不通过的功能需求 | 启动 analyst 局部修正 |
| 架构不可行 | Phase 2 原型调整 | 启动 designer 调整原型 |
| 代码与原型不符 | Phase 3 架构调整 | 通知 runner 修复指令 |
| 测试不通过 | Phase 3 开发修复 | 通知 runner 修复指令 |
| 模块验收不通过 | 最小必要回退点 | 按具体情况决定 |

**打回限制：**
- 每模块最多打回 3 次
- 打回必须有反馈
- 打回决策权归 orchestrator，runner 只能上报

### 4.3 预准备机制

```yaml
预准备机制:
  触发条件: Phase 1 pm-planner 完成拆解
  监听者: orchestrator
  接收者: Phase 3 pm-runner
  传递内容:
    - 模块清单
    - 依赖DAG
    - Skills需求清单
  目的: Phase 3 可提前准备技术架构方案，减少等待时间
  约束: 预准备不等于提前执行，Phase 3 仍需等待 Phase 1 完全结束
```

### 4.4 HEARTBEAT 两级记忆架构

```
项目级 HEARTBEAT ({context_root}/HEARTBEAT.md)
├── 项目概览
├── 任务看板
├── 决策记录
├── 风险追踪
├── 文件索引
└── 变更日志

任务级 HEARTBEAT ({context_root}/context_pool/progress/T{XXX}-heartbeat.md)
├── 任务目标
├── 执行进度
├── 产出物清单
├── 关键发现/代码统计
├── 依赖阻塞
└── 推理验证点记录
```

### 4.5 策略引擎

| 规则 | 触发条件 | 动作 |
|------|---------|------|
| auto_unblock | 上游任务 COMPLETED | 自动派发下游任务 |
| auto_recover | 子Agent FAILED | 按恢复配方恢复（最多1次） |
| budget_warning | 轮次超80% | 通知用户 |
| constraint_escalate | 硬约束违规 | 立即通知用户 |
| timeout_escalate | 子Agent长时间无进度 | 检查 HEARTBEAT |
| health_alert | 子Agent健康度 < 30 | 立即通知 orchestrator |

### 4.6 经验积累四级分类

| 级别 | 名称 | 沉淀位置 | 触发条件 | 作用范围 |
|------|------|---------|---------|---------|
| L1 | 微模式 | `{context_root}/lessons-learned.md` | 同类问题 ≥2 次 | 当前项目 |
| L2 | 规则强化 | Harness 策略引擎 | 微模式被 ≥3 个项目验证 | 所有项目 |
| L3 | Skill增强 | SKILL.md / references/ | 需 Agent 遵守的行为规范 | 绑定该 Skill 的 Agent |
| L4 | 独立Skill | 新建 pm-{name}/ | 跨领域通用 | 全局可用 |

---

## 5. 文件结构

```
{skills_root}/                             # Skills 安装目录
├── pm-orchestrator/SKILL.md               # 主控器
├── pm-backlog-manager/SKILL.md            # 需求池管理专家
├── pm-analyst/SKILL.md                    # 需求澄清专家
├── pm-planner/SKILL.md                    # 任务规划专家
├── pm-designer/SKILL.md                   # 原型设计专家（并行可选）
├── pm-runner/SKILL.md                     # 执行调度专家
├── pm-coder/SKILL.md                      # 编码执行专家
├── pm-researcher/SKILL.md                 # 信息检索专家
├── pm-writer/SKILL.md                     # 内容输出专家
└── AI_PM_SKills/                          # 完整架构包
    ├── ARCHITECTURE.md                    # 架构详细说明
    ├── ARCHITECTURE_SPEC.md              # 架构说明书（本文件）
    ├── ENTERPRISE_FRAMEWORK.md           # Ironforge企业工程化框架文档
    ├── README.md                          # 快速入门
    ├── QUICKSTART.md                      # 快速开始指南
    ├── pm-core/                           # 🧠 共享内核
    │   ├── context-protocol.md            # 上下文自适应协议
    │   ├── agent-lifecycle.md             # Agent 生命周期规范
    │   ├── platform-adapter.md            # 平台抽象层
    │   ├── SKILL.md                       # 内核入口
    │   ├── security/                      # 🔒 安全协议（v3.1 Ironforge）
    │   │   ├── permission-framework.md    #   权限强制执行框架
    │   │   ├── secrets-protection.md      #   敏感信息防护规则
    │   │   └── audit-protocol.md          #   审计日志协议
    │   ├── stability/                     # 🛡️ 稳定性协议（v3.1 Ironforge）
    │   │   ├── checkpoint-protocol.md     #   检查点与回滚
    │   │   ├── circuit-breaker.md         #   三级熔断机制
    │   │   └── idempotency.md             #   幂等性规范
    │   ├── observability/                 # 📊 可观测性协议（v3.1 Ironforge）
    │   │   ├── heartbeat-v2.md            #   结构化HEARTBEAT v2
    │   │   ├── metrics-protocol.md        #   指标收集协议
    │   │   └── trace-protocol.md          #   分布式追踪协议
    │   ├── references/
    │   │   ├── recovery-recipes.md        # 恢复配方
    │   │   ├── default-skills.md          # 默认技能
    │   │   ├── health-check-protocols.md  # 健康检查协议
    │   │   ├── checkpoint-recovery.md     # 检查点恢复
    │   │   ├── message-protocol.md        # 通信协议
    │   │   └── lessons-learned.md         # 经验积累
    │   └── templates/
    │       └── heartbeat-template.md      # HEARTBEAT v2模板
    ├── orchestrations/                    # 🔄 编排模板
    │   ├── code-only.yaml
    │   ├── research-only.yaml
    │   ├── analysis-only.yaml
    │   ├── doc-only.yaml
    │   ├── full-pipeline.yaml
    │   └── custom/                        # 自定义编排
    ├── harnesses/                         # ⚙️ Harness 定义（平台适配器）
    │   ├── base/                          # 🆕 公共层（v3.1 Ironforge）
    │   │   ├── permission-framework.md
    │   │   ├── security-hooks.md
    │   │   ├── audit-logging.md
    │   │   ├── checkpoint-protocol.md
    │   │   ├── handoff-protocol.md
    │   │   ├── context-engineering.md
    │   │   └── observability-config.md
    │   ├── README.md
    │   ├── orchestrator-harness.md
    │   ├── backlog-manager-harness.md
    │   ├── analyst-harness.md
    │   ├── planner-harness.md
    │   ├── designer-harness.md
    │   ├── runner-harness.md
    │   ├── coder-harness.md
    │   ├── researcher-harness.md
    │   └── writer-harness.md
    │   ├── web-design-engineer-harness.md   # 🎨 视觉设计工程（v3.1 新增）
    ├── pm-backlog-manager/
    │   ├── SKILL.md
    │   └── standalone-prompt.md
    ├── pm-analyst/
    │   ├── SKILL.md
    │   └── standalone-prompt.md
    ├── pm-planner/
    │   ├── SKILL.md
    │   └── standalone-prompt.md
    ├── pm-designer/
    │   ├── SKILL.md
    │   └── standalone-prompt.md
    ├── pm-runner/
    │   ├── SKILL.md
    │   └── standalone-prompt.md
    ├── pm-coder/
    │   ├── SKILL.md
    │   ├── standalone-prompt.md
    │   └── references/
    ├── pm-researcher/
    │   ├── SKILL.md
    │   ├── standalone-prompt.md
    │   └── references/
    └── pm-writer/
        ├── SKILL.md
        ├── standalone-prompt.md
        └── references/
    ├── web-design-engineer/                  # 🎨 视觉设计工程（v3.1 新增）
    │   ├── SKILL.md
    │   ├── standalone-prompt.md
    │   └── references/
    │       └── advanced-patterns.md
```

---

## 6. 数据流图

```
用户输入
    │
    ▼
Phase 0: pm-backlog-manager → 需求池排序 + MVP定义 → backlog.md + mvp-definition.md + batch-plan.md
    │
    ▼
orchestrator: 初始化 HEARTBEAT + 上下文池
    │
    ▼
Phase 1: pm-analyst → 需求澄清 → goal.md
    │                          ↓
    │         pm-planner → 颗粒化拆解 → modules.md + DAG + skills-needed.md
    │
    ├──→ orchestrator 审核
    │
    ├──→ Phase 2 (并行): pm-designer → 原型设计 → wireframe.html + component-tree.md
    │                                                + interaction-flow.md + page-routes.md
    │   │
    │   ├──→ RC-P2-01 覆盖度验证
    │   ├──→ RC-P2-02 用户确认（HITL Gate）
    │   └──→ 原型完成 → 成为 Phase 3 方向指标
    │
    ▼
Phase 3: pm-runner
    ├──→ RC-P3-01 架构方案树搜索验证
    ├──→ Skills 安装与验证
    ├──→ 按 DAG 批次调度
    │    ├──→ pm-coder × N → 代码 + 测试
    │    ├──→ pm-researcher → 调研报告
    │    └──→ pm-writer → 文档
    ├──→ RC-P3-02 每模块开发对齐验证
    ├──→ 自动化验收标准执行
    └──→ 初步整合 → 通知 orchestrator
    │
    ▼
Phase 4: 打回循环（按需）
    ├──→ RC-P4-01 打回根因验证
    └──→ 最小必要回退
    │
    ▼
Phase 5: 整合交付
    ├──→ RC-P5-01 整合一致性验证
    ├──→ 三角验证加权验收
    ├──→ 协作奖励模型评估
    ├──→ 经验沉淀（L1-L4）
    └──→ 清理团队资源
    │
    ▼
使用者人工验收
```

---

## 7. 版本变更摘要

| 维度 | v1 | v2 | v2.1 | v3 | v3.1 Ironforge |
|------|----|----|------|-----|---------------|
| 阶段数 | 7 阶段 | 5 阶段 | 6 阶段（+Phase 0） | 6 阶段 | 6 阶段 |
| 架构模式 | 流程驱动 | 流程驱动 | 流程驱动 | 能力驱动 | 能力驱动 + 企业工程化 |
| 独立运行 | 不支持 | 不支持 | 不支持 | 每个Skill可独立运行 | 同 v3 |
| 共享层 | shared/ | shared/ | shared/ | pm-core/ | pm-core/ + security/ + stability/ + observability/ |
| 安全机制 | 无 | 无 | 无 | 无 | **三层权限 + 敏感信息防护 + 审计日志** |
| 故障恢复 | 无 | 无 | 无 | 文档级恢复配方 | **检查点确定性恢复 + 三级熔断 + 幂等性** |
| 可观测性 | 无 | 无 | Markdown HEARTBEAT | 同 v2.1 | **HEARTBEAT v2 + 指标 + 追踪** |
| Harness | 无 | 独立文件 | 独立文件 | 独立文件 | **base/公共层 + 特化层** |
| 上下文依赖 | 必须有前置 | 必须有前置 | 必须有前置 | 三级自适应 | 同 v3 + 上下文工程 |
| 编排方式 | 固定流程 | 固定5阶段 | 固定6阶段 | 预定义+自定义编排 | 同 v3 |
| 平台绑定 | 无 | 有 | 有 | 行为规范层平台无关 | 同 v3 |
| orchestrator 职责 | 全包 | 监听+同步+收集+决策 | 同 v2 + 职责分层 | 同 v2.1 | 同 v3 |
| 需求池管理 | 无 | 无 | pm-backlog-manager | 同 v2.1 | 同 v3 |
| 原型设计 | 无 | 独立 Phase 2，阻塞开发 | 并行，方向指标 | 同 v2.1 | 同 v3 |
| 验收标准 | 无 | 三角验证 | 三角验证+自动化验收 | 同 v2.1 | 同 v3 |

---

## 8. 限制与未来方向

### 当前限制
1. 推理验证点依赖 Agent 自觉执行，无强制机制
2. 协作奖励模型不用于实时决策，仅事后评估
3. 多方案并行增加 token 消耗（架构讨论阶段约 2-3x）
4. 预准备机制尚未写入 Harness 约束
5. harnesses/ 仍绑定特定AI平台，跨平台迁移需手动适配
6. Ironforge 协议目前为规范文档，尚未实现自动化执行引擎

### v3.1 Ironforge 已解决
- ~~HEARTBEAT 手动更新，无自动定时机制~~ → HEARTBEAT v2 + 自动化监控5条规则
- ~~无安全强制执行~~ → 三层权限 + 敏感信息防护 + 审计日志
- ~~无确定性恢复~~ → 检查点协议 + 熔断 + 幂等性
- ~~每个Harness重复1000+行~~ → base/公共层 + 特化层

### 未来方向
1. **验证点自动化**：通过 hooks 自动触发推理验证点
2. **奖励模型在线化**：Phase 3 实时调整子Agent优先级
3. **方案并行 token 优化**：共享上下文池减少重复读取
4. **预准备机制写入 Harness**：显式约束 Phase 1 → Phase 3 的信息传递
5. **Skill 渐进披露 Tier 4**：基于 Agent 使用频率动态调整加载策略
6. **Harness 跨平台适配器**：为更多AI平台提供 harnesses 适配
7. **Ironforge 执行引擎**：将协议文档转化为可执行的自动化引擎
