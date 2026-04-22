# Orchestrator Harness 定义（v2 架构重构版）

> Harness 是子Agent的执行载体，定义了运行环境、绑定Skills、通信配置。
> 本文件描述 pm-orchestrator 作为瘦身后主控Agent的 Harness 配置。
> v2 核心变革：orchestrator 只做监听+同步+收集+决策，具体执行全部分发。

## 基本配置

```yaml
name: pm-orchestrator
harness_type: main_controller
description: |
  Multi-Agent 协作主控器（瘦身后）。
  只做：上下文同步 · 结果收集 · 全局监听 · 阶段切换决策 · 跨阶段打回路由 · 用户交互 · 复盘。
  不做：需求澄清、任务拆解、Skills安装、Agent调度、编码、文档。

runtime: default
max_turns: 100
```

## 目标驱动模型（Goal-Driven）

> Goal 由 pm-analyst 构建，orchestrator 使用但不构建。

### Goal 注入规则

- pm-analyst 在 Phase 1 构建 Goal → 写入 `context_pool/goal.md`
- orchestrator 审核后，Goal 的 `success_criteria` 和 `constraints` 注入后续子Agent prompt
- Phase 5 整合时，用 success_criteria 做加权验收

## 三角验证模型

### orchestrator 的验证角色

| 验证层 | 执行者 | 内容 |
|--------|--------|------|
| 确定性规则 | 子Agent自验证 | 格式、编译、关键词 |
| **语义评估** | **orchestrator** | 对比 Goal success_criteria 加权打分 |
| 人工判断 | 用户 | 原型确认、MVP交付、架构决策 |

### 验收算法

```
子Agent完成 → 自验证通过 → runner 上报 → orchestrator 语义评估
  ├── 置信度 ≥ 80% → COMPLETED
  ├── 置信度 < 80% → 打回循环（附反馈）
  └── 硬约束违规 → ESCALATE
```

## 三层 Prompt 洋葱模型

子Agent 的 prompt 由 orchestrator 构建时按三层组织：

```
Layer 1 — Identity:
  SKILL.md 核心内容 + Goal 定义
  静态，永不改变

Layer 2 — Narrative:
  从 HEARTBEAT + context_pool 动态构建
  每次子Agent启动时拼接

Layer 3 — Focus:
  具体任务描述 + 输出格式 + 工具约束
  每次切换任务时替换
```

## Skill 渐进式披露

| 层级 | 加载内容 | 时机 | Token成本 |
|------|---------|------|----------|
| Tier 1 — Catalog | name + description | spawn时注入prompt | ~50-100 |
| Tier 2 — Instructions | 完整SKILL.md | Agent自主read_file | <5000 |
| Tier 3 — Resources | 脚本、参考文档 | 按需read_file | 因情况而异 |

## 绑定的 Skills

| Skill | 路径 | 加载时机 |
|-------|------|---------|
| pm-orchestrator | pm-orchestrator/SKILL.md | always |
| agent-team-orchestration | ~/.workbuddy/skills/agent-team-orchestration/ | when_needed |

## 可用工具

| 工具 | 用途 | Phase | 权限 |
|------|------|-------|------|
| `team_create` | 创建项目团队通信通道 | Phase 1 | 全权 |
| `task` | spawn analyst/planner/designer/runner | Phase 1-3 | 全权 |
| `send_message` | Agent间结构化通信 | 全程 | 全权 |
| `team_delete` | 清理团队 | Phase 5 | 全权 |
| `read_file` | 读取子Agent产出物和HEARTBEAT | 全程 | 只读 |
| `write_to_file` | 创建HEARTBEAT/上下文池/检查点 | Phase 1,5 | 受限 |
| `replace_in_file` | 更新HEARTBEAT | 全程 | 受限 |

## 通信配置

```yaml
communication:
  project_heartbeat: ".workbuddy/HEARTBEAT.md"
  context_pool: ".workbuddy/context_pool/"
  
  team:
    create_tool: team_create
    message_tool: send_message
    cleanup_tool: team_delete

  message_types:
    - type: task_complete
      required_fields: [task_id, agent_name, status, deliverables]
    - type: task_progress
      required_fields: [task_id, progress_pct, current_step]
    - type: task_blocked
      required_fields: [task_id, block_reason, needed_from]
    - type: task_failed
      required_fields: [task_id, error_kind, error_detail, recoverable]
    - type: task_partial_success
      required_fields: [task_id, completed_items, incomplete_items, suggestion]
    - type: recovery_attempt
      required_fields: [task_id, recipe_id, attempt_count, result]
    - type: decision_request
      required_fields: [task_id, decision_description, options]
    - type: health_report
      required_fields: [task_id, agent_name, health_score, concerns, next_checkpoint]
    - type: kickback_suggestion     # 新增：runner 建议打回
      required_fields: [task_id, module_id, reason, suggested_target, feedback]
```

## 阶段切换决策规则

> orchestrator 的核心新增职责：决定何时进入下一阶段。

```yaml
phase_transition:
  P1_to_P2:
    condition: "analyst 完成 + planner 完成 + orchestrator 审核通过"
    action: "spawn pm-designer"
    on_fail: "send_message 给 analyst/planner 要求调整"
    
  P2_to_P3:
    condition: "designer 完成 + 用户确认原型"
    action: "spawn pm-runner"
    on_fail: "打回到 Phase 1（最小回退）"
    
  P3_to_P4:
    condition: "runner 上报问题 / 模块验收不通过"
    action: "按打回路由表执行"
    on_pass: "直接进入 Phase 5"
    
  P4_to_P5:
    condition: "所有打回问题解决 + 所有模块验收通过"
    action: "整合交付 + 复盘"
```

## 打回路由决策

> orchestrator 专属职责，runner 不能自行决定打回。

```yaml
kickback_routing:
  # 原型不通过
  - trigger: "原型校验失败"
    min_rollback: "Phase 1 中不通过的功能需求"
    action: "spawn analyst 局部修正 → spawn designer 局部调整"
    
  # 架构不可行
  - trigger: "架构方案评审不通过"
    min_rollback: "Phase 2 原型调整"
    action: "spawn designer 调整原型 → runner 重新架构讨论"
    
  # 代码与原型不符
  - trigger: "runner 上报代码-原型不一致"
    min_rollback: "Phase 3 架构调整"
    action: "send_message 给 runner 修复指令"
    
  # 测试不通过
  - trigger: "runner 上报测试失败"
    min_rollback: "Phase 3 开发修复"
    action: "send_message 给 runner 修复指令"
    
  # 模块验收不通过
  - trigger: "模块验收加权得分 < 80%"
    min_rollback: "最小必要回退点"
    action: "按具体情况决定回退目标"
    
  # 打回限制
  max_kickback_per_module: 3
  kickback_must_have_feedback: true
```

## 策略引擎

```yaml
policies:
  auto_unblock:
    trigger: "上游任务 COMPLETED"
    action: "检查 BLOCKED 任务，依赖满足则自动派发"
    note: "由 runner 执行，orchestrator 监听结果"

  auto_recover:
    trigger: "子Agent FAILED"
    action: "按恢复配方恢复，最多1次"
    note: "由 runner 执行"

  budget_warning:
    trigger: "项目总轮次 > max_turns 的 80%"
    action: "评估剩余任务优先级，通知用户"
    
  constraint_escalate:
    trigger: "子Agent输出违反硬约束"
    action: "立即通知用户，不自动恢复"
    
  timeout_escalate:
    trigger: "子Agent长时间无进度"
    action: "主动 read_file 检查 HEARTBEAT"
    note: "被动触发：runner health_alert 或子Agent task_blocked"

  auto_cleanup:
    trigger: "所有任务完成"
    action: "shutdown_request + team_delete"
```

## 健康度检查

```yaml
health_monitoring:
  factors:
    - name: progress_velocity
      weight: 0.3
    - name: heartbeat_freshness
      weight: 0.25
    - name: error_count
      weight: 0.2
    - name: constraint_compliance
      weight: 0.25

  thresholds:
    healthy: 75
    attention: 50
    warning: 30
    critical: 0
    
  # runner 上报健康度 < 30 时，orchestrator 介入
  on_critical:
    - "send_message 询问 runner 当前情况"
    - "如无改善 → shutdown_request + 重新 spawn 或 ESCALATE"
```

## 检查点与崩溃恢复

```yaml
checkpoints:
  types:
    - type: project_snapshot
      trigger: "每个 Phase 完成时"
      location: ".workbuddy/checkpoints/project-phase{N}-v{n}.json"
      
    - type: task_snapshot
      trigger: "子Agent里程碑完成时"
      location: ".workbuddy/checkpoints/task-{task_id}-v{n}.json"
      
    - type: decision_snapshot
      trigger: "关键决策后"
      location: ".workbuddy/checkpoints/decision-{d_id}.json"

  crash_recovery:
    L1: "HEARTBEAT 压缩恢复（子Agent会话中断）"
    L2: "检查点快照恢复（orchestrator中断/多Agent异常）"
    L3: "人工介入恢复（不可恢复的系统性故障）"
```

## 经验积累

```yaml
experience:
  levels:
    L1: "微模式 → shared/lessons-learned.md"
    L2: "规则强化 → 策略引擎规则"
    L3: "Skill增强 → SKILL.md / references/"
    L4: "独立Skill → 新建 pm-{name}/"
    
  retrospective:
    timing: "Phase 5 整合交付完成后"
    steps:
      - "数据收集: 扫描所有任务HEARTBEAT"
      - "模式识别: 聚类相似问题"
      - "经验分级: L1→L2→L3→L4"
      - "更新资产: 按级别更新"
      - "生成复盘报告"
```

## 预算控制

```yaml
budget:
  max_turns: 100
  per_task_max_turns:
    analyst: 25
    planner: 30
    designer: 35
    runner: 80
    coder: 50
    researcher: 40
    writer: 35
```

## 推理验证点（Reasoning Checkpoints）

> **设计参考**：DeepSeek-R1 的"推理链检查点"思想。
> **核心理念**：在关键决策节点，Agent 必须暂停推理、输出显式的验证步骤，然后才能继续。
> 防止"想当然"导致的错误沿链路放大。

### 验证点定义

```yaml
reasoning_checkpoints:
  # Phase 1 验证点
  - id: "RC-P1-01"
    name: "需求理解验证"
    phase: "Phase 1"
    trigger: "pm-analyst 完成追问后"
    verification: |
      1. 输出需求理解摘要
      2. 用户确认是否准确（HITL Gate）
      3. 检查 In Scope / Out Scope 是否与用户意图一致
    failure_action: "追加追问轮次"
    
  # Phase 2 验证点
  - id: "RC-P2-01"
    name: "原型覆盖度验证"
    phase: "Phase 2"
    trigger: "pm-designer 完成原型后"
    verification: |
      1. 逐条对照 goal.md 的 success_criteria
      2. 逐模块对照 modules.md
      3. 检查交互流程完整性
    failure_action: "打回到 Phase 1 局部需求补充"

  - id: "RC-P2-02"
    name: "原型用户确认"
    phase: "Phase 2"
    trigger: "orchestrator 通过覆盖度验证后"
    verification: |
      1. 向用户展示原型
      2. 用户确认原型符合预期（HITL Gate）
    failure_action: "收集反馈 → spawn designer 调整"

  # Phase 3 验证点
  - id: "RC-P3-01"
    name: "架构方案可行性验证"
    phase: "Phase 3"
    trigger: "pm-runner 收集多方案后"
    verification: |
      1. 检查方案是否覆盖原型中所有组件
      2. 检查技术栈是否满足 Goal constraints
      3. 检查方案间是否存在矛盾
    failure_action: "回退到架构讨论/原型调整"

  - id: "RC-P3-02"
    name: "模块开发对齐验证"
    phase: "Phase 3"
    trigger: "每个 pm-coder 完成模块开发后"
    verification: |
      1. 代码实现 vs 原型设计一致性检查
      2. API 端点 vs 原型交互流一致性
      3. 组件树 vs 实际代码结构一致性
    failure_action: "打回到该模块开发修复"

  # Phase 4 验证点
  - id: "RC-P4-01"
    name: "打回根因验证"
    phase: "Phase 4"
    trigger: "打回循环触发前"
    verification: |
      1. 确认打回原因是否准确
      2. 确认回退目标是否为"最小必要回退点"
      3. 确认回退不会影响已验收通过的模块
    failure_action: "调整回退范围"

  # Phase 5 验证点
  - id: "RC-P5-01"
    name: "整合一致性验证"
    phase: "Phase 5"
    trigger: "模块化讨论验收时"
    verification: |
      1. 模块间接口一致性
      2. 数据模型一致性
      3. 用户体验一致性
    failure_action: "指定模块修复"

  # 通用验证点
  - id: "RC-GEN-01"
    name: "Goal 约束合规验证"
    phase: "任何 Phase"
    trigger: "硬约束可能被违反时"
    verification: |
      1. 逐条检查 Goal constraints
      2. 硬约束违规 → ESCALATE
      3. 软约束偏离 → 记录并评估
    failure_action: "硬约束违规 → 立即中断通知用户"
```

### 验证点执行流程

```
Agent 到达验证点
    │
    ├──→ Agent 暂停当前推理
    ├──→ Agent 输出显式验证步骤（文字化推理过程）
    ├──→ orchestrator 评估验证结果
    │    ├── 通过 → Agent 继续推理
    │    ├── 不通过 → 按失败动作处理
    │    └── 需要人工 → HITL Gate（通知用户确认）
    └──→ 更新 HEARTBEAT 记录验证结果
```

## 协作奖励模型（Collaborative Reward Model）

> **设计参考**：协作 RL 的"团队奖励"思想。
> **核心理念**：评估 Agent 贡献时，不只看个体产出质量，还看其对团队协作的正外部性。
> 奖励协作行为，惩罚"自顾自"行为，防止局部最优损害全局质量。

### 奖励信号定义

```yaml
reward_signals:
  # 个体质量奖励（Individual Quality）
  individual:
    - id: "IQ-01"
      name: "交付物质量"
      weight: 0.3
      metric: "三角验证置信度"
      range: [0, 1]
      
    - id: "IQ-02"
      name: "约束合规"
      weight: 0.2
      metric: "约束违规次数（反向）"
      range: [0, 1]

  # 协作正外部性奖励（Collaborative Externality）
  collaborative:
    - id: "CE-01"
      name: "下游便利度"
      weight: 0.2
      metric: "下游 Agent 是否无需额外澄清即可使用本 Agent 产出"
      range: [0, 1]
      description: |
        高分行为：
        - 产出物格式规范，下游无需二次解析
        - 上下文信息完整，下游无需追问
        - 接口定义清晰，下游可直接对接
        低分行为：
        - 产出物模糊，下游需要追问
        - 格式不规范，下游需要转换
        - 信息缺失，下游需要自行补充调研

    - id: "CE-02"
      name: "信息同步及时性"
      weight: 0.1
      metric: "HEARTBEAT 更新及时率 + send_message 响应速度"
      range: [0, 1]
      description: |
        高分行为：
        - 关键节点及时更新 HEARTBEAT
        - 阻塞时立即 send_message 通知
        - 产出变更时主动同步上下文池
        低分行为：
        - HEARTBEAT 长时间不更新
        - 阻塞时不通知
        - 产出变更不同步

    - id: "CE-03"
      name: "可复用性"
      weight: 0.1
      metric: "产出物是否可被其他模块/项目复用"
      range: [0, 1]
      description: |
        高分行为：
        - 组件设计遵循通用规范
        - 工具函数独立可复用
        - 文档结构清晰可迁移
        低分行为：
        - 硬编码业务逻辑
        - 组件耦合不可拆分

    - id: "CE-04"
      name: "冲突避免"
      weight: 0.1
      metric: "是否主动避免与其他 Agent 的文件/资源冲突"
      range: [0, 1]
      description: |
        高分行为：
        - 修改前检查 HEARTBEAT 文件索引
        - 主动声明文件所有权
        - 发现冲突主动协调
        低分行为：
        - 未检查就修改他人文件
        - 不声明文件所有权
        - 发现冲突不报告

  # 惩罚信号（Penalty）
  penalty:
    - id: "PN-01"
      name: "下游阻塞惩罚"
      weight: -0.3
      metric: "因本 Agent 产出问题导致下游 Agent 阻塞的次数"
      description: |
        触发条件：
        - 产出物格式错误导致下游无法解析
        - 信息缺失导致下游无法继续
        - 接口变更未同步导致下游对接失败
        
    - id: "PN-02"
      name: "打回连锁惩罚"
      weight: -0.2
      metric: "因本 Agent 产出问题导致跨阶段打回的次数"
      description: |
        触发条件：
        - 需求文档模糊导致开发方向偏离 → 打回 Phase 1
        - 原型设计遗漏导致代码无法实现 → 打回 Phase 2
        - 代码与原型不符 → 打回 Phase 3
```

### 奖励计算与使用

```yaml
reward_calculation:
  # 综合奖励 = Σ(个体奖励) + Σ(协作奖励) + Σ(惩罚)
  formula: |
    total_reward = 
      (IQ-01 * 0.3 + IQ-02 * 0.2) +
      (CE-01 * 0.2 + CE-02 * 0.1 + CE-03 * 0.1 + CE-04 * 0.1) +
      (PN-01 * (-0.3) + PN-02 * (-0.2))
    
    normalized: total_reward ∈ [-0.5, 1.0]

  # 奖励使用方式
  usage:
    - "Phase 5 复盘时：低分 Agent 的 Skill 规范需优先改进（经验积累 L2/L3）"
    - "低分模式识别：连续 3 个项目得分 < 0.3 → 触发 Skill 重写（L4）"
    - "高分模式沉淀：连续 3 个项目得分 > 0.8 → 最佳实践写入 references/"
    - "runer 调度优化：同类型任务优先选择历史高分 Agent 配置"
    - "不直接用于实时决策（避免过拟合），仅用于事后经验沉淀"
```

### 奖励数据采集

```yaml
data_collection:
  # 自动采集（无需额外 token 消耗）
  automatic:
    - "HEARTBEAT 更新频率 → CE-02"
    - "约束违规次数 → IQ-02（从恢复台账统计）"
    - "打回链路追溯 → PN-01, PN-02"
    - "文件所有权冲突 → CE-04（从 HEARTBEAT 文件索引检查）"
    
  # 评估时采集（Phase 5 复盘时一次性评估）
  retrospective:
    - "下游便利度 → CE-01（读取下游 HEARTBEAT，检查是否有'上游信息缺失'记录）"
    - "产出物可复用性 → CE-03（评估代码/文档的模块化程度）"
    - "交付物质量 → IQ-01（复用三角验证结果）"
```

## 执行流程绑定（v2 五阶段）

```
Phase 1: 需求→分析→拆解
  → write_to_file 创建 HEARTBEAT + 上下文池
  → task(spawn analyst) → task(spawn planner) → 审核拆解结果

Phase 2: 原型+UI定型
  → task(spawn designer) → 原型校验 + 用户确认

Phase 3: 架构→开发→测试
  → task(spawn runner) → 监听 + 三角验证 + 健康度监控

Phase 4: 打回循环
  → 按打回路由表决策 → 最小必要回退

Phase 5: 整合交付
  → 模块化验收 → 整合检查 → 检查点 → 复盘 → team_delete
```
