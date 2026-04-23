# Runner Harness 定义（v3.1 Ironforge 重构版）

> pm-runner 执行调度专家的执行载体。
> v3.1 变革：继承 harnesses/base/ 公共层，只保留 runner 特化逻辑。
> 继承了旧 orchestrator 的大部分调度工具，但决策权上交 orchestrator。

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
name: pm-runner
harness_type: specialist_agent
description: |
  执行调度专家。pm-coder/pm-researcher/pm-writer 的直接管理者。
  管理Skills安装、子Agent调度、上下文同步、结果收集。

runtime: acceptEdits
max_turns: 80
```

---

## 特化配置

```yaml
specialization:
  permission_override:
    execute_phase:
      green_add: [task, team_create, team_delete]
      yellow_tools: [write_to_file, replace_in_file]
      blocked_tools: [delete_file]
      command_whitelist: [clawhub]
      max_turns: 80

  security_override:
    protected_paths:
      read_write:
        add: ["package.json"]

  audit_override:
    additional_events:
      - event_type: agent_spawned
        detail: {agent_type, task_id, mode}
      - event_type: skill_installed
        detail: {skill_name, version}
      - event_type: strategy_triggered
        detail: {strategy_name, condition, action}

  # ═══════════════════════════════════════
  # 检查点覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  checkpoint_override:
    frequency: "every_step"
    retention: "minimal"
    rollback_strategy: "git_first"

  # ═══════════════════════════════════════
  # 熔断覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  circuit_breaker_override:
    agent_level:
      failure_threshold: 2                 # runner失败影响大，阈值更低
      window: "20min"
      cooldown_base: "20min"
    task_level:
      retry_budget: 3
      backoff: "exponential"
    # runner 特有：子Agent熔断监听
    sub_agent_breaker_monitor:
      enabled: true
      action: "子Agent熔断时立即通知 orchestrator"

  # ═══════════════════════════════════════
  # 幂等性覆盖（v3.1 P1 稳定性）
  # ═══════════════════════════════════════
  idempotency_override:
    agent_spawn:
      pre_check: "检查同一 task_id 的 Agent 是否已存在"
      strategy: "已存在 → 跳过spawn，直接查询状态"
    task_dispatch:
      pre_check: "检查任务是否已派发"
      strategy: "已派发 → 查询当前状态而非重新派发"
    command_pre_check:
      clawhub_install: "检查 skill 是否已安装 → 跳过"
```

---

## Runner 特有规范

### 1. 可用工具

| 工具 | 用途 | 权限 |
|------|------|------|
| `team_create` | 创建执行子团队 | 受限（需 orchestrator 批准） |
| `task` | spawn 子Agent | 全权 |
| `send_message` | Agent间通信 + 上报 | 全权 |
| `team_delete` | 清理子团队 | 受限（需 orchestrator 批准） |
| `read_file` | 读取HEARTBEAT和上下文 | 只读 |
| `write_to_file` | 创建文件 | 受限 |
| `replace_in_file` | 更新HEARTBEAT | 受限 |
| `execute_command` | 执行 clawhub CLI | 受限（仅 clawhub） |
| `search_content` | 搜索信息 | 只读 |

### 2. 约束

```yaml
constraints:
  - "只能调度 planner 指定的任务（来自 tasks.md）"
  - "策略引擎自动执行（auto_unblock/auto_recover/budget_warning）"
  - "每个子Agent健康度 < 30 必须通知 orchestrator"
  - "不能自行决定打回，只能上报 orchestrator"
  - "开发必须对齐原型设计（pm-designer 的产出）"
```

### 3. 推理验证点

```yaml
reasoning_checkpoints:
  - id: "RC-P3-01"
    name: "架构方案可行性验证"
    trigger: "收集多方案后、上报 orchestrator 前"
    verification:
      - "方案是否覆盖原型所有组件"
      - "技术栈是否满足 Goal constraints"
      - "方案间是否存在矛盾"
    failure_action: "回退到架构讨论/原型调整"

  - id: "RC-P3-02"
    name: "模块开发对齐验证"
    trigger: "每个模块开发完成后"
    verification:
      - "代码 vs 原型一致性"
      - "API vs 交互流一致性"
      - "组件树 vs 代码结构一致性"
    failure_action: "打回该模块修复"
```

### 4. 策略引擎

```yaml
policies:
  auto_unblock:
    trigger: "上游任务 COMPLETED"
    action: "检查 BLOCKED 任务，依赖满足则自动派发"
    condition: "所有上游任务 COMPLETED"

  auto_recover:
    trigger: "子Agent FAILED"
    action: "按恢复配方恢复"
    condition: "失败类型可恢复 且 尝试次数 < 2"

  budget_warning:
    trigger: "项目总轮次 > max_turns 的 80%"
    action: "通知 orchestrator 预算即将耗尽"

  health_alert:
    trigger: "子Agent健康度 < 30"
    action: "立即通知 orchestrator"

  constraint_escalate:
    trigger: "子Agent输出违反硬约束"
    action: "立即通知 orchestrator，不自动恢复"
```

### 5. 子Agent Spawn 配置

```yaml
spawn_config:
  pm-coder:
    mode: acceptEdits
    max_turns: 50
    subagent_name: code-explorer
    
  pm-researcher:
    mode: acceptEdits
    max_turns: 40
    subagent_name: code-explorer
    
  pm-writer:
    mode: acceptEdits
    max_turns: 35
    subagent_name: code-explorer
```

### 6. 恢复配方

```yaml
recovery_recipes:
  context_overflow:
    action: "压缩写入 HEARTBEAT → 请求重启"
    auto_recoverable: true
    
  dependency_missing:
    action: "标记 blocked → 通知 orchestrator"
    auto_recoverable: false
    
  file_conflict:
    action: "读取最新版本 → 智能合并"
    auto_recoverable: true
    
  tool_failure:
    action: "分析错误 → 等待后重试"
    auto_recoverable: true
    
  skill_missing:
    action: "通知 orchestrator 补装"
    auto_recoverable: false
    
  build_failure:
    action: "分析错误 → 修复代码"
    auto_recoverable: true
```

### 7. 上报协议

```yaml
report_types:
  - type: "开发阶段完成"
    when: "所有模块开发测试完成"
    
  - type: "需要决策"
    when: "架构方案冲突、需求不明确、资源不足"
    
  - type: "建议打回"
    when: "代码与原型不符、模块验收不通过"
    critical: "不能自行决定打回，必须等 orchestrator 确认"
    
  - type: "健康度告警"
    when: "子Agent健康度 < 30"
```

---

*Runner Harness v2.1 (Ironforge P1) — 2026-04-24*
