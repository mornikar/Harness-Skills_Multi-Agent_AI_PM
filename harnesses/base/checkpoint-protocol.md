# 检查点协议（Checkpoint Protocol）— Harness 公共层

> 从 `pm-core/stability/checkpoint-protocol.md` 衍生。
> 定义所有 Agent 共享的检查点自动保存和回滚规范。
> v3.1 Ironforge 对齐版本。

---

## 1. 检查点类型

### 1.1 自动检查点

```yaml
auto_checkpoint:
  trigger: "每个 plan step 完成后"
  storage: "{context_root}/checkpoints/T{XXX}.jsonl"
  
  format:
    step: "number"                         # 当前步骤编号
    timestamp: "ISO8601"
    status: "enum"                         # step_completed|verification_passed|verification_failed
    files_created: "list"
    files_modified: "list"
    file_hashes: "map"                     # {path: sha256:前16位hex}
    heartbeat_snapshot: "object"           # {status, progress, health_score}
    verification_result: "enum|null"      # passed|failed|null
```

### 1.2 验证点检查点

```yaml
verification_checkpoint:
  trigger: "推理验证点（RC-P3-02等）通过后"
  additional_fields:
    verification_id: "string"              # RC-P3-02
    verification_detail: "string"
  importance: "high"                       # 回滚优先使用验证点检查点
```

### 1.3 交接检查点（v3.1 新增）

```yaml
handoff_checkpoint:
  trigger: "Agent 交接时（HANDOFF 写入前）"
  importance: "critical"                   # 确保交接信息不丢失
  additional_fields:
    handoff_to: "string"                   # 接收Agent
    handoff_reason: "string"               # 交接原因
  guarantee: "新Agent可从交接检查点确定性恢复"
```

---

## 2. 回滚机制

```yaml
rollback:
  trigger: "验证点不通过 且 重试2次仍失败"
  
  # 安全保障
  safeguards:
    - "回滚前备份当前状态作为新检查点"
    - "回滚后验证文件hash与检查点一致"
    - "同一任务最多回滚2次，超过则上报 orchestrator"
  
  procedure:
    - step: "定位安全检查点"
      action: "找到最近的 status=verification_passed 的检查点"
      
    - step: "文件还原"
      action: |
        遍历当前检查点之后的所有检查点：
          - files_created → 删除这些文件
          - files_modified → 从检查点恢复hash对应的版本
        方法：
          优先: git checkout（如果项目在git管理下）
          降级: 逆向操作（删除新文件、还原修改内容）
          
    - step: "状态还原"
      action: "恢复 HEARTBEAT 到检查点的 snapshot"
      
    - step: "重新执行"
      action: "从检查点的下一步开始重新执行 plan（幂等保障）"
      
    - step: "审计记录"
      action: "写入审计日志 rollback 事件"
```

---

## 3. 检查点清理

```yaml
cleanup:
  trigger: "任务 COMPLETED 后"
  action: |
    保留：第一个 + 最后一个 + 所有 verification_passed
    删除：中间的 step_completed 检查点
  reason: "减少存储占用，保留关键恢复点"
  
  # 紧急清理
  emergency:
    trigger: "存储超过 100MB"
    action: "只保留最近3天的检查点"
```

---

## 4. 与熔断/幂等性联动（v3.1 新增）

```yaml
integration:
  # 检查点验证失败 → 熔断
  checkpoint_failure_to_breaker:
    condition: "连续2次检查点验证失败"
    action: "触发 Agent 级熔断（见 circuit-breaker.md）"
    
  # 回滚耗尽 → 任务级熔断
  rollback_exhaust_to_breaker:
    condition: "同一任务回滚2次"
    action: "触发 task_level exhaust_action"
    
  # 恢复后重试 → 幂等保障
  recovery_retry_idempotency:
    principle: "从检查点恢复后重试，每步操作幂等，不会重复执行已完成步骤"
    guarantee: "检查点记录了精确的文件状态+hash，重试可确定性地恢复"
```

---

## 5. 各 Agent 默认检查点策略

| Agent | 默认频率 | 保留策略 | 回滚策略 | 特殊说明 |
|-------|---------|---------|---------|---------|
| coder | every_step | all | git_first | 文件修改密集，每步检查点 |
| runner | every_step | minimal | git_first | 需要快速回滚 |
| analyst | verification_only | verification_only | reverse_only | 只读为主，检查点少 |
| planner | verification_only | verification_only | reverse_only | 只读为主 |
| designer | every_step | all | git_first | 原型文件修改密集 |
| researcher | verification_only | verification_only | reverse_only | 纯只读 |
| writer | every_step | all | git_first | 文档修改密集 |
| orchestrator | every_step | minimal | git_first | 需要追踪全局状态 |
| backlog-manager | verification_only | verification_only | reverse_only | 只读为主 |

---

## 6. Harness 覆盖模板

```yaml
# 在特化层声明（如果需要覆盖默认行为）
specialization:
  checkpoint_override:
    additional_trigger: "每次 replace_in_file 后"
    frequency: "every_step"           # every_step | every_n_steps | verification_only
    retention: "all"                   # all | minimal | verification_only
    rollback_strategy: "git_first"     # git_first | checkpoint_first | reverse_only
```

---

*Ironforge Base: Checkpoint Protocol v2.0 — 2026-04-24*
