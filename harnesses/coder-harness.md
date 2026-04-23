# Coder Harness 定义（v3.1 Ironforge 重构版）

> pm-coder 子Agent 的执行载体定义。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 coder 特化逻辑。
>
> 公共逻辑（权限、安全、审计、检查点、交接、上下文、可观测）全部在 base/ 层定义，
> 本文件只声明 coder 独有的配置和行为。
>
> 设计参考：Claude Code 的规划管制 + 上下文工程 + 风险分级权限 + 交接棒 + 事件驱动钩子；
> Guides/Sensors 双闭环 + 两阶段 Code Review + 系统性调试 SOP。

---

## 版本与继承

```yaml
harness_version: "2.0"
compatible_skill_version: ">=3.0"
compatible_framework: "Ironforge v1.0"

# ═══════════════════════════════════════
# 继承公共层
# ═══════════════════════════════════════
inherits:
  - base/permission-framework.md         # 三层权限执行 + spawn配置
  - base/security-hooks.md              # 敏感信息扫描 + 受保护路径
  - base/audit-logging.md               # 审计日志格式和事件
  - base/checkpoint-protocol.md         # 检查点自动保存和回滚
  - base/handoff-protocol.md            # 交接棒触发和流程
  - base/context-engineering.md         # 上下文分层和预算
  - base/observability-config.md        # HEARTBEAT和通信事件
```

---

## 基本配置

```yaml
name: pm-coder
harness_type: subagent
description: |
  编程执行子Agent。负责代码编写、调试、重构、测试。
  通过 task 工具以 Team Mode spawn，加入项目团队协作。
  内置"先规划后编码"管制、风险分级权限、自动质量钩子。
  v2 新增：开发必须对齐原型设计。

spawn_config:
  subagent_name: "code-explorer"         # 内置 subagent 类型（代码探索+编写）
  explore_phase:
    mode: "plan"
    max_turns: 10
  execute_phase:
    mode: "acceptEdits"
    max_turns: 40
  # name 和 team_name 由 pm-runner 动态填入
```

---

## 特化配置

```yaml
specialization:
  # ═══════════════════════════════════════
  # 权限覆盖
  # ═══════════════════════════════════════
  permission_override:
    execute_phase:
      blocked_tools: [delete_file]        # coder 不应删除文件
      yellow_tools: [replace_in_file]
      command_whitelist:
        - npm install
        - npm run build
        - npm run test
        - npm run lint
        - tsc --noEmit
        - python -m pytest
        - python -m black --check
        - eslint
        - prettier --check
        - pylint
        - mypy

  # ═══════════════════════════════════════
  # 安全钩子覆盖
  # ═══════════════════════════════════════
  security_override:
    protected_paths:
      read_only:
        add: ["src/**/*.test.*"]          # 测试文件只读（防误改）

  # ═══════════════════════════════════════
  # 审计覆盖
  # ═══════════════════════════════════════
  audit_override:
    additional_events:
      - event_type: code_review
        detail: {stage, result, findings}

  # ═══════════════════════════════════════
  # 检查点覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  checkpoint_override:
    frequency: "every_step"
    retention: "all"
    rollback_strategy: "git_first"
    additional_trigger: "每次 replace_in_file 后"

  # ═══════════════════════════════════════
  # 熔断覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  circuit_breaker_override:
    agent_level:
      failure_threshold: 3
      window: "30min"
      cooldown_base: "30min"
    task_level:
      retry_budget: 3
      backoff: "exponential"
    non_retryable_errors:
      - "syntax_error"                     # 语法错误不重试
      - "type_error"                       # 类型错误不重试
      - "import_error"                     # 导入错误不重试

  # ═══════════════════════════════════════
  # 幂等性覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  idempotency_override:
    replace_in_file:
      old_str_min_context_lines: 3         # old_str 至少包含3行上下文
    write_to_file:
      always_check_exists: true            # 始终检查文件是否已存在
    command_pre_check:
      npm_install: "node_modules/.package-lock.json 存在 → 跳过"
      npm_run_build: "无前置检查（重复无害）"
      npm_run_test: "无前置检查（重复无害）"

  # ═══════════════════════════════════════
  # 上下文预算覆盖
  # ═══════════════════════════════════════
  context_budget_override:
    layer_1_hot: "≤ 3000 tokens"
    layer_2_working: "≤ 15000 tokens"
    handoff_trigger: "70% 轮次使用"

  # ═══════════════════════════════════════
  # 通信事件覆盖
  # ═══════════════════════════════════════
  communication_override:
    additional_notify_events:
      - event: debug_blocked
        template: |
          【debug_blocked】T{XXX} | pm-coder | 同一bug 2轮未修复
```

---

## Coder 特有规范

### 1. 双闭环控制模型（Guides + Sensors）

```yaml
guides_sensors:
  # ─── Guides（前馈控制）───
  guides:
    - "角色身份 → SKILL.md Layer 1"
    - "目标约束 → Goal success_criteria"
    - "工作流程 → SKILL.md Steps"
    - "编码标准 → references/code-standards.md"
    - "审查协议 → references/code-review-protocol.md"
    - "调试 SOP → references/debugging-protocol.md"

  # ─── Sensors（反馈控制）───
  sensors:
    computational:                        # Coder 自执行（确定性，快，无幻觉）
      - "H1 plan_compliance — plan 范围检查"
      - "H2 file_ownership — 文件所有权检查"
      - "H3 syntax_check — 编译/语法检查"
      - "H4 heartbeat_sync — 产出物清单同步"
      - "H7 full_test_suite — 测试套件"
      - "H8 lint_check — 代码风格"
      - "H9 deliverable_integrity — 产出物完整性"
      trigger: "每次文件操作后 + 任务完成时"

    lightweight_self_check:               # 关键词匹配级
      - "关键词匹配检查"
      - "API 端点名称核对"
      - "文件路径验证"
      trigger: "任务完成时"

    reasoning:                            # orchestrator 验收时执行
      - "语义评估 — 对比 Goal success_criteria"
      - "Spec 深度对照"
      - "代码质量语义审查"
      executor: "orchestrator（三角验证）"
```

### 2. 规划优先管制（Planning-First Workflow）

```yaml
planning_first:
  phases:
    - phase: "explore"                    # Phase A: 探索（只读）
      mode: "plan"
      max_turns: 10
      steps:
        - "读取项目 HEARTBEAT"
        - "读取上游任务 HEARTBEAT"
        - "读取 context_pool/tech_stack.md"
        - "读取 context_pool/architecture.md"
        - "扫描现有代码结构"
        - "识别影响范围和风险点"
      output: "{context_root}/context_pool/progress/T{XXX}-plan.md"
      notification: "send_message(event_type='plan_ready')"

    - phase: "approval_gate"              # Phase B: 审批等待
      trigger: "orchestrator 审阅 plan.md"
      outcomes:
        approved: "进入 Phase C"
        rejected: "附带修改意见 → 回到 Phase A"
        needs_research: "委托 researcher → 重新规划"

    - phase: "execute"                    # Phase C: 编码
      mode: "acceptEdits"
      max_turns: 40
      guardrails:
        - "严格按 plan.md 执行"
        - "超出 plan 范围 → 请求 orchestrator 批准"
        - "每完成一个里程碑 → 自动触发 post-step 钩子"
```

### 3. 事件驱动钩子（Coder 特有）

```yaml
hooks:
  # ─── 编码前钩子 ───
  pre_edit:
    - name: "plan_compliance_check"
      trigger: "每次 write_to_file 或 replace_in_file 前"
      check: "操作是否在 plan.md 范围内"

    - name: "file_ownership_check"
      trigger: "每次 replace_in_file 前"
      check: "文件是否被其他 Agent 占用"

  # ─── 编码后钩子 ───
  post_edit:
    - name: "syntax_check"
      trigger: "每次修改 .ts/.tsx/.py/.js/.vue 文件后"
      commands: ["tsc --noEmit {file}", "python -m py_compile {file}"]
      on_failure: "分析错误 → 修复 → 重试（最多3次）"

    - name: "heartbeat_sync"
      trigger: "每次 write_to_file / replace_in_file 成功后"
      action: "更新 HEARTBEAT 产出物清单"

  # ─── 步骤完成钩子 ───
  post_step:
    - name: "milestone_quality_gate"
      trigger: "完成 plan.md 中的每个里程碑步骤后"
      check: "执行对应的验证命令"

    - name: "progress_report"
      trigger: "每完成一个 plan step 后"
      action: "send_message(event_type='task_progress')"

  # ─── 任务完成钩子 ───
  on_complete:
    - name: "full_test_suite"
      trigger: "准备发送 task_complete 前"
      commands: ["npm run test", "python -m pytest"]
      on_failure: "修复代码 → 重试（最多2次） → 仍失败则 task_partial_success"

    - name: "lint_check"
      trigger: "准备发送 task_complete 前"
      commands: ["npm run lint", "pylint {src}"]
      on_failure: "自动修复可修复的 → 无法修复标 warning"

    - name: "deliverable_integrity"
      trigger: "准备发送 task_complete 前"
      check: "plan.md 文件是否都创建 + HEARTBEAT 产出物一致"

    - name: "code_review_stage1"
      type: "lightweight_self_check"
      trigger: "Stage 2 之前"
      check: "关键词匹配 + 文件完整性"

    - name: "code_review_stage2"
      type: "computational_sensor"
      trigger: "计算型传感器全部通过后"
      check: "H7/H8/H9 Hooks"

  # ─── 调试模式钩子 ───
  on_debug:
    - name: "debug_phase_gate"
      trigger: "调试类任务启动时"
      check: "强制使用四阶段调试 SOP"

    - name: "debug_stop_loss"
      trigger: "同一 bug 2轮未修复"
      action: "冻结 → send_message(task_blocked) → 建议 researcher 或人工介入"
```

### 4. Skill 加载策略

```yaml
skill_loading:
  tier_1_always:
    - pm-coder/SKILL.md

  tier_2_on_task:
    - Domain Skills（vue3, electron, fastapi 等）

  tier_3_on_demand:
    - pm-coder/references/heartbeat-ops.md
    - pm-coder/references/code-standards.md
    - pm-coder/references/acceptance-criteria.md
    - pm-coder/references/handoff-protocol.md
    - pm-coder/references/hooks-specification.md
    - pm-coder/references/code-review-protocol.md
    - pm-coder/references/debugging-protocol.md
    - pm-core/references/recovery-recipes.md
```

### 5. Prompt 注入（三层洋葱 + 意图解析层）

```yaml
prompt_layers:
  layer_0_intent:
    - "当前执行阶段: {explore|execute}"
    - "审批状态: {pending|approved|rejected}"
    - "如为 execute → 附 plan.md 路径"

  layer_1_identity:
    - "角色身份 + 核心职责"
    - "Goal success_criteria + constraints"
    - "风险分级权限表"
    - "可用 Skills Catalog"

  layer_2_narrative:
    - "项目 HEARTBEAT 摘要"
    - "上游产出"
    - "已完成阶段"

  layer_3_focus:
    - "当前任务描述 + ID + 类型"
    - "输出要求（格式、位置、验收标准）"
    - "上下文预算提醒"
    - "记忆要求（HEARTBEAT操作节奏）"
    - "三角验证流程"
    - "可加载参考资料路径"
```

### 6. 适用任务类型

| 任务类型 | 典型场景 | 额外 Skills | 预估轮次 |
|---------|---------|-----------|---------|
| frontend | Vue3/React页面开发 | vue3, react, electron | 50 |
| backend | API/服务端开发 | fastapi, express, prisma | 50 |
| database | 数据库设计与实现 | sql, mongodb | 30 |
| testing | 单元/集成测试 | jest, pytest | 30 |
| debugging | Bug修复 | 视具体项目 | 30 |
| refactoring | 代码重构 | 视具体项目 | 40 |

---

*Coder Harness v2.1 (Ironforge P1) — 2026-04-24*
