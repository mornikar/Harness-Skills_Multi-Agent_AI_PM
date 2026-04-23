# 交接棒协议（Handoff Protocol）— Harness 公共层

> 从原 coder-harness 模块五抽取并泛化。
> 定义所有 Agent 共享的交接触发条件、流程和文档模板。

---

## 1. 触发条件

```yaml
handoff_triggers:
  automatic:
    - condition: "已用轮次 > max_turns × 70%"
      action: "启动交接流程"
    - condition: "上下文使用率 > 预警阈值（Layer 2 budget 耗尽）"
      action: "启动交接流程"
    - condition: "DS3 上下文保存默认技能触发"
      action: "与 HANDOFF 合并执行"
      
  manual:
    - condition: "orchestrator send_message 要求交接"
      action: "立即启动交接流程"
    - condition: "Agent感知到上下文质量下降"
      action: "主动发起交接"
```

---

## 2. 交接流程

```yaml
handoff_procedure:
  # Step 1: 停止新操作
  - step: "freeze"
    action: "不再启动新的操作，只完成当前进行中的单步操作"

  # Step 2: 压缩当前状态
  - step: "compress"
    action: "生成 HANDOFF.md"
    output_path: "{context_root}/context_pool/progress/T{XXX}-handoff.md"

  # Step 3: 更新 HEARTBEAT
  - step: "sync"
    action: |
      更新任务 HEARTBEAT：
      - 状态改为 handoff
      - 记录交接原因和 HANDOFF.md 路径
      - 压缩"遇到的问题"区域（只保留未解决的问题）

  # Step 4: 通知 orchestrator
  - step: "notify"
    action: |
      send_message(type="message", recipient="main",
        event_type="task_handoff",
        task_id: "T{XXX}",
        handoff_path: "T{XXX}-handoff.md",
        progress_pct: {当前进度},
        reason: {交接原因})
```

---

## 3. HANDOFF 文档模板

```markdown
# HANDOFF: T{XXX}

## 交接原因
{auto: 上下文预算耗尽 | manual: orchestrator 请求}

## 当前进度
### 已完成
- [x] {步骤1}（产出：{文件路径}）
- [x] {步骤2}（产出：{文件路径}）
### 进行中
- [ ] {步骤3} — 进度 {N}%
### 未开始
- [ ] {步骤4}
- [ ] {步骤5}

## 系统状态
- **plan.md 路径**: {路径}
- **HEARTBEAT 路径**: {路径}
- **当前执行阶段**: {explore/execute}
- **已用轮次**: {N}/{max_turns}

## 关键决策记录
| # | 决策 | 选择 | 原因 |
|---|------|------|------|
| 1 | {决策描述} | {选择A} | {原因} |

## 遗留问题
| # | 问题描述 | 尝试过的方案 | 状态 |
|---|---------|-------------|------|
| 1 | {问题} | {方案} | 待解决 |

## 下一步建议
1. {建议的下一步}
2. {注意事项}

## 恢复指引
1. 读取 T{XXX}-plan.md → 了解完整规划
2. 读取 T{XXX}-heartbeat.md → 了解任务状态
3. 从"进行中"步骤的 {N}% 处继续
4. 注意：{关键注意事项}
```

---

## 4. 恢复指引

新 Agent 实例接手时：

```yaml
recovery_for_successor:
  steps:
    - "读取 HANDOFF.md → 理解完整上下文"
    - "读取 plan.md → 理解执行规划"
    - "读取 HEARTBEAT → 确认当前状态"
    - "从 HANDOFF 的'进行中'步骤继续执行"
    - "不再重复已完成的步骤"
```

---

*Ironforge Base: Handoff Protocol v1.0 — 2026-04-23*
