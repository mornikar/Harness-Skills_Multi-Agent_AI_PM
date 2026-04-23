# Writer Harness 定义（v3.1 Ironforge 重构版）

> pm-writer 内容输出子Agent的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 writer 特化逻辑。

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
name: pm-writer
harness_type: subagent
description: |
  内容输出子Agent。负责PRD、技术文档、用户手册、CHANGELOG等文档撰写。

spawn_config:
  subagent_name: "code-explorer"
  explore_phase:
    mode: "plan"
    max_turns: 10
  execute_phase:
    mode: "acceptEdits"
    max_turns: 25
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    execute_phase:
      blocked_tools: [delete_file, execute_command]
      max_turns: 35

  context_budget_override:
    layer_1_hot: "≤ 2000 tokens"
    layer_2_working: "≤ 12000 tokens"
    handoff_trigger: "75% 轮次使用"

  audit_override:
    additional_events:
      - event_type: document_created
        detail: {doc_type, word_count, chapters_completed}

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
```

---

## Writer 特有规范

### 一致性检查钩子

```yaml
hooks:
  post_step:
    - name: "upstream_consistency_check"
      trigger: "每完成一个章节后"
      check: "章节内容与上游产出物一致性"

    - name: "cross_reference_check"
      trigger: "章节之间有交叉引用时"
      check: "引用的章节/图片/表格编号正确"

    - name: "terminology_consistency"
      trigger: "每完成一个章节后"
      check: "同一概念使用统一术语"

  on_complete:
    - name: "template_completeness_gate"
      trigger: "文档完成后"
      check: "模板要求的章节/表格/示例齐全"

    - name: "glossary_check"
      trigger: "文档完成后"
      check: "首次出现的专业术语有解释"

    - name: "link_integrity_check"
      trigger: "文档完成后"
      check: "内部锚点 + 外部URL + 图片路径正确"

    - name: "final_consistency_audit"
      trigger: "文档完成后"
      check: "标题/摘要/结论/版本号与HEARTBEAT一致"
```

### 适用任务类型

| 任务类型 | 典型场景 | 额外 Skills | 预估轮次 |
|---------|---------|-----------|---------|
| prd | 产品需求文档 | docx, notion | 35 |
| tech_doc | 技术设计文档 | markdown, mermaid | 35 |
| api_doc | API接口文档 | openapi, swagger | 30 |
| user_manual | 用户手册 | docx, pdf | 30 |
| changelog | CHANGELOG | markdown | 20 |

---

*Writer Harness v2.1 (Ironforge P1) — 2026-04-24*
