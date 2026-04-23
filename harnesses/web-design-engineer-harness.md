# Web Design Engineer Harness 定义（v3.1 Ironforge）

> web-design-engineer 视觉设计工程 Skill 的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 web-design-engineer 特化逻辑。

---

## 版本与继承

```yaml
harness_version: "2.0"
compatible_skill_version: ">=3.1"
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
name: web-design-engineer
role: "视觉设计工程专家"
phase: "Phase 3（编码阶段，被 pm-coder 委托调用）或独立运行"
mode: acceptEdits
max_turns: 30
context_budget: 80000
```

---

## 权限覆盖

```yaml
permissions:
  file_operations:
    read: ["**/*.{html,css,js,jsx,ts,tsx,json,md,svg,png,jpg}"]
    write: ["**/*.{html,css,js,jsx,md}"]
    create: true
    delete: false           # 不删除文件，只创建和修改
  commands:
    allowed: ["npm", "node", "python", "git status", "git diff"]
    blocked: ["rm", "del", "format", "shutdown"]
  external_access:
    web_fetch: true         # 需要获取 CDN 资源和字体
    api_calls: false
```

---

## 审计事件覆盖

```yaml
audit_events:
  - event: "design_system_declared"
    level: "info"
    description: "设计系统声明完成"
  - event: "v0_draft_shown"
    level: "info"
    description: "v0 草稿展示给用户"
  - event: "visual_artifact_delivered"
    level: "info"
    description: "视觉产物交付"
  - event: "design_token_exported"
    level: "info"
    description: "设计 Token 导出给 pm-coder"
```

---

## P1 稳定性覆盖

```yaml
stability:
  circuit_breaker:
    failure_threshold: 3        # 连续3次失败触发熔断
    reset_timeout: 1800         # 30分钟后试探恢复
    monitoring_window: 1800     # 30分钟监控窗口
  checkpoint:
    frequency: "verification_only"  # 只在验证点保存检查点（视觉设计不涉及频繁文件修改）
    max_checkpoints: 5
  idempotency:
    file_strategy: "skip_if_identical"   # 文件内容相同则跳过
    command_strategy: "pre_check"        # 执行前检查
```

---

## 上下文预算覆盖

```yaml
context_allocation:
  skill_definition: 15000     # SKILL.md + references 较大
  task_context: 30000         # 设计上下文、参考资源
  working_memory: 20000       # 设计 Token、组件规范
  output_buffer: 15000        # 代码输出
```

---

## 交接协议覆盖

```yaml
handoff:
  input_from:
    - agent: "pm-coder"
      required: ["component-tree", "requirements"]
      optional: ["brand-guidelines", "screenshots"]
  output_to:
    - agent: "pm-coder"
      delivers: ["design-tokens.md", "component-visual-spec.md"]
    - agent: "user"
      delivers: ["HTML 产物", "设计系统声明"]
```

---

## 来源

本 Harness 基于 [ConardLi/web-design-skill](https://github.com/ConardLi/web-design-skill)（MIT License）适配为 v3.1 Ironforge 架构格式。
