# 审计日志（Audit Logging）— Harness 公共层

> 从 `pm-core/security/audit-protocol.md` 衍生。
> 定义所有 Agent 共享的审计日志写入规范。
> 各 Agent 特化层通过 `audit_override` 增加特有事件。

---

## 1. 日志配置

```yaml
audit_log:
  path: "{context_root}/audit/{YYYY-MM-DD}.jsonl"
  format: "JSONL"
  rotation: "30天滚动删除"
  immutable: true                         # 追加写入，不可修改历史条目
```

---

## 2. 条目格式

```json
{
  "timestamp": "2026-04-23T14:30:00.123Z",
  "trace_id": "proj-20260423-a3f7",
  "span_id": "proj-20260423-a3f7-coder-T001",
  "agent": "pm-coder",
  "task_id": "T001",
  "event_type": "file_modify",
  "detail": { "path": "src/App.vue", "diff_lines_added": 5, "diff_lines_removed": 2 },
  "risk_level": "yellow",
  "file_path": "src/App.vue",
  "duration_ms": null
}
```

---

## 3. 必须审计的事件（全 Agent 通用）

```yaml
mandatory_events:
  # 文件操作
  - event_type: file_create
    risk_level: green
  - event_type: file_modify
    risk_level: yellow
  - event_type: file_delete
    risk_level: red
    
  # 命令执行
  - event_type: command_execute
    risk_level: dynamic
    
  # 权限事件
  - event_type: approval_request
  - event_type: approval_result
  - event_type: permission_violation
    risk_level: red
    
  # 安全事件
  - event_type: security_scan_hit
    risk_level: critical
  - event_type: security_violation
    risk_level: critical
  - event_type: protected_path_access_denied
    risk_level: red
    
  # Agent 生命周期
  - event_type: agent_spawn
  - event_type: agent_shutdown
    
  # 协作
  - event_type: handoff
```

---

## 4. 写入时机

```yaml
write_timing:
  synchronous:                            # 立即写入
    - security_scan_hit
    - security_violation
    - permission_violation
    - protected_path_access_denied
    - circuit_breaker_triggered
    
  batched:                                # 批量写入
    - file_read
    - file_create
    - file_modify
    - message_sent
    batch_size: 10
    max_delay: "30s"
    
  end_of_step:                            # 步骤结束时
    - command_execute
    - checkpoint_created
```

---

*Ironforge Base: Audit Logging v1.0 — 2026-04-23*
