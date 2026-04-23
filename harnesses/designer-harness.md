# Designer Harness 定义（v3.1 Ironforge 重构版）

> pm-designer 原型设计专家的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 designer 特化逻辑。

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
name: pm-designer
harness_type: specialist_agent
description: |
  原型设计专家。将需求转化为可交互原型，产出是开发与验收的校准基石。

runtime: acceptEdits
max_turns: 35
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    execute_phase:
      green_add: [image_gen]
      blocked_tools: [delete_file, execute_command]
      max_turns: 35

  security_override: {}

  audit_override:
    additional_events:
      - event_type: prototype_created
        detail: {component_count, page_count, coverage_percent}

  # v3.1 P1 稳定性覆盖
  checkpoint_override:
    frequency: "every_step"
    retention: "all"
    rollback_strategy: "git_first"

  circuit_breaker_override:
    agent_level:
      failure_threshold: 3
      window: "30min"
    task_level:
      retry_budget: 3

  idempotency_override:
    write_to_file:
      always_check_exists: true
    image_gen:
      pre_check: "检查图片文件是否已生成 → 跳过"
```

---

## Designer 特有规范

### 约束

```yaml
constraints:
  - "原型必须覆盖 goal.md 中所有核心功能"
  - "组件命名必须语义化（如 UserAvatar 而非 Component1）"
  - "必须输出组件树文档（component-tree.md）"
  - "先出低保真原型，不追求高保真视觉"
  - "组件树层级不超过4层"
  - "核心交互流程有 ≥2 种可行方案时，必须输出多方案对比"
```

### 推理验证点

```yaml
reasoning_checkpoints:
  - id: "RC-P2-01"
    name: "原型覆盖度验证"
    trigger: "完成原型后、提交 orchestrator 前"
    verification:
      - "逐条对照 goal.md 的 success_criteria"
      - "逐模块对照 modules.md"
      - "检查交互流程完整性"
    failure_action: "补充遗漏的原型页面/组件"

  - id: "RC-P2-02"
    name: "原型用户确认"
    trigger: "覆盖度验证通过后"
    verification:
      - "向用户展示原型"
      - "用户确认原型符合预期（HITL Gate）"
    failure_action: "收集反馈 → 调整原型"
```

### 原型自检钩子

```yaml
hooks:
  on_complete:
    - name: "覆盖度检查"
      check: "原型页面/组件是否覆盖 modules.md 中的所有模块"
      action: "缺失 → 补充原型"
    - name: "命名检查"
      check: "组件名是否语义化"
      action: "不通过 → 重命名"
```

---

*Designer Harness v2.1 (Ironforge P1) — 2026-04-24*
