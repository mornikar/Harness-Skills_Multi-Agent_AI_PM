# Researcher Harness 定义（v3.1 Ironforge 重构版）

> pm-researcher 信息检索与分析子Agent的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 researcher 特化逻辑。

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
name: pm-researcher
harness_type: subagent
description: |
  信息检索与分析子Agent。负责技术调研、竞品分析、方案选型。

spawn_config:
  subagent_name: "code-explorer"
  explore_phase:
    mode: "plan"
    max_turns: 10
  execute_phase:
    mode: "acceptEdits"
    max_turns: 30
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    execute_phase:
      green_add: [web_search, web_fetch]  # 调研核心工具
      blocked_tools: [execute_command, delete_file]
      max_turns: 40

  context_budget_override:
    layer_1_hot: "≤ 2000 tokens"
    layer_2_working: "≤ 10000 tokens"
    handoff_trigger: "75% 轮次使用"

  audit_override:
    additional_events:
      - event_type: research_completed
        detail: {dimensions_covered, sources_count, recommendation}

  # v3.1 P1 稳定性覆盖
  checkpoint_override:
    frequency: "verification_only"
    retention: "verification_only"
    rollback_strategy: "reverse_only"

  circuit_breaker_override:
    agent_level:
      failure_threshold: 3
      window: "45min"                     # 调研失败常因外部，窗口更长
    task_level:
      retry_budget: 3

  idempotency_override:
    write_to_file:
      always_check_exists: true
```

---

## Researcher 特有规范

### 调研质量钩子

```yaml
hooks:
  post_step:
    - name: "source_freshness_check"
      trigger: "每完成一个调研维度后"
      check: "标注每条关键信息的来源时效（🟢/🟡/🔴/⚪）"

    - name: "dimension_coverage_check"
      trigger: "每个调研维度完成后"
      check: "对照 plan.md 调研维度清单，确认覆盖"

    - name: "claim_verification"
      trigger: "报告中提出重要结论后"
      check: "重要结论必须有证据支撑"
      on_failure: "标注⚠️结论待验证"

  on_complete:
    - name: "report_completeness_gate"
      trigger: "生成报告后、发送 task_complete 前"
      check: "推荐结论 + 风险评估 + 下游影响 + 维度覆盖"

    - name: "terminology_consistency_check"
      trigger: "报告完成后"
      check: "同一概念统一术语"

    - name: "citation_completeness"
      trigger: "报告完成后"
      check: "来源引用完整 + URL有效 + 时间标注"
```

### 适用任务类型

| 任务类型 | 典型场景 | 额外 Skills | 预估轮次 |
|---------|---------|-----------|---------|
| tech_research | 技术框架调研选型 | web-search, github | 40 |
| competitive | 竞品分析 | web-search | 30 |
| feasibility | 可行性评估 | web-search | 30 |
| api_research | API文档查阅 | - | 25 |
| best_practice | 最佳实践收集 | web-search | 30 |

---

*Researcher Harness v2.1 (Ironforge P1) — 2026-04-24*
