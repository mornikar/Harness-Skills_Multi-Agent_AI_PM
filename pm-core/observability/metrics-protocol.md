# 指标收集协议

> **版本**：v3.1 Ironforge
> **定位**：pm-core/observability 层 — 量化 Agent 运行状态
> **原则**：从"感觉Agent很慢"到"数据说话：平均15.3轮，P95=28轮"

---

## 一、核心哲学

```
当前状态                              目标状态
──────────                            ──────────
"感觉coder经常超时"                    "coder P95 耗时 28 轮，超出预算40%"
"runner调度效率还行吧"                 "runner 平均调度延迟 3.2 轮"
"最近好像回滚了好几次"                 "本周回滚 7 次，其中 coder 占 5 次"
无法量化改进效果                       指标对比 → 改进前后可量化评估
```

---

## 二、指标存储

### 2.1 存储格式

```yaml
storage:
  path: "{context_root}/metrics/{YYYY-MM-DD}.jsonl"
  format: "JSONL（追加写入）"
  rotation: "30天滚动删除"
  immutable: true                       # 追加写，不允许修改
```

### 2.2 指标条目格式

```json
{
  "timestamp": "2026-04-23T14:30:00.123Z",
  "trace_id": "proj-20260423-a3f7",
  "span_id": "proj-20260423-a3f7-coder-T001",
  "metric_name": "task_duration_turns",
  "metric_type": "histogram",
  "value": 18,
  "labels": {
    "agent_type": "pm-coder",
    "task_type": "frontend",
    "result": "success"
  }
}
```

---

## 三、指标类型

### 3.1 计数器（Counter）

> 只增不减的累计值。用于统计事件发生次数。

| 指标名 | 说明 | 标签 |
|--------|------|------|
| `agent_spawn_total` | Agent启动总数 | `agent_type`, `task_id`, `mode` |
| `task_complete_total` | 任务完成总数 | `agent_type`, `task_id`, `result` |
| `file_operation_total` | 文件操作总数 | `agent`, `operation`, `risk_level` |
| `command_execute_total` | 命令执行总数 | `agent`, `command`, `exit_code` |
| `security_event_total` | 安全事件总数 | `agent`, `event_type`, `severity` |
| `retry_total` | 重试总数 | `agent`, `task_id`, `reason` |
| `rollback_total` | 回滚总数 | `agent`, `task_id`, `trigger_reason` |
| `handoff_total` | 交接总数 | `from_agent`, `to_agent`, `reason` |
| `circuit_breaker_total` | 熔断触发总数 | `agent`, `level`, `trigger_reason` |
| `checkpoint_total` | 检查点创建总数 | `agent`, `task_id`, `type` |

### 3.2 仪表盘（Gauge）

> 可增可减的当前值。用于统计瞬时状态。

| 指标名 | 说明 | 标签 |
|--------|------|------|
| `context_usage_percent` | 上下文使用率 | `agent`, `task_id` |
| `health_score` | Agent健康度 | `agent`, `task_id` |
| `active_agents` | 活跃Agent数 | `team_name`, `agent_type` |
| `project_progress_percent` | 项目总体进度 | `project_name` |
| `pending_tasks` | 待执行任务数 | `project_name`, `priority` |
| `blocked_tasks` | 阻塞任务数 | `project_name`, `block_reason` |

### 3.3 直方图（Histogram）

> 值的分布统计。用于分析耗时、频率等分布。

| 指标名 | 说明 | 标签 | 桶边界 |
|--------|------|------|--------|
| `task_duration_turns` | 任务耗时（轮次） | `agent_type`, `task_type` | [5, 10, 20, 30, 50, 80] |
| `step_duration_turns` | 步骤耗时（轮次） | `agent_type`, `step_type` | [1, 3, 5, 10, 15] |
| `handoff_frequency` | 交接频率 | `agent_type`, `reason` | [1, 2, 3, 5, 10] |
| `error_to_recovery_turns` | 错误到恢复耗时 | `agent_type`, `error_type` | [1, 2, 5, 10, 20] |
| `checkpoint_size_bytes` | 检查点大小 | `agent_type` | [1KB, 10KB, 100KB, 1MB] |

---

## 四、指标收集时机

### 4.1 自动收集

| 时机 | 收集的指标 | 收集者 |
|------|----------|--------|
| Agent spawn | `agent_spawn_total`, `active_agents` | runner |
| Agent shutdown | `task_complete_total`, `active_agents` | runner |
| 文件操作 | `file_operation_total` | 各Agent |
| 命令执行 | `command_execute_total` | 各Agent |
| 安全事件 | `security_event_total` | 安全钩子 |
| 重试 | `retry_total` | 熔断机制 |
| 回滚 | `rollback_total` | 检查点协议 |
| 交接 | `handoff_total` | 交接棒协议 |
| 熔断触发 | `circuit_breaker_total` | 熔断机制 |
| 检查点创建 | `checkpoint_total` | 检查点协议 |

### 4.2 定期收集

| 频率 | 收集的指标 | 收集者 |
|------|----------|--------|
| 每5分钟 | `health_score`, `context_usage_percent` | runner |
| 每15分钟 | `project_progress_percent` | orchestrator |
| 任务完成时 | `task_duration_turns`, `step_duration_turns` | 各Agent |
| 交接完成时 | `handoff_frequency`, `error_to_recovery_turns` | 交接棒协议 |

---

## 五、指标查询与分析

### 5.1 查询方式

```yaml
query:
  by_time:
    method: "筛选 timestamp 在范围内的条目"
  by_metric:
    method: "筛选 metric_name 匹配的条目"
  by_label:
    method: "筛选 labels 中包含指定键值对的条目"
  by_trace:
    method: "筛选 trace_id 匹配的条目"
```

### 5.2 常用分析

```yaml
analysis:
  agent_efficiency:
    query: "task_duration_turns group by agent_type"
    output: "各Agent平均耗时、P95耗时、超预算比例"
    
  project_health:
    query: "health_score, error_count, retry_count group by agent"
    output: "项目整体健康趋势、问题聚集点"
    
  security_analysis:
    query: "security_event_total group by severity, event_type"
    output: "安全事件分布、高风险操作频率"
    
  rollback_analysis:
    query: "rollback_total group by agent, trigger_reason"
    output: "回滚频率、根因分布、改进方向"
```

---

## 六、与审计日志的关系

```yaml
relationship:
  metrics: "聚合数据，用于趋势分析和量化评估"
  audit: "详细事件，用于故障定位和合规审计"
  correlation: "通过 trace_id + span_id 关联"
  
  typical_workflow:
    step_1: "指标发现异常（如 task_duration_turns P95飙升）"
    step_2: "通过 trace_id 查审计日志找到具体事件"
    step_3: "分析根因 → 修复 → 量化改进效果"
```

---

*v3.1 Ironforge — metrics-protocol.md — 2026-04-24*
