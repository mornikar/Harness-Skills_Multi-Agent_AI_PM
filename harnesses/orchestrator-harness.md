# Orchestrator Harness 定义（v3.1 Ironforge 重构版）

> pm-orchestrator Multi-Agent 协作主控器的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 orchestrator 特化逻辑。
> v2 核心变革：orchestrator 只做监听+同步+收集+决策，具体执行全部分发。

---

## 版本与继承

```yaml
harness_version: "2.0"
compatible_skill_version: ">=3.0"
compatible_framework: "Ironforge v1.0"

inherits:
  - base/permission-framework.md
  - base/security-hooks.md
  - base/audit-logging.md
  - base/checkpoint-protocol.md
  - base/handoff-protocol.md
  - base/context-engineering.md
  - base/observability-config.md
```

---

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

---

## 特化配置

```yaml
specialization:
  permission_override:
    mode: "default"
    allowed_tools: [read_file, search_file, search_content, list_dir, send_message, task, team_create, team_delete]
    blocked_tools: [write_to_file, replace_in_file, execute_command, delete_file]
    max_turns: 100

  security_override: {}

  audit_override:
    additional_events:
      - event_type: phase_transition
        detail: {from_phase, to_phase, reason}
      - event_type: kickback_executed
        detail: {module_id, from_phase, to_phase, reason}
      - event_type: goal_verified
        detail: {goal_id, confidence, criteria_results}

  # v3.1 P1 稳定性覆盖
  checkpoint_override:
    frequency: "every_step"
    retention: "minimal"
    rollback_strategy: "git_first"

  circuit_breaker_override:
    agent_level:
      failure_threshold: 1               # orchestrator失败最严重
      window: "15min"
    task_level:
      retry_budget: 1                    # orchestrator任务不自动重试
    # orchestrator 熔断 → 人工介入
    on_breaker_open:
      action: "立即通知用户，请求人工决策"

  idempotency_override:
    # orchestrator 状态更新幂等
    state_update:
      rule: "状态转换操作天然幂等（同状态重复设置无副作用）"
    phase_transition:
      pre_check: "确认当前阶段确实不是目标阶段"
      strategy: "已在目标阶段 → 跳过"
```

---

## Orchestrator 特有规范

### 三角验证模型

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

### 阶段切换决策规则

```yaml
phase_transition:
  P0_to_P1:
    condition: "backlog-manager 完成 + MVP确认"
    action: "spawn pm-analyst"

  P1_to_P2:
    condition: "analyst 完成 + planner 完成 + orchestrator 审核通过"
    action: "spawn pm-designer"

  P2_to_P3:
    condition: "designer 完成 + 用户确认原型"
    action: "spawn pm-runner"

  P3_to_P4:
    condition: "runner 上报问题 / 模块验收不通过"
    action: "按打回路由表执行"

  P4_to_P5:
    condition: "所有打回问题解决 + 所有模块验收通过"
    action: "整合交付 + 复盘"
```

### 打回路由决策

```yaml
kickback_routing:
  - trigger: "原型校验失败"
    min_rollback: "Phase 1 中不通过的功能需求"
    action: "spawn analyst 局部修正 → spawn designer 局部调整"

  - trigger: "架构方案评审不通过"
    min_rollback: "Phase 2 原型调整"
    action: "spawn designer 调整原型 → runner 重新架构讨论"

  - trigger: "代码与原型不符"
    min_rollback: "Phase 3 架构调整"
    action: "send_message 给 runner 修复指令"

  - trigger: "测试不通过"
    min_rollback: "Phase 3 开发修复"
    action: "send_message 给 runner 修复指令"

  max_kickback_per_module: 3
  kickback_must_have_feedback: true
```

### 推理验证点

```yaml
reasoning_checkpoints:
  - id: "RC-P0-01"
    name: "MVP范围验证"
    phase: "Phase 0"
    trigger: "backlog-manager 完成 MVP 定义后"
    verification: ["P0需求是否覆盖核心路径", "MVP范围是否经用户确认"]
    failure_action: "调整MVP范围"

  - id: "RC-P1-01"
    name: "需求理解验证"
    phase: "Phase 1"
    trigger: "pm-analyst 完成追问后"
    failure_action: "追加追问轮次"

  - id: "RC-P2-01"
    name: "原型覆盖度验证"
    phase: "Phase 2"
    failure_action: "打回到 Phase 1 局部需求补充"

  - id: "RC-P3-01"
    name: "架构方案可行性验证"
    phase: "Phase 3"
    failure_action: "回退到架构讨论"

  - id: "RC-P3-02"
    name: "模块开发对齐验证"
    phase: "Phase 3"
    failure_action: "打回到该模块开发修复"

  - id: "RC-P4-01"
    name: "打回根因验证"
    phase: "Phase 4"
    failure_action: "调整回退范围"

  - id: "RC-P5-01"
    name: "整合一致性验证"
    phase: "Phase 5"
    failure_action: "指定模块修复"

  - id: "RC-GEN-01"
    name: "Goal 约束合规验证"
    phase: "任何 Phase"
    failure_action: "硬约束违规 → 立即中断通知用户"
```

### 协作奖励模型

```yaml
reward_signals:
  individual:
    - id: "IQ-01"
      name: "交付物质量"
      weight: 0.3
    - id: "IQ-02"
      name: "约束合规"
      weight: 0.2

  collaborative:
    - id: "CE-01"
      name: "下游便利度"
      weight: 0.2
    - id: "CE-02"
      name: "信息同步及时性"
      weight: 0.1
    - id: "CE-03"
      name: "可复用性"
      weight: 0.1
    - id: "CE-04"
      name: "冲突避免"
      weight: 0.1

  penalty:
    - id: "PN-01"
      name: "下游阻塞惩罚"
      weight: -0.3
    - id: "PN-02"
      name: "打回连锁惩罚"
      weight: -0.2
```

### 预算控制

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

---

*Orchestrator Harness v2.1 (Ironforge P1) — 2026-04-24*
