# 熔断机制协议

> **版本**：v3.1 Ironforge
> **定位**：pm-core/stability 层 — 故障隔离与自动降级
> **原则**：快速失败，优雅降级，不拖垮全局

---

## 一、核心哲学

```
当前状态                              目标状态
──────────                            ──────────
Agent 失败 → 无限重试                  Agent 失败 → 熔断 → 试探 → 恢复
一个Agent卡死 → 阻塞整个项目           一个Agent熔断 → 其他Agent继续
任务失败3次 → 还是继续                 任务重试预算耗尽 → 上报人工决策
项目陷入死循环 → 无人知道              项目级异常 → 紧急暂停 + 告警
```

**三条铁律**：
1. **快速失败**：检测到异常立即停止，不浪费时间在注定失败的操作上
2. **优雅降级**：熔断不是终结，而是降级等待恢复的机会
3. **故障隔离**：一个Agent的故障不应拖垮整个项目

---

## 二、三级熔断模型

### 2.1 Agent 级熔断

| 属性 | 说明 |
|------|------|
| **作用域** | 同一 project 内的同一 agent 类型 |
| **失败阈值** | 连续 3 次失败触发熔断 |
| **滑动窗口** | 30 分钟 |
| **目的** | 防止持续向故障Agent派发任务 |

#### 状态机

```
         成功
    ┌────────────┐
    │            │
    ▼            │
┌────────┐   成功  ┌──────────┐
│ CLOSED │◄───────│ HALF_OPEN│
│ (正常)  │        │ (试探)    │
└───┬────┘        └────┬─────┘
    │                  │
    │ 连续3次失败       │ 试探失败
    │                  │
    ▼                  ▼
┌──────────┐     cooldown加倍
│   OPEN   │─────────────────► OPEN
│  (熔断)   │
└──────────┘
    │
    │ cooldown: 30min
    │（首次）/ 60min（二次）
    ▼
┌──────────┐
│ HALF_OPEN│ ──试探──► 成功 → CLOSED
│ (试探)    │          失败 → OPEN（cooldown加倍）
└──────────┘
```

#### CLOSED → OPEN 触发条件

```yaml
trigger:
  condition: "30分钟滑动窗口内，同一Agent类型连续3次失败"
  failure_types:
    - task_failed: "任务执行失败"
    - timeout: "任务超时（超过 max_turns）"
    - crash: "Agent 崩溃（无法响应心跳）"
    - checkpoint_corruption: "检查点数据损坏"
```

#### OPEN 状态行为

```yaml
open_state:
  action:
    1: "停止向该类型Agent派发新任务"
    2: "send_message 通知 orchestrator（event: circuit_breaker_open）"
    3: "写入审计日志 circuit_breaker_open 事件"
    4: "等待 cooldown 时间后进入 HALF_OPEN"
  
  cooldown:
    first: "30min"
    second: "60min"           # 加倍
    subsequent: "120min"      # 再加倍，上限2h
    max: "120min"
  
  # 已排队的任务处理
  queued_tasks:
    action: "转派给其他可用Agent，或等待恢复"
    fallback: "上报 orchestrator 人工决策"
```

#### HALF_OPEN 试探

```yaml
half_open_state:
  action: "派发1个低风险任务（如只读操作）"
  evaluation:
    success: "→ CLOSED（恢复正常）"
    failure: "→ OPEN（继续熔断，cooldown加倍）"
  
  # 试探任务选择标准
  probe_task_criteria:
    risk_level: "low"          # 低风险
    type: "read_only"          # 只读操作
    max_turns: 5               # 短任务
    timeout: "5min"            # 快速判定
```

### 2.2 任务级熔断

| 属性 | 说明 |
|------|------|
| **作用域** | 单个任务 |
| **重试预算** | 3 次 |
| **退避策略** | 指数退避 |
| **目的** | 防止无限重试浪费资源 |

#### 重试策略

```yaml
retry:
  budget: 3                           # 同一任务最多重试3次
  backoff: "exponential"              # 指数退避
  
  schedule:
    attempt_1: "立即重试"
    attempt_2: "等待 1min"
    attempt_3: "等待 2min"
    
  # 重试前操作
  pre_retry:
    1: "回滚到最近的安全检查点"
    2: "分析失败原因（如果是可恢复错误）"
    3: "调整重试策略（如降低并发、简化操作）"
    
  # 重试条件判断
  retryable_errors:
    - "timeout"                        # 超时 → 可重试
    - "transient_error"               # 临时错误 → 可重试
    - "resource_contention"           # 资源争用 → 可重试
    
  non_retryable_errors:
    - "permission_denied"             # 权限拒绝 → 不重试
    - "invalid_input"                 # 输入无效 → 不重试
    - "logic_error"                   # 逻辑错误 → 不重试（需修改方案）
```

#### 预算耗尽处理

```yaml
exhaust_action:
  1: "写入审计日志 task_exhausted"
  2: "send_message 上报 orchestrator（event: task_exhausted）"
  3: "标记任务 status: failed"
  4: "不再自动重试，等待人工决策"
  
  # orchestrator 可选决策
  orchestrator_options:
    - "重新拆解任务（更小粒度）"
    - "换一个Agent类型执行"
    - "降级交付（部分完成）"
    - "标记为 blocked，等待用户决策"
```

### 2.3 项目级熔断

| 属性 | 说明 |
|------|------|
| **作用域** | 整个项目 |
| **触发条件** | 总轮次 > 80% 且 完成率 < 30% |
| **严重度** | CRITICAL |
| **目的** | 防止项目陷入死循环 |

#### 触发条件

```yaml
project_circuit_breaker:
  trigger:
    condition: "项目总轮次 > 80% 且 完成率 < 30%"
    evaluation_interval: "每5轮检查一次"
    
  additional_signals:
    - "连续5个任务失败"
    - "3个以上Agent同时处于OPEN状态"
    - "项目运行时间超过预期的2倍"
    
  action:
    1: "全局暂停所有Agent"
    2: "send_message 紧急通知 orchestrator（event: project_circuit_breaker）"
    3: "写入审计日志 project_circuit_breaker 事件"
    4: "建议：重新评估项目范围或调整任务拆解"
    
  severity: "CRITICAL"
  
  # 恢复条件
  recovery:
    requires: "人工确认"
    options:
      - "缩减项目范围（MVP再裁剪）"
      - "重新拆解任务（更细粒度）"
      - "更换技术方案"
      - "终止项目"
```

---

## 三、熔断与检查点的联动

```yaml
integration:
  # 检查点验证失败 → 熔断
  checkpoint_failure_to_breaker:
    condition: "连续2次检查点验证失败"
    action: "触发 Agent 级熔断"
    
  # 回滚耗尽 → 任务级熔断
  rollback_exhaust_to_breaker:
    condition: "同一任务回滚2次"
    action: "触发 task_level exhaust_action"
    
  # 熔断恢复后 → 从检查点重启
  breaker_recovery_to_checkpoint:
    condition: "HALF_OPEN 试探成功 → CLOSED"
    action: "从最近 verification_passed 检查点重新开始"
```

---

## 四、熔断与审计日志的联动

| 事件类型 | 触发时机 | 详情字段 |
|---------|---------|---------|
| `circuit_breaker_open` | Agent级熔断触发 | `{agent, failure_count, window, cooldown}` |
| `circuit_breaker_half_open` | 进入试探状态 | `{agent, probe_task_id}` |
| `circuit_breaker_closed` | 恢复正常 | `{agent, probe_result, downtime_duration}` |
| `task_retry` | 任务重试 | `{task_id, attempt, backoff_seconds, reason}` |
| `task_exhausted` | 重试预算耗尽 | `{task_id, total_attempts, final_error}` |
| `project_circuit_breaker` | 项目级熔断 | `{total_turns, completion_rate, failed_agents}` |

---

## 五、各 Agent 熔断特化

| Agent | 失败阈值 | 滑动窗口 | 试探策略 | 特殊说明 |
|-------|---------|---------|---------|---------|
| coder | 3次 | 30min | 派发只读探索任务 | 编码失败多因需求不清 |
| runner | 2次 | 20min | 派发轻量调度任务 | 调度失败影响全局 |
| analyst | 3次 | 30min | 派发简单澄清任务 | 分析失败可降级 |
| planner | 2次 | 20min | 派发小任务拆解 | 拆解失败影响下游 |
| designer | 3次 | 30min | 派发简单组件设计 | 设计失败可跳过 |
| researcher | 3次 | 45min | 派发简单查询任务 | 调研失败常因外部 |
| writer | 3次 | 30min | 派发简单文档片段 | 写作失败可降级 |
| orchestrator | 1次 | 15min | 人工介入 | 编排失败最严重 |
| backlog-manager | 3次 | 30min | 派发简单排序任务 | 排序失败可降级 |

---

## 六、配置模板（Harness 使用）

```yaml
# Harness 特化层可覆盖默认熔断配置
specialization:
  circuit_breaker_override:
    # Agent级覆盖
    agent_level:
      failure_threshold: 3            # 覆盖默认值
      window: "30min"                 # 覆盖默认窗口
      cooldown_base: "30min"          # 覆盖默认cooldown
      
    # 任务级覆盖
    task_level:
      retry_budget: 3                 # 覆盖重试预算
      backoff: "exponential"
      
    # 自定义不可重试错误
    non_retryable_errors:
      - "syntax_error"                # coder: 语法错误不重试
      - "type_error"                  # coder: 类型错误不重试
```

---

*v3.1 Ironforge — circuit-breaker.md — 2026-04-24*
