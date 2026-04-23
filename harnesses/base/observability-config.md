# 可观测性配置（Observability Config）— Harness 公共层

> 从 pm-core/observability/ 三个协议衍生。
> 定义所有 Agent 共享的 HEARTBEAT v2 格式、通信事件、指标收集触发和追踪传播规范。
> v3.1 Ironforge 对齐版本。

---

## 1. HEARTBEAT v2 格式

```yaml
heartbeat:
  path: "{context_root}/context_pool/progress/T{XXX}-heartbeat.md"
  project_heartbeat: "{context_root}/HEARTBEAT.md"
  
  update_frequency: "每次文件操作后 + 每个步骤完成后"
  
  # v2 格式：YAML front matter（机器可解析）+ Markdown body（人类可读）
  format: |
    ---
    # 结构化元数据（机器可解析）
    heartbeat_version: "2.0"
    trace_id: "{trace_id}"
    span_id: "{span_id}"
    task_id: "T{XXX}"
    agent: "{agent_name}"
    status: "running"                  # pending|running|blocked|completed|failed|handoff
    progress: 60                       # 0-100
    health_score: 82                   # 0-100
    context_level: "FULL"              # MINIMAL|PARTIAL|FULL
    current_phase: "execute"           # explore|approval_gate|execute
    started_at: "2026-04-23T09:00:00Z"
    updated_at: "2026-04-23T14:30:00Z"
    checkpoint_step: 3
    error_count: 0
    retry_count: 0
    risk_level: "low"                  # low|medium|high|critical
    turn_usage: "18/50"
    ---
    
    # 任务进度（人类可读）
    ## 已完成步骤
    - [x] Step 1: ...
    - [x] Step 2: ...
    
    ## 进行中
    - [ ] Step 3: ...（60%）
    
    ## 产出物
    - {file_path} ({size})
    
    ## 问题与风险
    - 无

  # 健康度计算公式
  health_score_formula: |
    base = 100
    deductions = error_count*5 + retry_count*3
    if turn_usage_percent > 70: deductions += 10
    if stale_minutes > 15: deductions += 15
    health_score = max(0, base - deductions)
```

---

## 2. 追踪传播（v3.1 P2 新增）

```yaml
tracing:
  # trace_id 生成
  trace_id:
    format: "proj-{YYYYMMDD}-{random4}"
    generated_by: "orchestrator（编排启动时）"
    
  # span_id 生成
  span_id:
    format: "{trace_id}-{agent}-{task_id}"
    generated_by: "每个Agent启动时"
    
  # 传播渠道
  propagation:
    - channel: "HEARTBEAT"
      fields: ["trace_id", "span_id"]
    - channel: "send_message"
      fields: ["trace_id", "span_id"]
    - channel: "audit_log"
      fields: ["trace_id", "span_id"]
    - channel: "metrics"
      fields: ["trace_id", "span_id"]
    - channel: "checkpoint"
      fields: ["trace_id", "span_id"]
    - channel: "HANDOFF"
      fields: ["trace_id", "span_id"]
```

---

## 3. 指标收集触发（v3.1 P2 新增）

```yaml
metrics_collection:
  storage: "{context_root}/metrics/{YYYY-MM-DD}.jsonl"
  
  # 自动触发的指标
  auto_collected:
    - trigger: "Agent spawn"
      metrics: ["agent_spawn_total", "active_agents"]
      collector: "runner"
      
    - trigger: "Agent shutdown"
      metrics: ["task_complete_total", "active_agents"]
      collector: "runner"
      
    - trigger: "文件操作"
      metrics: ["file_operation_total"]
      collector: "各Agent"
      
    - trigger: "命令执行"
      metrics: ["command_execute_total"]
      collector: "各Agent"
      
    - trigger: "安全事件"
      metrics: ["security_event_total"]
      collector: "安全钩子"
      
    - trigger: "重试"
      metrics: ["retry_total"]
      collector: "熔断机制"
      
    - trigger: "回滚"
      metrics: ["rollback_total"]
      collector: "检查点协议"
      
    - trigger: "交接"
      metrics: ["handoff_total"]
      collector: "交接棒协议"
      
    - trigger: "熔断触发"
      metrics: ["circuit_breaker_total"]
      collector: "熔断机制"
```

---

## 4. 通信事件定义

```yaml
communication_events:
  # ─── 规划阶段事件 ───
  - event: plan_ready
    template: |
      【plan_ready】T{XXX} | {agent} | Phase A
      规划文档: {plan_path}
      请求审批

  - event: plan_approved
    template: |
      【plan_approved】T{XXX} | {agent} | Phase A→C
      开始执行

  # ─── 执行阶段事件 ───
  - event: task_progress
    template: |
      【task_progress】T{XXX} | {agent} | {progress_pct}%
      已完成: {completed_step}
      下一步: {next_step}

  - event: task_blocked
    template: |
      【task_blocked】T{XXX} | {agent}
      原因: {block_reason}
      需要: {needed_from}

  - event: task_partial_success
    template: |
      【task_partial_success】T{XXX} | {agent}
      完成: {completed_items}
      未完成: {incomplete_items}

  - event: task_complete
    template: |
      【task_complete】T{XXX} | {agent} | 100%
      产出物: {deliverable_paths}

  - event: task_failed
    template: |
      【task_failed】T{XXX} | {agent}
      原因: {failure_reason}
      建议: {suggestion}

  # ─── 权限事件 ───
  - event: risk_notification
    template: |
      【risk_notification】T{XXX} | {agent} | 🟡
      操作: {operation}
      原因: {reason}

  - event: risk_approval_request
    template: |
      【risk_approval_request】T{XXX} | {agent} | 🔴
      操作: {operation}
      影响: {impact}
      回滚: {rollback}

  - event: permission_violation
    template: |
      【permission_violation】T{XXX} | {agent} | 🚫
      尝试: {attempted_operation}

  # ─── 安全事件 ───
  - event: security_scan_hit
    template: |
      【security_scan_hit】T{XXX} | {agent} | ⚠️
      模式: {pattern_name} ({severity})
      文件: {file_path}
      修复: {remediation}

  # ─── 交接事件 ───
  - event: task_handoff
    template: |
      【task_handoff】T{XXX} | {agent}
      交接文档: {handoff_path}
      进度: {progress_pct}%
      原因: {handoff_reason}

  # ─── 健康度事件 ───
  - event: health_alert
    template: |
      【health_alert】T{XXX} | {agent} | 健康度 {health_score}
      因素: {factors}

  # ─── 熔断事件（v3.1 P1 新增）───
  - event: circuit_breaker_open
    template: |
      【circuit_breaker_open】| {agent} | 🔥
      失败次数: {failure_count}/{threshold}
      冷却时间: {cooldown}
      建议: {suggestion}

  # ─── 监控事件（v3.1 P2 新增）───
  - event: monitoring_alert
    template: |
      【monitoring_alert】T{XXX} | {agent} | {alert_type}
      条件: {condition}
      严重度: {severity}
```

---

## 5. 健康度评估

```yaml
health_evaluation:
  factors:
    - name: progress_velocity
      weight: 0.3
      description: "进度推进速度（实际进度 / 预估进度）"
      
    - name: heartbeat_freshness
      weight: 0.25
      description: "HEARTBEAT 更新及时性"
      
    - name: error_count
      weight: 0.2
      description: "错误数量（越少越好）"
      
    - name: constraint_compliance
      weight: 0.25
      description: "约束遵守程度"

  thresholds:
    healthy: 75                          # 75+ → 健康
    attention: 50                        # 50-74 → 关注
    warning: 30                          # 30-49 → 预警
    critical: 0                          # <30 → 严重，需人工介入
```

---

## 6. 监控规则（增强版）

```yaml
monitoring_rules:
  - name: "任务停滞"
    condition: "updated_at 超过15分钟未更新 且 status=running"
    severity: "warning"
    action: "send_message 健康检查请求 + health_score扣15分"
    escalation: "30分钟 → critical → 通知 orchestrator"
    
  - name: "健康度下降"
    condition: "health_score < 50"
    severity: "warning"
    action: "send_message 健康预警 + 评估交接或熔断"
    escalation: "health_score < 30 → critical → 触发熔断评估"
    
  - name: "轮次超支"
    condition: "turn_usage 使用率 > 70%"
    severity: "warning"
    action: "send_message 预算预警 + 启动交接评估"
    escalation: "使用率 > 90% → critical → 强制交接"
    
  - name: "错误聚集"
    condition: "error_count >= 3"
    severity: "warning"
    action: "send_message 错误聚集预警 + 启动熔断评估"
    escalation: "error_count >= 5 → critical → 触发Agent级熔断"
    
  - name: "阻塞超时"
    condition: "status=blocked 且 持续超过30分钟"
    severity: "warning"
    action: "send_message 阻塞预警"
    escalation: "超过60分钟 → critical → 通知 orchestrator"

  # 监控执行
  monitor:
    primary: "pm-runner"
    fallback: "pm-orchestrator"
    check_interval: "5分钟"
```

---

*Ironforge Base: Observability Config v2.0 — 2026-04-24*
