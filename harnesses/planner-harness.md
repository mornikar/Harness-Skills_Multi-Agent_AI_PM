# Planner Harness 定义（v3.1 Ironforge 重构版）

> pm-planner 任务规划专家的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 planner 特化逻辑。

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
name: pm-planner
harness_type: specialist_agent
description: |
  任务规划专家。将需求最小颗粒化、模块化、功能化拆解，构建依赖DAG。

runtime: default
max_turns: 30
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    explore_phase_only: true
    mode: "default"
    allowed_extra: [write_to_file, execute_command]
    command_whitelist: [clawhub]           # 仅 clawhub list
    blocked_tools: [replace_in_file, delete_file]
    max_turns: 30

  security_override: {}

  audit_override:
    additional_events:
      - event_type: dag_created
        detail: {module_count, edge_count, max_depth}

  # v3.1 P1 稳定性覆盖
  checkpoint_override:
    frequency: "verification_only"
    retention: "verification_only"
    rollback_strategy: "reverse_only"

  circuit_breaker_override:
    agent_level:
      failure_threshold: 2               # planner失败影响下游，阈值更低
      window: "20min"
    task_level:
      retry_budget: 3

  idempotency_override:
    write_to_file:
      always_check_exists: true
```

---

## Planner 特有规范

### 约束

```yaml
constraints:
  - "每个模块必须包含验收标准"
  - "依赖关系必须形成DAG，不允许循环依赖"
  - "单模块功能点不超过5个"
  - "每个任务必须可在一个Agent会话内完成"
  - "Skills需求清单必须区分必需和可选"
```

### 推理验证点

```yaml
reasoning_checkpoint:
  id: "RC-P1-01b"
  name: "拆解完整性验证"
  trigger: "完成拆解后、输出文档前"
  verification:
    - "每个需求是否有对应的模块"
    - "每个模块的验收标准是否可度量"
    - "DAG 是否无循环、所有模块可达"
    - "Skills 需求是否覆盖所有任务类型"
  failure_action: "补充遗漏模块或修正DAG"
```

### DAG 验证规则

```yaml
dag_validation:
  no_cycles: true
  single_root: true
  all_reachable: true
  max_depth: 5
```

### 协作奖励模型

```yaml
reward_alignment:
  high_score_behaviors:
    - "模块划分独立可测试，coder 无需跨模块调试"
    - "DAG 清晰，runner 可直接按批次调度"
  low_score_behaviors:
    - "模块间耦合导致 coder 开发时频繁依赖其他模块"
    - "DAG 有遗漏导致 runner 调度顺序出错"
```

---

*Planner Harness v2.1 (Ironforge P1) — 2026-04-24*
