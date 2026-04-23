# 审计日志协议（Audit Protocol）

> Ironforge 支柱A 核心组件。
> v3.1 新增。所有操作可追溯，满足合规审计和故障定位需求。
>
> 路径变量和操作映射见 `pm-core/platform-adapter.md`。

---

## 1. 设计哲学

```
v3 状态：操作无结构化记录 → 出了问题无法追溯
v3.1 状态：每个文件操作、命令执行、权限变更都有审计记录 → 完整操作链可回溯
```

**核心原则**：操作可追溯。每个文件操作、命令执行、权限变更必须有审计记录。

---

## 2. 日志格式

```yaml
audit_log:
  # ═══════════════════════════════════════
  # 存储配置
  # ═══════════════════════════════════════
  path: "{context_root}/audit/{YYYY-MM-DD}.jsonl"
  format: "JSONL（每行一条JSON记录）"
  rotation: "30天滚动删除"
  immutable: true                         # 追加写入，不允许修改历史条目
  encoding: "UTF-8"

  # ═══════════════════════════════════════
  # 条目结构
  # ═══════════════════════════════════════
  entry_schema:
    # --- 必填字段 ---
    timestamp: "ISO8601"                  # 2026-04-23T14:30:00.123Z
    trace_id: "string"                    # proj-20260423-a3f7
    span_id: "string"                     # proj-20260423-a3f7-coder-T001
    agent: "string"                       # pm-coder
    task_id: "string"                     # T001
    event_type: "enum"                    # 见下表
    
    # --- 可选字段 ---
    detail: "object"                      # 事件详情（类型特定）
    risk_level: "enum|null"              # green|yellow|red|prohibited|null
    file_path: "string|null"             # 涉及的文件路径
    duration_ms: "number|null"           # 操作耗时
```

---

## 3. 必须审计的事件

### 3.1 Agent 生命周期事件

```yaml
lifecycle_events:
  - event_type: agent_spawn
    detail:
      agent: "string"
      task_id: "string"
      parent_agent: "string"
      mode: "string"                      # plan|acceptEdits|default
      max_turns: "number"
    risk_level: null
    
  - event_type: agent_shutdown
    detail:
      agent: "string"
      task_id: "string"
      reason: "string"                    # completed|failed|handoff|timeout|killed
      final_status: "string"
      turns_used: "number"
    risk_level: null
```

### 3.2 文件操作事件

```yaml
file_events:
  - event_type: file_read
    detail:
      path: "string"
      size: "number|null"
    risk_level: null                      # 读操作无风险
    
  - event_type: file_create
    detail:
      path: "string"
      size: "number"
      content_hash: "string"              # SHA256 前8位
    risk_level: "green"
    
  - event_type: file_modify
    detail:
      path: "string"
      diff_lines_added: "number"
      diff_lines_removed: "number"
      content_hash: "string"              # 修改后的 SHA256 前8位
    risk_level: "yellow"
    
  - event_type: file_delete
    detail:
      path: "string"
      was_backed_up: "boolean"
      backup_path: "string|null"
    risk_level: "red"
```

### 3.3 命令执行事件

```yaml
command_events:
  - event_type: command_execute
    detail:
      command: "string"
      exit_code: "number"
      duration_ms: "number"
      output_summary: "string"            # 截断到200字符
    risk_level: "根据白名单动态判定"        # 白名单内=green, 其他=yellow/red
```

### 3.4 权限事件

```yaml
permission_events:
  - event_type: approval_request
    detail:
      agent: "string"
      operation: "string"
      risk_level: "string"                # yellow|red
      reason: "string"
    risk_level: null
    
  - event_type: approval_result
    detail:
      approver: "string"
      approved: "boolean"
      reason: "string"
    risk_level: null
    
  - event_type: permission_violation
    detail:
      agent: "string"
      attempted_operation: "string"
      blocked_by: "string"                # Layer1|Layer2|Layer3
      risk_level_of_operation: "string"
    risk_level: "red"
```

### 3.5 安全事件

```yaml
security_events:
  - event_type: security_scan_hit
    detail:
      agent: "string"
      pattern_id: "string"                # PAT-001
      pattern_name: "string"              # API Key
      severity: "string"                  # CRITICAL|HIGH|MEDIUM
      file: "string"
      remediation: "string"
    risk_level: "critical"
    
  - event_type: security_violation
    detail:
      agent: "string"
      violation_type: "string"            # secrets_leak|protected_path|network_policy
      detail: "string"
      action_taken: "string"              # blocked|remediated|reported
    risk_level: "critical"
    
  - event_type: protected_path_access_denied
    detail:
      agent: "string"
      path: "string"
      access_type: "string"               # read|write
      rule: "string"                      # never_read|read_only
    risk_level: "red"
```

### 3.6 协作事件

```yaml
collaboration_events:
  - event_type: handoff
    detail:
      from_agent: "string"
      to_agent: "string"
      task_id: "string"
      reason: "string"
      progress_pct: "number"
    risk_level: null
    
  - event_type: message_sent
    detail:
      from_agent: "string"
      to_agent: "string"
      event_type: "string"
      summary: "string"
    risk_level: null
```

### 3.7 稳定性事件

```yaml
stability_events:
  - event_type: checkpoint_created
    detail:
      agent: "string"
      task_id: "string"
      step: "number"
      files_affected: "list"
    risk_level: null
    
  - event_type: rollback_executed
    detail:
      agent: "string"
      task_id: "string"
      from_step: "number"
      to_step: "number"
      reason: "string"
    risk_level: "yellow"
    
  - event_type: circuit_breaker_triggered
    detail:
      scope: "string"                     # agent|task|project
      target: "string"
      reason: "string"
    risk_level: "red"
```

---

## 4. 写入规范

```yaml
write_rules:
  # ═══════════════════════════════════════
  # 追加写入
  # ═══════════════════════════════════════
  append_only: |
    审计日志只允许追加写入，不允许修改或删除历史条目。
    实施方式：每次事件发生时，追加一行 JSON 到当天的 .jsonl 文件。
    如果文件不存在 → 创建新文件。
    
  # ═══════════════════════════════════════
  # 写入时机
  # ═══════════════════════════════════════
  write_timing:
    synchronous:                          # 同步写入（高优先级事件）
      - security_scan_hit
      - security_violation
      - permission_violation
      - protected_path_access_denied
      - circuit_breaker_triggered
      
    batched:                              # 批量写入（常规事件）
      - file_read
      - file_create
      - file_modify
      - message_sent
      batch_size: 10                      # 每10条批量写入
      max_delay: "30s"                    # 最长30秒必须写入
      
    end_of_step:                          # 步骤结束时写入
      - command_execute
      - checkpoint_created
```

---

## 5. 查询接口

```yaml
query:
  by_time_range:
    description: "按时间范围查询"
    parameters: [start_time, end_time]
    
  by_agent:
    description: "按Agent类型查询"
    parameters: [agent_name]
    
  by_risk_level:
    description: "按风险等级查询"
    parameters: [risk_level]              # green|yellow|red|critical
    
  by_trace_id:
    description: "按 trace_id 追踪完整调用链"
    parameters: [trace_id]
    use_case: "故障定位 — 追踪一个请求从 orchestrator 到各子Agent的完整路径"
    
  by_event_type:
    description: "按事件类型查询"
    parameters: [event_type]
    
  by_file_path:
    description: "按文件路径查询（谁在什么时候操作了这个文件）"
    parameters: [file_path]
    use_case: "文件变更审计"
```

---

## 6. 日志清理

```yaml
cleanup:
  retention: "30天"
  action: |
    每次项目启动时检查 audit/ 目录下的文件：
    - 超过30天的 .jsonl 文件 → 删除
    - 安全事件（security_*）的日志 → 保留90天
    - 权限事件（permission_violation）的日志 → 保留90天
    
  archive:
    trigger: "手动执行"
    action: "将旧日志压缩为 .jsonl.gz，移动到 {context_root}/audit/archive/"
```

---

*Ironforge Audit Protocol v1.0 — 2026-04-23*
