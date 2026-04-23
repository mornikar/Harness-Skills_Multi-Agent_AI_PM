# Backlog Manager Harness 定义（v3.1 Ironforge 重构版）

> pm-backlog-manager 需求池管理专家的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 backlog-manager 特化逻辑。

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
name: pm-backlog-manager
harness_type: specialist_agent
description: |
  需求池管理专家。负责需求池管理、优先级排序、MVP定义、分批交付计划。

runtime: default
max_turns: 20
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    mode: "default"
    allowed_extra: [write_to_file, replace_in_file]
    blocked_tools: [execute_command, delete_file]
    max_turns: 20

  security_override: {}

  audit_override:
    additional_events:
      - event_type: mvp_defined
        detail: {p0_count, p1_count, p2_count, mvp_scope}
      - event_type: backlog_created
        detail: {total_requirements, categories}

  # v3.1 P1 稳定性覆盖
  checkpoint_override:
    frequency: "verification_only"
    retention: "verification_only"
    rollback_strategy: "reverse_only"

  circuit_breaker_override:
    agent_level:
      failure_threshold: 3
      window: "30min"
    task_level:
      retry_budget: 3

  idempotency_override:
    write_to_file:
      always_check_exists: true
```

---

## Backlog Manager 特有规范

### 约束

```yaml
constraints:
  - "不做需求澄清，只做分类和排序（需求澄清归 pm-analyst）"
  - "MVP定义必须经用户确认后才算完成"
  - "批次交付计划必须标注每批的进入条件"
  - "P0需求必须包含至少一个核心路径功能"
  - "不产出代码"
```

### Sensors

```yaml
sensors:
  computational:
    - check: "backlog.md 是否包含 P0/P1/P2 三级分类"
    - check: "mvp-definition.md 是否包含 scope + success_criteria"
  reasoning:
    - check: "MVP范围是否与用户意图一致（orchestrator 验证）"
```

---

*Backlog Manager Harness v2.1 (Ironforge P1) — 2026-04-24*
