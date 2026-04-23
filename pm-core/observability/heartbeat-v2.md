# 结构化 HEARTBEAT v2 协议

> **版本**：v3.1 Ironforge
> **定位**：pm-core/observability 层 — 机器可解析 + 人类可读
> **原则**：从"Markdown给人看"到"YAML front matter给机器看 + Markdown body给人看"

---

## 一、核心哲学

```
当前状态（v1）                        目标状态（v2）
────────────────                      ────────────────
HEARTBEAT 是纯 Markdown               HEARTBEAT = YAML front matter + Markdown body
只有人类能理解进度                     机器可解析 front matter → 自动化监控
停滞检测靠人工                        规则引擎扫描 front matter → 自动告警
健康度是模糊描述                      health_score 是 0-100 数值 → 可量化
无追踪信息                            trace_id 贯穿审计/指标/检查点
```

---

## 二、HEARTBEAT v2 格式

### 2.1 完整模板

```markdown
---
# ═══════════════════════════════════════
# 结构化元数据（机器可解析）
# ═══════════════════════════════════════
heartbeat_version: "2.0"
trace_id: "proj-20260423-a3f7"
span_id: "proj-20260423-a3f7-coder-T001"
task_id: "T001"
agent: "pm-coder"
status: "running"                        # pending|running|blocked|completed|failed|handoff
progress: 60                             # 0-100
health_score: 82                         # 0-100
context_level: "FULL"                    # MINIMAL|PARTIAL|FULL
current_phase: "execute"                 # explore|approval_gate|execute
started_at: "2026-04-23T09:00:00Z"
updated_at: "2026-04-23T14:30:00Z"
checkpoint_step: 3
error_count: 0
retry_count: 0
risk_level: "low"                        # low|medium|high|critical
turn_usage: "18/50"                      # 已用/上限
---

# 任务进度（人类可读）

## 已完成步骤
- [x] Step 1: 创建组件文件
- [x] Step 2: 实现表单逻辑
- [x] Step 3: 添加路由配置

## 进行中
- [ ] Step 4: 编写单元测试（60%）

## 产出物
- src/components/UserForm.vue (2.3KB)
- src/router/index.ts (已修改)

## 问题与风险
- 无
```

### 2.2 字段定义

#### 核心字段（必填）

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `heartbeat_version` | string | 协议版本 | `"2.0"` |
| `trace_id` | string | 项目级追踪ID | `"proj-20260423-a3f7"` |
| `span_id` | string | Agent级追踪ID | `"proj-20260423-a3f7-coder-T001"` |
| `task_id` | string | 任务ID | `"T001"` |
| `agent` | string | Agent名称 | `"pm-coder"` |
| `status` | enum | 当前状态 | `running` |
| `progress` | integer | 进度百分比 0-100 | `60` |
| `health_score` | integer | 健康度 0-100 | `82` |
| `updated_at` | ISO8601 | 最后更新时间 | `"2026-04-23T14:30:00Z"` |

#### 扩展字段（可选但推荐）

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `context_level` | enum | 上下文等级 | `FULL` |
| `current_phase` | enum | 当前执行阶段 | `execute` |
| `started_at` | ISO8601 | 任务启动时间 | - |
| `checkpoint_step` | integer | 最近检查点步骤 | `0` |
| `error_count` | integer | 累计错误数 | `0` |
| `retry_count` | integer | 累计重试数 | `0` |
| `risk_level` | enum | 风险等级 | `low` |
| `turn_usage` | string | 轮次使用情况 | `"0/50"` |

### 2.3 状态枚举

```yaml
status_values:
  pending: "任务已分配，等待启动"
  running: "正在执行"
  blocked: "被阻塞（等待上游/审批/资源）"
  completed: "已完成"
  failed: "执行失败"
  handoff: "正在交接给其他Agent"
```

### 2.4 健康度计算

```yaml
health_score:
  formula: |
    base_score = 100
    
    # 扣分项
    - error_count * 5                    # 每个错误扣5分
    - retry_count * 3                    # 每次重试扣3分
    - (turn_usage_percent > 70 ? 10 : 0) # 轮次超70%扣10分
    - (stale_minutes > 15 ? 15 : 0)      # 停滞超15分钟扣15分
    
    # 最低0分
    health_score = max(0, base_score - deductions)
  
  thresholds:
    critical: 0-30                       # 需要立即干预
    warning: 31-50                       # 需要关注
    healthy: 51-80                       # 正常但有风险
    excellent: 81-100                    # 一切正常
```

---

## 三、解析方式

```yaml
parsing:
  machine:
    method: "读取 YAML front matter"
    tool: "YAML 解析器"
    output: "结构化监控数据"
    use_cases:
      - "自动化监控规则（见第四节）"
      - "指标收集（写入 metrics JSONL）"
      - "追踪关联（通过 trace_id）"
      
  human:
    method: "阅读 Markdown body"
    output: "理解任务进度和产出物"
    use_cases:
      - "orchestrator 评估任务状态"
      - "交接时了解已完成工作"
      - "复盘时回溯执行过程"
```

---

## 四、自动化监控规则

### 4.1 规则引擎

```yaml
monitoring_rules:
  # ─── 停滞检测 ───
  - name: "任务停滞"
    condition: "updated_at 超过15分钟未更新 且 status=running"
    severity: "warning"
    action: |
      1. send_message 健康检查请求
      2. health_score 扣15分
    escalation: "30分钟 → severity=critical → 通知 orchestrator"
    
  # ─── 健康度下降 ───
  - name: "健康度下降"
    condition: "health_score < 50"
    severity: "warning"
    action: |
      1. send_message 健康预警
      2. 评估是否需要交接或熔断
    escalation: "health_score < 30 → severity=critical → 触发熔断评估"
    
  # ─── 轮次超支 ───
  - name: "轮次超支"
    condition: "turn_usage 使用率 > 70%"
    severity: "warning"
    action: |
      1. send_message 预算预警
      2. 启动交接评估（是否需要 handoff）
    escalation: "使用率 > 90% → severity=critical → 强制交接"
    
  # ─── 错误聚集 ───
  - name: "错误聚集"
    condition: "error_count >= 3"
    severity: "warning"
    action: |
      1. send_message 错误聚集预警
      2. 启动熔断评估
    escalation: "error_count >= 5 → severity=critical → 触发Agent级熔断"
    
  # ─── 阻塞超时 ───
  - name: "阻塞超时"
    condition: "status=blocked 且 持续超过30分钟"
    severity: "warning"
    action: |
      1. send_message 阻塞预警
      2. 评估是否可以解除阻塞
    escalation: "超过60分钟 → severity=critical → 通知 orchestrator"
```

### 4.2 监控执行者

```yaml
monitor_executor:
  primary: "pm-runner"                    # runner 监控子Agent
  fallback: "pm-orchestrator"             # orchestrator 监控 runner
  
  check_interval: "5分钟"                 # 每5分钟扫描一次 HEARTBEAT
  
  scan_method: |
    1. 读取所有活跃任务的 HEARTBEAT front matter
    2. 对每条规则执行条件检查
    3. 触发的规则 → 执行对应 action
    4. 写入审计日志 monitoring_event
```

---

## 五、与 v1 的兼容

### 5.1 升级路径

```yaml
upgrade:
  # v1 → v2 自动转换
  v1_to_v2:
    step_1: "在现有 Markdown 前插入 YAML front matter"
    step_2: "从 Markdown body 提取已知字段填充 front matter"
    step_3: "保留 Markdown body 不变"
    
  # v2 向下兼容 v1
  v2_to_v1:
    method: "忽略 YAML front matter，只读 Markdown body"
    guarantee: "v1 工具仍然可以读取 v2 HEARTBEAT 的 Markdown 部分"
```

### 5.2 读写协议

```yaml
read_write:
  # 写入：每次状态变更时更新
  write_trigger:
    - "任务启动时"
    - "每个 plan step 完成后"
    - "状态变更时（running→blocked→completed等）"
    - "健康度变更时"
    - "错误/重试发生时"
    
  # 更新方式
  update_method: |
    1. 读取当前 HEARTBEAT
    2. 更新 YAML front matter 字段
    3. 更新 Markdown body 对应部分
    4. 使用 replace_in_file 写入（幂等）
    
  # 幂等保障
  idempotency: "相同状态的重复写入不会产生副作用"
```

---

## 六、各 Agent HEARTBEAT 特化

| Agent | 特殊字段 | Markdown body 特殊内容 |
|-------|---------|----------------------|
| coder | `current_phase`, `code_review_stage` | 产出物清单 + 代码变更摘要 |
| runner | `active_sub_agents`, `strategy_triggered` | 子Agent状态列表 + 策略执行日志 |
| analyst | `question_rounds` | 追问轮次 + 需求澄清摘要 |
| planner | `dag_status` | 模块DAG状态 + 依赖关系 |
| designer | `prototype_coverage` | 原型页面清单 + 覆盖度 |
| researcher | `dimensions_covered` | 调研维度 + 来源时效标注 |
| writer | `chapters_completed` | 章节完成度 + 术语一致性 |
| orchestrator | `active_phase`, `global_health` | 全局状态 + 阶段进度 |
| backlog-manager | `mvp_status` | P0/P1/P2需求数 + MVP范围 |

---

*v3.1 Ironforge — heartbeat-v2.md — 2026-04-24*
