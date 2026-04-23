# 分布式追踪协议

> **版本**：v3.1 Ironforge
> **定位**：pm-core/observability 层 — 跨 Agent 调用链追踪
> **原则**：从"不知道哪个Agent出了问题"到"trace_id 一键定位完整调用链"

---

## 一、核心哲学

```
当前状态                              目标状态
──────────                            ──────────
Agent 间消息传递无追踪ID               每条消息自动携带 trace_id + span_id
故障定位靠人工排查                     trace_id 查询 → 完整调用链
无法分析 Agent 间耗时分布              span 耗时统计 → 瓶颈定位
跨 Agent 调试需要翻多个日志            一条 trace_id 串联所有事件
```

**追踪三问**：
1. **哪个Agent出了问题？** → trace_id 查询调用链
2. **卡在哪一步？** → span 耗时统计
3. **怎么传过去的？** → 消息传播路径追踪

---

## 二、ID 生成规则

### 2.1 Trace ID

```yaml
trace_id:
  format: "proj-{YYYYMMDD}-{random4}"
  example: "proj-20260423-a3f7"
  generated_by: "orchestrator（编排启动时）"
  scope: "一个完整的项目周期"
  uniqueness: "项目内唯一，跨项目可能重复（可接受）"
  
  generation_timing:
    - "orchestrator 接收到用户需求时"
    - "新的编排周期开始时"
```

### 2.2 Span ID

```yaml
span_id:
  format: "{trace_id}-{agent}-{task_id}"
  example: "proj-20260423-a3f7-coder-T001"
  generated_by: "每个Agent启动时（runner spawn 时注入）"
  scope: "一个Agent处理一个任务的完整生命周期"
  uniqueness: "trace_id 内唯一"
  
  generation_timing:
    - "runner spawn 子Agent 时"
    - "orchestrator 直接执行任务时"
```

### 2.3 Parent Span ID

```yaml
parent_span_id:
  format: "同 span_id 格式"
  example: "proj-20260423-a3f7-runner"
  usage: "标识触发当前 span 的上游 span"
  typical_chain: |
    orchestrator (span: proj-...-orchestrator)
      └→ runner (span: proj-...-runner, parent: proj-...-orchestrator)
          └→ coder (span: proj-...-coder-T001, parent: proj-...-runner)
```

---

## 三、传播方式

### 3.1 传播渠道

| 渠道 | 携带字段 | 注入时机 |
|------|---------|---------|
| `send_message` | 消息头部自动携带 `trace_id` + `span_id` | 每条消息 |
| `HEARTBEAT` | YAML front matter 的 `trace_id` + `span_id` | 每次更新 |
| `audit_log` | 每条审计记录的 `trace_id` + `span_id` | 每次写入 |
| `metrics` | 每条指标的 `trace_id` + `span_id` labels | 每次收集 |
| `checkpoint` | 每个检查点的 `trace_id` + `span_id` | 每次创建 |
| `HANDOFF` | 交接文档头部 | 每次交接 |

### 3.2 传播规则

```yaml
propagation_rules:
  rule_1:
    description: "orchestrator 生成 trace_id 后，所有后续操作都携带同一 trace_id"
    guarantee: "通过 trace_id 可查询项目的完整操作历史"
    
  rule_2:
    description: "每个 Agent 处理一个任务时使用唯一 span_id"
    guarantee: "通过 span_id 可查询该 Agent 处理该任务的完整历史"
    
  rule_3:
    description: "子 span 记录 parent_span_id，形成调用树"
    guarantee: "可还原完整的 Agent 间调用链"
    
  rule_4:
    description: "trace_id 跨审计日志/指标/检查点/HEARTBEAT 传播"
    guarantee: "任何系统的事件都可通过 trace_id 关联"
```

---

## 四、调用链还原

### 4.1 调用链示例

```
orchestrator [proj-20260423-a3f7-orchestrator]
├── spawn runner [proj-20260423-a3f7-runner] (parent: orchestrator)
│   ├── spawn coder [proj-20260423-a3f7-coder-T001] (parent: runner)
│   │   ├── file_create: src/components/LoginForm.vue
│   │   ├── file_modify: src/router/index.ts
│   │   └── task_complete → runner
│   ├── spawn coder [proj-20260423-a3f7-coder-T002] (parent: runner)
│   │   ├── file_create: src/api/auth.ts
│   │   ├── security_scan_hit: API Key detected → blocked
│   │   ├── file_modify: src/api/auth.ts (replaced with env var)
│   │   └── task_complete → runner
│   └── spawn researcher [proj-20260423-a3f7-researcher-T003] (parent: runner)
│       ├── web_search: "best auth library 2026"
│       └── task_complete → runner
└── phase_transition: P3 → P4
```

### 4.2 调用链查询

```yaml
query:
  by_trace:
    input: "proj-20260423-a3f7"
    source: "audit_log + metrics + checkpoints"
    output: "按时间排序的所有事件"
    
  by_span:
    input: "proj-20260423-a3f7-coder-T001"
    source: "audit_log + checkpoints"
    output: "该Agent的所有操作"
    
  by_parent:
    input: "proj-20260423-a3f7-runner"
    source: "audit_log"
    output: "runner 派发的所有子Agent操作"
```

---

## 五、追踪用途

### 5.1 故障定位

```yaml
fault_localization:
  scenario: "coder T002 报告 security_scan_hit"
  steps:
    1: "查询 trace_id → 找到完整调用链"
    2: "定位到 span_id=...-coder-T002 的所有事件"
    3: "发现 security_scan_hit → 审计日志有详情"
    4: "发现后续 file_modify 事件 → 已自动替换为环境变量"
    5: "确认问题已解决，无需人工干预"
```

### 5.2 性能分析

```yaml
performance_analysis:
  scenario: "项目整体耗时偏高"
  steps:
    1: "查询 metrics 的 task_duration_turns"
    2: "按 agent_type 分组 → 发现 researcher P95=35轮"
    3: "查 trace_id → 找到 researcher 的具体任务"
    4: "分析 span 级耗时 → 定位到 web_search 步骤"
    5: "优化建议：缓存常见搜索结果"
```

### 5.3 合规审计

```yaml
compliance_audit:
  scenario: "需要回溯某个文件的所有变更历史"
  steps:
    1: "审计日志查询 file_modify event where path=目标文件"
    2: "获取所有 trace_id + span_id"
    3: "还原每次变更的完整上下文"
    4: "输出合规报告"
```

### 5.4 跨Agent调试

```yaml
cross_agent_debug:
  scenario: "交接后新Agent行为异常"
  steps:
    1: "查 HANDOFF 的 trace_id"
    2: "追踪交接前后的消息传递"
    3: "发现交接时遗漏了某个上下文文件"
    4: "修正交接协议，补充遗漏文件"
```

---

## 六、与 HEARTBEAT 和指标的关系

```yaml
integration:
  heartbeat:
    trace_id: "从项目启动时注入，整个项目周期不变"
    span_id: "从 Agent spawn 时注入，整个任务周期不变"
    
  metrics:
    trace_id_label: "每条指标都带 trace_id label"
    span_id_label: "每条指标都带 span_id label"
    
  audit:
    trace_id_field: "每条审计记录都带 trace_id"
    span_id_field: "每条审计记录都带 span_id"
    
  typical_workflow:
    step_1: "指标发现异常"
    step_2: "通过 trace_id 查审计日志"
    step_3: "通过 span_id 定位具体Agent操作"
    step_4: "通过 parent_span_id 追踪上游"
    step_5: "分析根因 → 修复 → 量化改进"
```

---

*v3.1 Ironforge — trace-protocol.md — 2026-04-24*
