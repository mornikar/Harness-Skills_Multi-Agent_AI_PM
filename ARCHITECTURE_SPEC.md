# AI PM Skills v2 架构说明书

> 版本: v2.0  
> 日期: 2026-04-22  
> 状态: 已实施  
> 作者: AI PM Team

---

## 1. 架构总览

### 1.1 设计哲学

AI PM Skills v2 的核心设计哲学是 **"解耦瘦身 + 校准基石 + 多信号验证"**：

1. **解耦瘦身**：将旧 orchestrator 的全包职责拆分为 7 个专业 Agent，每个 Agent 只做一件事
2. **校准基石**：原型设计作为独立阶段，是开发与验收的校准基准
3. **多信号验证**：三角验证 + 推理验证点 + 协作奖励模型，多信号收敛 = 可靠性

### 1.2 五阶段流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     pm-orchestrator（瘦身后：只做监听+同步+收集+决策）         │
│                                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  阶段切换决策  │ │  打回路由    │ │  三角验证     │ │  复盘沉淀     │       │
│  │  推理验证点    │ │  最小回退    │ │  语义评估     │ │  奖励模型     │       │
│  │  用户确认     │ │  路由决策    │ │  加权验收     │ │  L1-L4分级   │       │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       │                            │                            │
 Phase 1 团队              Phase 2 团队               Phase 3 团队
       │                            │                            │
       ▼                            ▼                            ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  pm-analyst     │      │  pm-designer    │      │  pm-runner      │
│  需求澄清        │─────→│  原型设计        │─────→│  执行调度        │
│  Goal构建       │      │  线框图/组件树   │      │  架构讨论        │
│      ↓          │      │  多方案设计      │      │  Skills管理      │
│  pm-planner     │      │  原型自检       │      │  子Agent调度     │
│  颗粒化拆解      │      │                 │      │  ↓  ↓  ↓        │
│  依赖DAG        │      └─────────────────┘      │  coder×N       │
│  Skills分析     │                                │  researcher    │
└─────────────────┘                                │  writer        │
                                                   └─────────────────┘
       │                            │                            │
       └────────────────────────────┼────────────────────────────┘
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
```

---

## 2. 七大 Agent 角色体系

### 2.1 Agent 一览

| Agent | 层级 | 核心职责 | 触发方式 | Harness Mode | Max Turns |
|-------|------|---------|---------|-------------|-----------|
| pm-orchestrator | 主控 | 监听+同步+收集+决策+打回路由+复盘 | 用户触发 | default | 100 |
| pm-analyst | Phase 1 | 需求澄清 + Goal构建 | orchestrator spawn | default | 25 |
| pm-planner | Phase 1 | 颗粒化拆解 + 依赖DAG + Skills分析 | orchestrator spawn | default | 30 |
| pm-designer | Phase 2 | 原型设计 + 组件树 + 多方案 | orchestrator spawn | acceptEdits | 35 |
| pm-runner | Phase 3 | 架构讨论 + Skills管理 + 子Agent调度 | orchestrator spawn | acceptEdits | 80 |
| pm-coder | Phase 3 子Agent | 编码执行 + 测试 + 调试 | runner spawn | acceptEdits | 50 |
| pm-researcher | Phase 3 子Agent | 信息检索 + 竞品分析 | runner/coder spawn | acceptEdits | 40 |
| pm-writer | Phase 3 子Agent | 文档输出 + PRD | runner/coder spawn | acceptEdits | 35 |

### 2.2 委托关系

```
orchestrator
    ├──→ pm-analyst（Phase 1）
    ├──→ pm-planner（Phase 1）
    ├──→ pm-designer（Phase 2）
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
| 文件位置 | `~/.workbuddy/skills/pm-{name}/SKILL.md` | `AI_PM_SKills/harnesses/{name}-harness.md` |
| 生命周期 | 项目级（不随任务变化） | 任务级（每次 spawn 重新配置） |

---

## 3. 核心创新点

### 3.1 推理验证点（Reasoning Checkpoints）

> 设计参考：DeepSeek-R1 的"推理链检查点"思想

在关键决策节点，Agent 必须暂停推理、输出显式的验证步骤，然后才能继续。防止"想当然"导致的错误沿链路放大。

**验证点清单：**

| ID | 名称 | Phase | 触发时机 | 执行者 |
|----|------|-------|---------|--------|
| RC-P1-01 | 需求理解验证 | Phase 1 | analyst 完成追问后 | analyst |
| RC-P1-01b | 拆解完整性验证 | Phase 1 | planner 完成拆解后 | planner |
| RC-P2-01 | 原型覆盖度验证 | Phase 2 | designer 完成原型后 | designer |
| RC-P2-02 | 原型用户确认 | Phase 2 | 覆盖度验证通过后 | orchestrator + 用户 |
| RC-P3-01 | 架构方案可行性验证 | Phase 3 | runner 收集多方案后 | runner |
| RC-P3-02 | 模块开发对齐验证 | Phase 3 | 每个模块开发完成后 | runner |
| RC-P3-02a | 代码-原型一致性 | Phase 3 | 模块开发完成时 | coder |
| RC-P4-01 | 打回根因验证 | Phase 4 | 打回循环触发前 | orchestrator |
| RC-P5-01 | 整合一致性验证 | Phase 5 | 模块化讨论验收时 | orchestrator |

**执行流程：**
```
Agent 到达验证点
    ├──→ 暂停当前推理
    ├──→ 输出显式验证步骤（文字化推理过程）
    ├──→ orchestrator 评估验证结果
    │    ├── 通过 → Agent 继续推理
    │    ├── 不通过 → 按失败动作处理
    │    └── 需要人工 → HITL Gate
    └──→ 更新 HEARTBEAT 记录验证结果
```

### 3.2 协作奖励模型（Collaborative Reward Model）

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

**使用方式（不用于实时决策，仅用于事后经验沉淀）：**
- Phase 5 复盘时：低分 Agent 的 Skill 规范需优先改进
- 连续 3 个项目得分 < 0.3 → 触发 Skill 重写
- 连续 3 个项目得分 > 0.8 → 最佳实践写入 references/

### 3.3 树搜索架构讨论（Architecture Tree Search）

> 设计参考：论文中"树搜索"思想

在架构阶段探索多棵子树，基于评估剪枝，而非仅比较两个方案。

**流程：**
1. 识别架构决策点（状态管理、数据层、路由架构等）
2. 为每个决策点生成 2-3 个候选方案（spawn 多个 coder Agent）
3. 局部评估 + 剪枝（原型覆盖度 0.3 + 技术可行性 0.25 + 开发效率 0.2 + 可维护性 0.15 + 扩展性 0.1）
4. 全局组合优化（各决策点 Top 方案组合，检查一致性）
5. 上报 orchestrator 最终决策

### 3.4 多方案设计并行（Design Tree Search）

在原型阶段，当核心交互流程有 ≥2 种可行方案时，pm-designer 必须输出多方案对比：
- 方案A：保守方案（成熟模式）
- 方案B：创新方案（可能更好但有风险）
- 方案C：简化方案（快速实现）

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
| 原型不通过 | Phase 1 中不通过的功能需求 | spawn analyst 局部修正 |
| 架构不可行 | Phase 2 原型调整 | spawn designer 调整原型 |
| 代码与原型不符 | Phase 3 架构调整 | send_message 给 runner 修复指令 |
| 测试不通过 | Phase 3 开发修复 | send_message 给 runner 修复指令 |
| 模块验收不通过 | 最小必要回退点 | 按具体情况决定 |

**打回限制：**
- 每模块最多打回 3 次
- 打回必须有反馈
- 打回决策权归 orchestrator，runner 只能上报

### 4.3 HEARTBEAT 两级记忆架构

```
项目级 HEARTBEAT (.workbuddy/HEARTBEAT.md)
├── 项目概览
├── 任务看板
├── 决策记录
├── 风险追踪
├── 文件索引
└── 变更日志

任务级 HEARTBEAT (context_pool/progress/T{XXX}-heartbeat.md)
├── 任务目标
├── 执行进度
├── 产出物清单
├── 关键发现/代码统计
├── 依赖阻塞
└── 推理验证点记录
```

### 4.4 策略引擎

| 规则 | 触发条件 | 动作 |
|------|---------|------|
| auto_unblock | 上游任务 COMPLETED | 自动派发下游任务 |
| auto_recover | 子Agent FAILED | 按恢复配方恢复（最多1次） |
| budget_warning | 轮次超80% | 通知用户 |
| constraint_escalate | 硬约束违规 | 立即通知用户 |
| timeout_escalate | 子Agent长时间无进度 | 检查 HEARTBEAT |
| health_alert | 子Agent健康度 < 30 | 立即通知 orchestrator |

### 4.5 经验积累四级分类

| 级别 | 名称 | 沉淀位置 | 触发条件 | 作用范围 |
|------|------|---------|---------|---------|
| L1 | 微模式 | shared/lessons-learned.md | 同类问题 ≥2 次 | 当前项目 |
| L2 | 规则强化 | Harness 策略引擎 | 微模式被 ≥3 个项目验证 | 所有项目 |
| L3 | Skill增强 | SKILL.md / references/ | 需 Agent 遵守的行为规范 | 绑定该 Skill 的 Agent |
| L4 | 独立Skill | 新建 pm-{name}/ | 跨领域通用 | 全局可用 |

---

## 5. 文件结构

```
~/.workbuddy/skills/                          # 用户级 Skills
├── pm-orchestrator/SKILL.md                   # 主控器（v2 瘦身后）
├── pm-analyst/SKILL.md                        # 需求澄清专家
├── pm-planner/SKILL.md                        # 任务规划专家
├── pm-designer/SKILL.md                       # 原型设计专家
├── pm-runner/SKILL.md                         # 执行调度专家
├── pm-coder/SKILL.md                          # 编码执行专家
├── pm-researcher/SKILL.md                     # 信息检索专家
├── pm-writer/SKILL.md                         # 内容输出专家
└── AI_PM_SKills/                              # 完整架构包
    ├── ARCHITECTURE.md                        # 架构详细说明
    ├── ARCHITECTURE_SPEC.md                   # 架构说明书（本文件）
    ├── README.md                              # 快速入门
    ├── QUICKSTART.md                          # 快速开始指南
    ├── harnesses/                             # Harness 定义
    │   ├── orchestrator-harness.md
    │   ├── analyst-harness.md
    │   ├── planner-harness.md
    │   ├── designer-harness.md
    │   ├── runner-harness.md
    │   ├── coder-harness.md
    │   ├── researcher-harness.md
    │   └── writer-harness.md
    ├── pm-analyst/SKILL.md
    ├── pm-planner/SKILL.md
    ├── pm-designer/SKILL.md
    ├── pm-runner/SKILL.md
    ├── pm-coder/                              # Coder 详细参考
    │   ├── SKILL.md
    │   └── references/                        # 渐进加载参考
    │       ├── heartbeat-ops.md
    │       ├── code-standards.md
    │       ├── acceptance-criteria.md
    │       ├── handoff-protocol.md
    │       ├── hooks-specification.md
    │       ├── code-review-protocol.md
    │       └── debugging-protocol.md
    ├── pm-researcher/
    │   ├── SKILL.md
    │   └── references/report-templates.md
    ├── pm-writer/
    │   ├── SKILL.md
    │   └── references/doc-templates.md
    ├── shared/                                # 共享资源
    │   ├── references/
    │   │   ├── recovery-recipes.md
    │   │   ├── default-skills.md
    │   │   ├── health-check-protocols.md
    │   │   ├── checkpoint-recovery.md
    │   │   └── lessons-learned.md
    │   └── templates/
    │       └── heartbeat-template.md
    └── scripts/
        └── setup-clawhub.ps1
```

---

## 6. 数据流图

```
用户输入
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
    ▼
Phase 2: pm-designer → 原型设计 → wireframe.html + component-tree.md
    │                                    + interaction-flow.md + page-routes.md
    │
    ├──→ RC-P2-01 覆盖度验证
    ├──→ RC-P2-02 用户确认（HITL Gate）
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
    └──→ 初步整合 → send_message 给 orchestrator
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
    └──→ team_delete 清理
    │
    ▼
用户交付
```

---

## 7. v1 → v2 变更摘要

| 维度 | v1 | v2 |
|------|----|----|
| 阶段数 | 7 阶段 | 5 阶段 |
| orchestrator 职责 | 全包（需求+拆解+调度+编码+文档） | 只做监听+同步+收集+决策 |
| 原型设计 | 无 | 独立 Phase 2，校准基石 |
| 需求澄清 | orchestrator 自己做 | pm-analyst |
| 任务拆解 | orchestrator 自己做 | pm-planner |
| 开发调度 | orchestrator 直接派发 | pm-runner（二级调度） |
| 架构讨论 | 无 | 树搜索多方案并行 |
| 验证机制 | 无 | 三角验证 + 推理验证点 |
| Agent 协作评估 | 无 | 协作奖励模型 |
| 多方案设计 | 无 | 设计树搜索 |
| 打回机制 | 简单回退 | 最小必要回退 + 打回路由表 |
| 经验积累 | 无 | L1-L4 四级分类 + 演化循环 |

---

## 8. 限制与未来方向

### 当前限制
1. 推理验证点依赖 Agent 自觉执行，无强制机制
2. 协作奖励模型不用于实时决策，仅事后评估
3. 多方案并行增加 token 消耗（架构讨论阶段约 2-3x）
4. HEARTBEAT 手动更新，无自动定时机制

### 未来方向
1. **验证点自动化**：通过 hooks 自动触发推理验证点
2. **奖励模型在线化**：Phase 3 实时调整子Agent优先级
3. **方案并行 token 优化**：共享上下文池减少重复读取
4. **HEARTBEAT 自动化**：定时检查 + 自动压缩
5. **Skill 渐进披露 Tier 4**：基于 Agent 使用频率动态调整加载策略
