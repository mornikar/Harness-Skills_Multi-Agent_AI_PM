# Analyst Harness 定义（v3.1 Ironforge 重构版）

> pm-analyst 需求澄清专家的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 analyst 特化逻辑。

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
name: pm-analyst
harness_type: specialist_agent
description: |
  需求澄清专家。专注于与用户深度沟通，澄清需求范围、约束条件、成功标准。

runtime: default
max_turns: 25
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    explore_phase_only: true
    mode: "default"
    allowed_extra: [write_to_file]        # 允许创建分析文档
    blocked_tools: [execute_command, replace_in_file, delete_file]
    max_turns: 25

  security_override: {}

  audit_override:
    additional_events:
      - event_type: goal_created
        detail: {goal_id, success_criteria_count, constraints_count}

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

## Analyst 特有规范

### 约束

```yaml
constraints:
  - "只能产出需求文档，不能产出代码"
  - "必须追问至少3轮才可结束澄清"
  - "Goal 对象必须包含至少3条 success_criteria"
  - "硬约束必须有明确的违规后果描述"
```

### 推理验证点

```yaml
reasoning_checkpoint:
  id: "RC-P1-01"
  name: "需求理解验证"
  trigger: "完成追问后、输出文档前"
  verification:
    - "输出需求理解摘要"
    - "检查 In Scope / Out Scope 是否与用户意图一致"
    - "Goal success_criteria 是否可度量"
  failure_action: "追加追问轮次"
```

### 协作奖励模型

```yaml
reward_alignment:
  high_score_behaviors:
    - "Goal 的 success_criteria 足够明确，planner 可直接消费"
    - "需求文档零歧义，下游无需追问"
    - "约束条件完整，开发阶段无需补充调研"
  low_score_behaviors:
    - "需求模糊导致 planner 无法拆解"
    - "遗漏约束导致开发阶段返工"
```

---

*Analyst Harness v2.1 (Ironforge P1) — 2026-04-24*
