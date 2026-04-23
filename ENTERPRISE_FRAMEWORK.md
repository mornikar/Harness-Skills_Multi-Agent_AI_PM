# AI PM Skills 企业工程化框架

> **代号**：Ironforge（铁炉堡）—— 把软约束锻造成硬骨架
> **目标**：从"Agent自觉遵守"升级为"做不到不合规的事"
> **适用版本**：v3 高解耦架构 → v3.1 企业加固版

---

## 一、核心哲学

```
当前状态：设计级成熟                          目标状态：工程级成熟
─────────────────                          ─────────────────
"告诉Agent不要做"                            "让Agent做不到"
YAML描述 → Agent可能无视                      平台机制 → 物理阻断
文档级恢复配方 → 靠Agent理解执行               检查点协议 → 确定性恢复
HEARTBEAT Markdown → 人类阅读                结构化元数据 → 自动化监控
每个Harness独立1000+行 → 重复难维护            分层复用 → 改一处生效全局
```

**三条铁律**：
1. **约束可执行**：每条安全规则必须有对应的物理阻断机制，不能只靠prompt
2. **操作可追溯**：每个文件操作、命令执行、权限变更必须有审计记录
3. **故障可恢复**：每个执行步骤都有检查点，崩溃后可确定性恢复到最近安全状态

---

## 二、框架全景

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Ironforge 企业工程化框架                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Layer 0+: pm-core 扩展层                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │ context  │ │ agent    │ │ platform │ │ security │ 🆕    │   │
│  │  │ protocol │ │ lifecycle│ │ adapter  │ │ framework│       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │ stability│ 🆕│ observ-  │ 🆕│ existing │   │  ref/    │       │   │
│  │  │ framework│   │ ability │   │ refs    │   │  templates│     │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓ 继承                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Layer 1+: Skills 行为规范层（不变）               │   │
│  │  9个独立Skill，三模式工作流，平台无关                           │   │
│  │  新增：每个Skill的SKILL.md引用security/stability协议           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓ 适配                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Layer 2+: Harness 加固层（核心改造）              │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │  base/ 公共层（🆕 抽取复用）                            │   │   │
│  │  │  permission-framework · security-hooks · handoff      │   │   │
│  │  │  checkpoint-protocol · audit-logging                  │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │  特化层（每个Agent只写差异部分）                          │   │   │
│  │  │  coder-harness · runner-harness · analyst-harness ... │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、三大支柱

### 支柱A：安全加固（Fortify）

> 从"告知Agent什么不能做"到"让Agent根本做不到"

#### A1. 权限强制执行

**现状问题**：
```
coder-harness 说："红灯操作需审批"
Agent 行为：看见红灯 → 理解了 → 照做（大部分时候）/ 无视（偶尔）
结果：不可靠，安全靠Agent"自觉"
```

**企业级方案**：
```
分层权限执行模型：

┌────────────────────────────────────────────────────────┐
│ Layer 1: 平台物理层（不可绕过）                          │
│                                                        │
│ WorkBuddy spawn配置：                                  │
│   explore阶段 → mode: "plan" → 写/改/执行 物理不可用   │
│   execute阶段 → mode: "acceptEdits" → 写/改可用        │
│   blocked_tools: [delete_file] → 红灯工具直接移除       │
│                                                        │
│ → Agent 的 tool 列表里根本没有这些工具，想做也做不到    │
├────────────────────────────────────────────────────────┤
│ Layer 2: Harness 逻辑层（语义约束）                     │
│                                                        │
│ 黄灯操作：执行前 send_message 通知                      │
│   → orchestrator 可在下一个轮询拦截                     │
│   → 不是物理阻断，但有监督窗口                          │
│                                                        │
│ → 允许执行但有"跟踪摄像头"                              │
├────────────────────────────────────────────────────────┤
│ Layer 3: SKILL.md 行为层（自律约束）                    │
│                                                        │
│ 编码规范、禁止事项、最佳实践                             │
│ → 靠Agent理解执行，无法强制                             │
│                                                        │
│ → "道德准则"，作为兜底                                  │
└────────────────────────────────────────────────────────┘

安全强度：Layer 1 > Layer 2 > Layer 3
实际执行：三层叠加，越危险的操作越靠底层拦截
```

**spawn配置模板**：

```yaml
# harnesses/base/permission-framework.md

# ═══════════════════════════════════════════
# Phase A: 探索阶段 — 物理只读
# ═══════════════════════════════════════════
explore_phase:
  mode: "plan"                           # WorkBuddy原生只读模式
  max_turns: 10
  available_tools:                       # 显式白名单
    - read_file
    - search_file
    - search_content
    - list_dir
    - send_message
  blocked_tools:                         # 物理阻断
    - write_to_file
    - replace_in_file
    - execute_command
    - delete_file

# ═══════════════════════════════════════════
# Phase C: 执行阶段 — 分级放权
# ═══════════════════════════════════════════
execute_phase:
  mode: "acceptEdits"
  max_turns: 40
  available_tools:
    green:                               # 自主执行
      - read_file
      - search_file
      - search_content
      - list_dir
      - write_to_file                    # 创建新文件
      - send_message
    yellow:                              # 通知后执行
      - replace_in_file                  # 修改已有文件
      - execute_command:                 # 仅白名单命令
          whitelist: [npm_install, npm_run_build, npm_run_test,
                      npm_run_lint, tsc_noEmit, python_pytest,
                      python_black, python_mypy]
          require_notification: true
    red:                                 # 需审批
      - delete_file                      # 从可用工具中移除！
      - execute_command:                 # 非白名单命令
          require_approval: true
  blocked_tools:                         # 始终不可用
    - delete_file                        # coder 不应删除文件

# ═══════════════════════════════════════════
# 各Agent特化覆盖（在特化层声明）
# ═══════════════════════════════════════════
# coder: execute_phase.blocked_tools += [delete_file]
# runner: execute_phase.green += [task, team_create, team_delete]
# researcher: 始终 plan 模式，无 execute_phase
```

#### A2. 敏感信息防护

```yaml
# harnesses/base/security-hooks.md

secrets_protection:
  # 1. 写入前扫描
  pre_write_scan:
    trigger: "write_to_file / replace_in_file 执行前"
    type: "computational"                  # 确定性正则，不依赖推理
    patterns:
      - name: "API Key"
        regex: '(sk-|pk_|AKIA|ghp_|gho_)[a-zA-Z0-9]{20,}'
        severity: CRITICAL
      - name: "数据库连接串"
        regex: '(mongodb|mysql|postgres|redis)://[^\s]+:[^\s]+@'
        severity: CRITICAL
      - name: "密码字段"
        regex: '(password|passwd|pwd|secret|token)\s*[:=]\s*[''""][^'""]{8,}'
        severity: HIGH
      - name: "私钥"
        regex: '-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----'
        severity: CRITICAL
    on_hit:
      action: "abort_write"                # 阻止写入
      remediation: |
        1. 将敏感值替换为环境变量引用
           例: sk-abc123... → process.env.OPENAI_API_KEY
        2. 重新执行写入（替换后的内容）
      notification: |
        send_message(type="message", recipient="main",
          event_type="security_violation",
          severity: "{severity}",
          detail: "检测到{pattern_name}，已替换为环境变量引用",
          file: "{file_path}")

  # 2. 受保护路径
  protected_paths:
    never_read:                           # 任何Agent都不应读取
      - "**/.env"
      - "**/.env.*"
      - "**/credentials/**"
      - "**/.ssh/**"
      - "**/*.pem"
      - "**/*.key"
    read_only:                            # 可读不可写
      - "package.json"                    # 只有runner审批后可改
      - "tsconfig.json"
      - "docker-compose.yml"
    agent_specific:                       # Agent级覆盖
      coder:
        read_only += ["src/**/*.test.*"]  # 测试文件只读（防误改）
      runner:
        read_write += ["package.json"]    # runner可改依赖

  # 3. 网络访问控制
  network_policy:
    default: DENY                         # 默认禁止
    allowed_domains:
      - "npmjs.org"                       # npm install
      - "pypi.org"                        # pip install
      - "github.com"                      # git clone
    approval_required:
      - 任何其他域名
    prohibited:
      - 内网IP段（防止SSRF）
```

#### A3. 审计日志

```yaml
# pm-core/security/audit-protocol.md

audit:
  # 日志格式：JSONL（追加写入，不可修改）
  path: "{context_root}/audit/{YYYY-MM-DD}.jsonl"
  rotation: "30天滚动删除"
  immutable: true                         # 追加写，不允许修改历史条目

  # 日志条目格式
  entry_schema:
    timestamp: "ISO8601"                  # 2026-04-23T14:30:00.123Z
    trace_id: "string"                    # proj-20260423-a3f7
    span_id: "string"                     # proj-20260423-a3f7-T001
    agent: "string"                       # pm-coder
    task_id: "string"                     # T001
    event_type: "enum"                    # 见下表
    detail: "object"                      # 事件详情
    risk_level: "enum|null"              # green|yellow|red|prohibited|null

  # 必须审计的事件
  events:
    # ─── Agent 生命周期 ───
    - event_type: agent_spawn
      detail: {agent, task_id, parent_agent, mode, max_turns}
      
    - event_type: agent_shutdown
      detail: {agent, task_id, reason, final_status}
      
    # ─── 文件操作 ───
    - event_type: file_read
      detail: {path, size}
      risk_level: null                    # 读操作无风险
      
    - event_type: file_create
      detail: {path, size, content_hash}
      risk_level: "green"
      
    - event_type: file_modify
      detail: {path, diff_lines_added, diff_lines_removed, content_hash}
      risk_level: "yellow"
      
    - event_type: file_delete
      detail: {path, was_backed_up}
      risk_level: "red"
      
    # ─── 命令执行 ───
    - event_type: command_execute
      detail: {command, exit_code, duration_ms, output_summary}
      risk_level: "根据白名单动态判定"
      
    # ─── 权限事件 ───
    - event_type: approval_request
      detail: {agent, operation, risk_level, reason}
      
    - event_type: approval_result
      detail: {approver, approved, reason}
      
    - event_type: permission_violation
      detail: {agent, attempted_operation, blocked_by}
      
    # ─── 安全事件 ───
    - event_type: security_scan_hit
      detail: {agent, pattern_name, severity, file, remediation}
      
    - event_type: security_violation
      detail: {agent, violation_type, detail, action_taken}
      
    # ─── 协作事件 ───
    - event_type: handoff
      detail: {from_agent, to_agent, task_id, reason}
      
    - event_type: message_sent
      detail: {from_agent, to_agent, event_type, summary}

  # 查询接口
  query:
    - "按时间范围查询"
    - "按Agent类型查询"
    - "按风险等级查询"
    - "按trace_id追踪完整调用链"
```

---

### 支柱B：稳定性加固（Stabilize）

> 从"文档级恢复配方"到"确定性检查点恢复"

#### B1. 检查点协议

```yaml
# pm-core/stability/checkpoint-protocol.md

checkpoint:
  # ═══════════════════════════════════════
  # 自动检查点
  # ═══════════════════════════════════════
  auto:
    trigger: "每个 plan step 完成后"
    storage: "{context_root}/checkpoints/T{XXX}.jsonl"
    format:
      step: 3
      timestamp: "2026-04-23T14:30:00Z"
      status: "step_completed"             # step_completed|verification_passed|verification_failed
      files_created: ["src/components/UserForm.vue"]
      files_modified: ["src/router/index.ts"]
      file_hashes:
        "src/components/UserForm.vue": "sha256:abc123..."
        "src/router/index.ts": "sha256:def456..."
      heartbeat_snapshot:
        status: "running"
        progress: 60
        health_score: 82
      verification_result: "passed"        # 该步骤验证结果

  # ═══════════════════════════════════════
  # 验证点检查点（更高级别）
  # ═══════════════════════════════════════
  verification:
    trigger: "推理验证点（RC-P3-02等）通过后"
    additional_fields:
      verification_id: "RC-P3-02"
      verification_detail: "代码与原型一致性检查通过"
    importance: "high"                     # 回滚优先使用验证点检查点

  # ═══════════════════════════════════════
  # 回滚机制
  # ═══════════════════════════════════════
  rollback:
    trigger: "验证点不通过 且 重试2次仍失败"
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
        action: "从检查点的下一步开始重新执行 plan"
        
      - step: "审计记录"
        action: "写入审计日志 rollback 事件"

  # ═══════════════════════════════════════
  # 检查点清理
  # ═══════════════════════════════════════
  cleanup:
    trigger: "任务 COMPLETED 后"
    action: |
      保留：第一个 + 最后一个 + 所有 verification_passed
      删除：中间的 step_completed 检查点
    reason: "减少存储占用，保留关键恢复点"
```

#### B2. 熔断机制

```yaml
# pm-core/stability/circuit-breaker.md

circuit_breaker:
  # ═══════════════════════════════════════
  # Agent级熔断
  # ═══════════════════════════════════════
  agent_level:
    scope: "同一 project 内的同一 agent 类型"
    failure_threshold: 3                   # 连续3次失败触发熔断
    window: "30min"                        # 30分钟滑动窗口
    states:
      closed:                              # 正常
        description: "正常派发任务"
        
      open:                                # 熔断
        description: "停止派发新任务给该Agent"
        action: |
          1. 停止向该类型Agent派发新任务
          2. send_message 通知 orchestrator
          3. 等待 cooldown 时间后进入 half_open
        cooldown: "30min"
        
      half_open:                           # 试探
        description: "允许1次试探性任务"
        action: |
          派发1个低风险任务
          ├── 成功 → closed（恢复正常）
          └── 失败 → open（继续熔断，cooldown加倍）

  # ═══════════════════════════════════════
  # 任务级熔断
  # ═══════════════════════════════════════
  task_level:
    retry_budget: 3                        # 同一任务最多重试3次
    backoff: "exponential"                 # 1min → 2min → 4min
    exhaust_action: |
      1. 写入审计日志 task_exhausted
      2. send_message 上报 orchestrator
      3. 标记任务 status: failed
      4. 不再自动重试，等待人工决策

  # ═══════════════════════════════════════
  # 项目级熔断
  # ═══════════════════════════════════════
  project_level:
    trigger: "项目总轮次 > 80% 且 完成率 < 30%"
    action: |
      1. 全局暂停所有Agent
      2. send_message 紧急通知 orchestrator
      3. 建议：重新评估项目范围或调整任务拆解
    severity: "CRITICAL"
```

#### B3. 幂等性规范

```yaml
# pm-core/stability/idempotency.md

idempotency:
  # ═══════════════════════════════════════
  # 文件操作幂等性
  # ═══════════════════════════════════════
  file_operations:
    write_to_file:
      rule: "创建前先检查文件是否存在"
      procedure: |
        1. read_file(path)
        2. 文件不存在 → write_to_file 创建
        3. 文件已存在 → 判断策略：
           a. plan.md 标记为"新建" → 确认覆盖（需审批）
           b. plan.md 标记为"修改" → 改用 replace_in_file
           c. 上下文恢复场景 → 直接覆盖（HANDOFF中有记录）
        
    replace_in_file:
      rule: "替换前验证 old_str 仍存在且唯一"
      procedure: |
        1. search_content(pattern=old_str, path=target_file)
        2. 恰好1个匹配 → 执行替换
        3. 0个匹配 → old_str已被替换，跳过（幂等成功）
        4. 多个匹配 → 缩小 old_str 范围增加上下文，重试
        
    delete_file:
      rule: "删除前确认文件存在且已备份"
      procedure: |
        1. read_file(path) → 确认存在
        2. 备份到 {context_root}/checkpoints/deleted/
        3. 执行删除
        4. 记录审计日志

  # ═══════════════════════════════════════
  # 命令执行幂等性
  # ═══════════════════════════════════════
  command_operations:
    npm_install:
      pre_check: "node_modules/.package-lock.json 存在 → 跳过"
      idempotent: true                     # 重复执行无害
      
    npm_run_build:
      pre_check: "无（每次重新构建）"
      idempotent: true                     # 重复构建无害
      
    database_migration:
      pre_check: "检查 migration 是否已执行"
      idempotent: "依赖迁移脚本设计（需包含 IF NOT EXISTS）"
      
    git_commit:
      pre_check: "git status 是否有未提交变更"
      idempotent: false                    # 非幂等，需特殊处理
```

---

### 支柱C：可观测性加固（Observe）

> 从"HEARTBEAT Markdown"到"结构化指标+追踪+告警"

#### C1. 结构化 HEARTBEAT v2

```yaml
# pm-core/observability/heartbeat-v2.md

heartbeat_v2:
  # ═══════════════════════════════════════
  # 格式：YAML front matter + Markdown body
  # ═══════════════════════════════════════
  format: |
    ---
    # 结构化元数据（机器可解析）
    heartbeat_version: "2.0"
    trace_id: "proj-20260423-a3f7"
    task_id: "T001"
    agent: "pm-coder"
    status: "running"                      # pending|running|blocked|completed|failed|handoff
    progress: 60                           # 0-100
    health_score: 82                       # 0-100
    context_level: "FULL"                  # MINIMAL|PARTIAL|FULL
    current_phase: "execute"               # explore|approval_gate|execute
    started_at: "2026-04-23T09:00:00Z"
    updated_at: "2026-04-23T14:30:00Z"
    checkpoint_step: 3
    error_count: 0
    retry_count: 0
    risk_level: "low"                      # low|medium|high|critical
    turn_usage: "18/50"                    # 已用/上限
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

  # ═══════════════════════════════════════
  # 解析方式
  # ═══════════════════════════════════════
  parsing:
    machine: "读取YAML front matter → 结构化监控"
    human: "阅读Markdown body → 理解进度"
    
  # ═══════════════════════════════════════
  # 自动化监控规则
  # ═══════════════════════════════════════
  monitoring_rules:
    - name: "任务停滞"
      condition: "updated_at 超过15分钟未更新 且 status=running"
      action: "send_message 健康检查请求"
      
    - name: "健康度下降"
      condition: "health_score < 50"
      action: "send_message 健康预警"
      
    - name: "轮次超支"
      condition: "turn_usage 使用率 > 70%"
      action: "send_message 预算预警 + 启动交接评估"
      
    - name: "错误聚集"
      condition: "error_count >= 3"
      action: "send_message 错误聚集预警 + 启动熔断评估"
```

#### C2. 指标收集协议

```yaml
# pm-core/observability/metrics-protocol.md

metrics:
  storage: "{context_root}/metrics/{YYYY-MM-DD}.jsonl"
  
  # ═══════════════════════════════════════
  # 计数器（Counter）
  # ═══════════════════════════════════════
  counters:
    - name: agent_spawn_total
      description: "Agent启动总数"
      labels: [agent_type, task_id, mode]
      
    - name: task_complete_total
      description: "任务完成总数"
      labels: [agent_type, task_id, result]
      result_values: [success, partial_success, failed]
      
    - name: file_operation_total
      description: "文件操作总数"
      labels: [agent, operation, risk_level]
      operation_values: [read, create, modify, delete]
      
    - name: command_execute_total
      description: "命令执行总数"
      labels: [agent, command, exit_code]
      
    - name: security_event_total
      description: "安全事件总数"
      labels: [agent, event_type, severity]
      
    - name: retry_total
      description: "重试总数"
      labels: [agent, task_id, reason]
      
    - name: rollback_total
      description: "回滚总数"
      labels: [agent, task_id, trigger_reason]

  # ═══════════════════════════════════════
  # 仪表盘（Gauge）
  # ═══════════════════════════════════════
  gauges:
    - name: context_usage_percent
      description: "上下文使用率"
      labels: [agent, task_id]
      
    - name: health_score
      description: "Agent健康度"
      labels: [agent, task_id]
      
    - name: active_agents
      description: "活跃Agent数"
      labels: [team_name, agent_type]
      
    - name: project_progress_percent
      description: "项目总体进度"
      labels: [project_name]

  # ═══════════════════════════════════════
  # 直方图（Histogram）
  # ═══════════════════════════════════════
  histograms:
    - name: task_duration_turns
      description: "任务耗时（轮次）"
      labels: [agent_type, task_type]
      buckets: [5, 10, 20, 30, 50, 80]
      
    - name: step_duration_turns
      description: "步骤耗时（轮次）"
      labels: [agent_type, step_type]
      buckets: [1, 3, 5, 10, 15]
      
    - name: handoff_frequency
      description: "交接频率"
      labels: [agent_type, reason]
```

#### C3. 分布式追踪

```yaml
# pm-core/observability/trace-protocol.md

tracing:
  # ═══════════════════════════════════════
  # Trace ID 生成规则
  # ═══════════════════════════════════════
  trace_id:
    format: "proj-{YYYYMMDD}-{random4}"
    example: "proj-20260423-a3f7"
    generated_by: "orchestrator（编排启动时）"
    
  # ═══════════════════════════════════════
  # Span ID 生成规则
  # ═══════════════════════════════════════
  span_id:
    format: "{trace_id}-{agent}-{task_id}"
    example: "proj-20260423-a3f7-coder-T001"
    generated_by: "每个Agent启动时"
    
  # ═══════════════════════════════════════
  # 传播方式
  # ═══════════════════════════════════════
  propagation:
    - channel: "send_message"
      field: "消息头部自动携带 trace_id + span_id"
    - channel: "HEARTBEAT"
      field: "YAML front matter 的 trace_id"
    - channel: "audit_log"
      field: "每条审计记录的 trace_id + span_id"
    - channel: "metrics"
      field: "每条指标的 trace_id label"
    - channel: "checkpoint"
      field: "每个检查点的 trace_id"

  # ═══════════════════════════════════════
  # 追踪用途
  # ═══════════════════════════════════════
  use_cases:
    - name: "故障定位"
      description: "根据 trace_id 查找完整调用链，定位哪一步出错"
      
    - name: "性能分析"
      description: "按 span 统计各阶段耗时，发现瓶颈"
      
    - name: "合规审计"
      description: "按 trace_id 回溯完整操作链，满足审计需求"
      
    - name: "跨Agent调试"
      description: "追踪消息在 Agent 间的传递路径"
```

---

## 四、Harness 分层架构

### 4.1 目录结构

```
harnesses/
├── base/                              # 🆕 公共层（所有Agent共享）
│   ├── README.md                      # 分层架构说明
│   ├── permission-framework.md        # A1: 权限强制执行
│   ├── security-hooks.md             # A2: 敏感信息防护
│   ├── audit-logging.md              # A3: 审计日志配置
│   ├── checkpoint-protocol.md        # B1: 检查点（Harness视角）
│   ├── handoff-protocol.md           # 交接棒（从coder-harness抽取）
│   ├── context-engineering.md        # 上下文工程（从coder-harness抽取）
│   └── observability-config.md       # C1-C3: 可观测性配置
│
├── coder-harness.md                   # 只写 coder 特化逻辑
├── runner-harness.md                  # 只写 runner 特化逻辑
├── analyst-harness.md                 # 只写 analyst 特化逻辑
├── planner-harness.md
├── designer-harness.md
├── researcher-harness.md
├── writer-harness.md
├── backlog-manager-harness.md
└── orchestrator-harness.md
```

### 4.2 特化层格式

```yaml
# harnesses/coder-harness.md（改造后）

harness_version: "2.0"
compatible_skill_version: ">=3.0"

# ═══════════════════════════════════════
# 继承公共层
# ═══════════════════════════════════════
inherits:
  - base/permission-framework.md
  - base/security-hooks.md
  - base/audit-logging.md
  - base/checkpoint-protocol.md
  - base/handoff-protocol.md
  - base/context-engineering.md
  - base/observability-config.md

# ═══════════════════════════════════════
# Coder 特化配置
# ═══════════════════════════════════════
specialization:
  # 基本配置
  spawn_config:
    subagent_name: "code-explorer"
    explore_phase:
      mode: "plan"
      max_turns: 10
    execute_phase:
      mode: "acceptEdits"
      max_turns: 40

  # 权限覆盖
  permission_override:
    execute_phase:
      blocked_tools: [delete_file]        # coder 不应删除文件
      yellow_tools: [replace_in_file]
      command_whitelist:
        - npm install
        - npm run build
        - npm run test
        - npm run lint
        - tsc --noEmit
        - python -m pytest
        - python -m black --check

  # 钩子覆盖
  hooks_override:
    add:
      - name: "debug_phase_gate"
        trigger: "调试类任务启动时"
      - name: "debug_stop_loss"
        trigger: "同一bug 2轮未修复"
    remove: []

  # 检查点覆盖
  checkpoint_override:
    additional_trigger: "每次 replace_in_file 后"

  # 审计覆盖
  audit_override:
    additional_events:
      - event_type: code_review
        detail: {stage, result, findings}

  # 上下文预算覆盖
  context_budget_override:
    layer_1_hot: "≤ 3000 tokens"
    layer_2_working: "≤ 15000 tokens"
    handoff_trigger: "70% 轮次使用"

  # 通信事件覆盖
  communication_override:
    additional_notify_events:
      - event: debug_blocked
        message: "【debug_blocked】T{XXX} | pm-coder | 同一bug 2轮未修复"

# ═══════════════════════════════════════
# Coder 特有的详细规范（不与公共层重复）
# ═══════════════════════════════════════
coder_specific:
  # 规划优先管制（Coder独有）
  planning_first_workflow: |
    Phase A: 探索 → Phase B: 审批等待 → Phase C: 编码
    详见原 coder-harness 模块一

  # 两阶段 Code Review（Coder独有）
  code_review: |
    Stage 1: 轻量自检（关键词匹配）
    Stage 2: 计算型检查（H7/H8/H9）
    语义审查由 orchestrator 三角验证执行

  # 调试 SOP（Coder独有）
  debugging_sop: |
    四阶段调试：复现 → 定位 → 修复 → 验证
    详见 references/debugging-protocol.md
```

### 4.3 改造前后对比

```
改造前：
  coder-harness.md: 1055行（包含大量与其他Harness重复的逻辑）
  runner-harness.md: ~400行
  其他7个Harness: 各~200行
  总计: ~3000行，大量重复

改造后：
  base/: 7个文件，每个~200行 = ~1400行（公共逻辑只写一次）
  coder-harness.md: ~200行（只写特化部分）
  runner-harness.md: ~150行
  其他7个Harness: 各~80行
  总计: ~2100行，零重复

  维护成本：改一处生效全部（如安全策略更新 → 只改 base/security-hooks.md）
```

---

## 五、pm-core 扩展

### 5.1 目录结构

```
pm-core/
├── context-protocol.md               # 现有
├── agent-lifecycle.md                 # 现有
├── platform-adapter.md               # 现有
├── SKILL.md                           # 现有
│
├── security/                          # 🆕 支柱A
│   ├── permission-framework.md        #   权限强制执行框架
│   ├── secrets-protection.md          #   敏感信息防护规则
│   └── audit-protocol.md              #   审计日志协议
│
├── stability/                         # 🆕 支柱B
│   ├── checkpoint-protocol.md         #   检查点与回滚
│   ├── circuit-breaker.md             #   熔断机制
│   └── idempotency.md                 #   幂等性规范
│
├── observability/                     # 🆕 支柱C
│   ├── heartbeat-v2.md               #   结构化HEARTBEAT
│   ├── metrics-protocol.md           #   指标收集
│   └── trace-protocol.md             #   分布式追踪
│
├── references/                        # 现有
│   ├── recovery-recipes.md
│   ├── default-skills.md
│   ├── health-check-protocols.md
│   ├── checkpoint-recovery.md
│   ├── message-protocol.md
│   └── lessons-learned.md
│
└── templates/                         # 现有
    └── heartbeat-template.md
```

### 5.2 与现有体系的兼容

```yaml
compatibility:
  # v3 现有功能完全保留
  existing:
    - context-protocol.md → 不变
    - agent-lifecycle.md → 不变
    - platform-adapter.md → 不变
    - 9个SKILL.md → 不变（新增引用security/stability协议）
    - standalone-prompt.md → 不变
    - orchestrations/ → 不变
    
  # v3.1 新增功能向后兼容
  new:
    - security/* → Skill 可选引用，不引用则无安全加固
    - stability/* → Skill 可选引用，不引用则无检查点
    - observability/* → Skill 可选引用，不引用则无指标
    - harnesses/base/ → Harness 可选继承，不继承则用旧格式
    
  # 升级路径
  upgrade_path:
    step_1: "部署 pm-core 扩展文件（security/stability/observability/）"
    step_2: "创建 harnesses/base/ 公共层"
    step_3: "逐个改造 Harness（继承 base/，去掉重复逻辑）"
    step_4: "更新 SKILL.md 引用新协议"
    step_5: "部署验证脚本"
```

---

## 六、验证体系

### 6.1 Harness 验证脚本

```yaml
# scripts/validate-harness.py

validation_checks:
  # ─── 结构完整性 ───
  - id: V01
    name: "Harness文件存在性"
    check: "每个SKILL.md都有对应的harness文件"
    
  - id: V02
    name: "继承声明完整性"
    check: "每个Harness都有 inherits 声明，且 base/ 文件都存在"
    
  - id: V03
    name: "版本兼容性"
    check: "harness_version 和 compatible_skill_version 声明正确"
    
  # ─── 权限一致性 ───
  - id: V04
    name: "权限矩阵无冲突"
    check: "green/yellow/red/blocked 工具列表无重叠"
    
  - id: V05
    name: "SKILL约束覆盖"
    check: "Harness权限 ≥ SKILL.md约束（更严格可以，更宽松不行）"
    
  # ─── 钩子完整性 ───
  - id: V06
    name: "钩子定义完整"
    check: "每个hook都有 name + trigger + check/action + on_failure"
    
  - id: V07
    name: "敏感信息扫描覆盖"
    check: "所有文件写入操作都有 pre_write_scan 钩子"
    
  # ─── 审计覆盖 ───
  - id: V08
    name: "审计事件覆盖"
    check: "file_create/modify/delete + command_execute 都有审计事件"
    
  - id: V09
    name: "审计不可篡改"
    check: "audit log path 声明为追加写入模式"
    
  # ─── 检查点覆盖 ───
  - id: V10
    name: "检查点触发"
    check: "每个 plan step 完成后都有检查点"
    
  - id: V11
    name: "回滚可行性"
    check: "每个检查点记录了 file_hashes，回滚可定位版本"
```

### 6.2 端到端安全测试

```yaml
# scripts/security-test.py

test_cases:
  - id: ST01
    name: "红灯操作阻断测试"
    procedure: "spawn coder → 尝试 delete_file → 验证被阻断"
    expected: "操作被阻止 + 审计日志记录 permission_violation"
    
  - id: ST02
    name: "敏感信息泄露测试"
    procedure: "spawn coder → 写入含API Key的文件 → 验证被拦截"
    expected: "写入被阻止 + 替换为环境变量引用 + 审计日志记录 security_scan_hit"
    
  - id: ST03
    name: "越权路径访问测试"
    procedure: "spawn coder → 尝试读取 .env 文件 → 验证被阻止"
    expected: "读取被阻止 + 审计日志记录"
    
  - id: ST04
    name: "检查点回滚测试"
    procedure: "coder执行3步 → 验证失败 → 触发回滚 → 验证文件恢复"
    expected: "文件恢复到检查点状态 + 从检查点重新执行"
    
  - id: ST05
    name: "熔断测试"
    procedure: "连续3次 spawn coder 失败 → 验证熔断生效"
    expected: "停止派发 + 通知orchestrator + 30min后试探"
```

---

## 七、实施路径

### 阶段1：基础设施（P0，3天）

```
Day 1: pm-core 扩展
  ├── 创建 security/permission-framework.md
  ├── 创建 security/secrets-protection.md
  └── 创建 security/audit-protocol.md

Day 2: Harness 公共层
  ├── 创建 harnesses/base/ 目录
  ├── 从 coder-harness 抽取公共逻辑
  └── 创建 base/ 下7个公共文件

Day 3: Coder Harness 改造 + 验证
  ├── 改造 coder-harness.md 为继承模式
  ├── 写 validate-harness.py
  └── 运行验证 + 修复问题
```

### 阶段2：稳定性（P1，3天）

```
Day 4: 检查点协议
  ├── 创建 stability/checkpoint-protocol.md
  ├── 创建 harnesses/base/checkpoint-protocol.md
  └── 更新 coder-harness 检查点配置

Day 5: 熔断 + 幂等性
  ├── 创建 stability/circuit-breaker.md
  ├── 创建 stability/idempotency.md
  └── 更新 runner-harness 熔断配置

Day 6: 其余 Harness 改造
  ├── 改造 runner-harness.md
  ├── 改造 analyst/planner/designer/researcher/writer-harness.md
  └── 改造 backlog-manager/orchestrator-harness.md
```

### 阶段3：可观测性（P2，3天）

```
Day 7: 结构化 HEARTBEAT
  ├── 创建 observability/heartbeat-v2.md
  ├── 更新 heartbeat-template.md
  └── 更新各 SKILL.md 的 HEARTBEAT 格式说明

Day 8: 指标 + 追踪
  ├── 创建 observability/metrics-protocol.md
  ├── 创建 observability/trace-protocol.md
  └── 更新 base/observability-config.md

Day 9: 安全测试 + 文档更新
  ├── 写 security-test.py
  ├── 运行端到端测试
  ├── 更新 README.md / ARCHITECTURE.md
  └── 记忆更新
```

### 里程碑

| 里程碑 | 完成标志 | 预计日期 |
|--------|---------|---------|
| **M1: 安全基线** | Harness强制执行 + 审计日志 + 敏感信息扫描 | Day 3 |
| **M2: 稳定基线** | 检查点 + 熔断 + 幂等性 + 全部Harness改造 | Day 6 |
| **M3: 可观测基线** | 结构化HEARTBEAT + 指标 + 追踪 + 安全测试 | Day 9 |

---

## 八、版本标识

```
v3.0  → 当前：高解耦 + 平台无关 + 三模式
v3.1  → 目标：v3 + Ironforge 企业工程化框架
         ├── 安全加固（Fortify）
         ├── 稳定性加固（Stabilize）
         └── 可观测性加固（Observe）
```

---

*Ironforge Framework v1.0 — 2026-04-24 (P0+P1+P2 全部完成)*
