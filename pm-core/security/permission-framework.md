# 权限强制执行框架（Permission Framework）

> Ironforge 支柱A 核心组件。
> v3.1 新增。从"告知Agent什么不能做"升级为"让Agent根本做不到"。
>
> 路径变量和操作映射见 `pm-core/platform-adapter.md`。

---

## 1. 设计哲学

```
v3 状态：YAML描述风险等级 → Agent可能遵守也可能无视 → 安全靠"自觉"
v3.1 状态：三层权限执行模型 → 越危险的操作越靠底层拦截 → 安全靠"机制"
```

**核心原则**：约束可执行。每条安全规则必须有对应的物理阻断机制，不能只靠 prompt。

---

## 2. 三层权限执行模型

```yaml
permission_execution_model:
  # ═══════════════════════════════════════════════════
  # Layer 1: 平台物理层（不可绕过）
  # ═══════════════════════════════════════════════════
  layer_1_platform:
    description: |
      利用平台原生的权限机制，在 spawn 配置层面物理阻断。
      Agent 的工具列表里根本没有这些工具，想做也做不到。
    
    implementation:
      WorkBuddy:
        explore_phase:
          mode: "plan"                    # plan 模式 = 只读
          blocked_tools:
            - write_to_file
            - replace_in_file
            - execute_command
            - delete_file
          available_tools:
            - read_file
            - search_file
            - search_content
            - list_dir
            - send_message
        
        execute_phase:
          mode: "acceptEdits"             # 审批后才给写权限
          blocked_tools:
            - delete_file                 # 始终从工具列表移除
        
      Cursor:
        explore_phase: "Read-only rules"
        execute_phase: "Auto-approve for safe ops, manual for risky"
        
      Claude_Code:
        explore_phase: "Plan mode"
        execute_phase: "Auto-accept for safe ops"
    
    strength: "最高 — Agent 物理上无法调用被移除的工具"

  # ═══════════════════════════════════════════════════
  # Layer 2: Harness 逻辑层（语义约束）
  # ═══════════════════════════════════════════════════
  layer_2_harness:
    description: |
      黄灯操作：执行前 send_message 通知。
      orchestrator 可在下一个轮询周期介入拦截。
      允许执行但有"跟踪摄像头"。
    
    implementation:
      - yellow_operations:
          - replace_in_file               # 修改已有文件需通知
          - execute_command:              # 仅白名单命令
              whitelist: [npm_install, npm_run_build, npm_run_test,
                          npm_run_lint, tsc_noEmit, python_pytest,
                          python_black, python_mypy]
              require_notification: true
      
      - notification_protocol: |
          执行黄灯操作前：
          send_message(type="message", recipient="main",
            event_type="risk_notification",
            level="yellow",
            operation="{具体操作}",
            reason="{为什么需要这个操作}")
          
          orchestrator 可在下一个轮询周期介入拦截。
          如果未被拦截 → 操作继续执行。
    
    strength: "中等 — 允许执行但有监督窗口"

  # ═══════════════════════════════════════════════════
  # Layer 3: SKILL.md 行为层（自律约束）
  # ═══════════════════════════════════════════════════
  layer_3_skill:
    description: |
      编码规范、禁止事项、最佳实践。
      靠 Agent 理解执行，无法强制。
      "道德准则"，作为兜底。
    
    coverage:
      - "编码标准（命名、格式、注释）"
      - "架构规范（分层、依赖方向）"
      - "安全最佳实践（输入验证、错误处理）"
      - "超红灯操作禁止（workspace外、全局包安装、硬编码敏感信息）"
    
    strength: "最低 — 靠 Agent 自觉，作为兜底约束"

  # ═══════════════════════════════════════════════════
  # 叠加规则
  # ═══════════════════════════════════════════════════
  stacking_rule: |
    三层叠加，越危险的操作越靠底层拦截：
    - 绿灯操作：仅 Layer 3 约束（自律即可）
    - 黄灯操作：Layer 2 + Layer 3（有监督窗口）
    - 红灯操作：Layer 1 物理阻断 + Layer 2 审批 + Layer 3 禁止
    安全强度：Layer 1 > Layer 2 > Layer 3
```

---

## 3. 权限分级标尺

```yaml
permission_levels:
  # ═══ 绿灯（低风险）— 自主执行 ═══
  green:
    description: "只读操作 + 创建新文件"
    auto_approve: true
    operations:
      - read_file
      - search_file
      - search_content
      - list_dir
      - send_message
      - write_to_file                    # 创建新文件（workspace内）
    enforcement_layer: [3]              # 仅行为层约束

  # ═══ 黄灯（中风险）— 通知后执行 ═══
  yellow:
    description: "修改已有文件 + 运行构建/测试命令"
    auto_approve: false
    require_notification: true
    operations:
      - replace_in_file                 # 修改已有文件
      - execute_command: [build_cmds]   # 编译、打包
      - execute_command: [test_cmds]    # 测试
    enforcement_layer: [2, 3]           # Harness逻辑层 + 行为层
    notification: |
      send_message(type="message", recipient="main",
        event_type="risk_notification",
        level="yellow",
        operation="{具体操作}",
        reason="{为什么需要这个操作}")
    rule: "不阻塞，但必须通知。orchestrator 可在下一轮询拦截"

  # ═══ 红灯（高风险）— 需审批 ═══
  red:
    description: "删除文件 + 修改配置 + 破坏性命令"
    auto_approve: false
    require_approval: true
    operations:
      - delete_file
      - replace_in_file: [config_files]
      - execute_command: [dangerous_cmds]
    enforcement_layer: [1, 2, 3]        # 平台物理层 + Harness逻辑层 + 行为层
    approval_flow: |
      send_message(type="message", recipient="main",
        event_type="risk_approval_request",
        level="red",
        operation="{具体操作}",
        impact="{影响分析}",
        rollback="{回滚方案}")
      等待 orchestrator 响应：
        ├── approved → 继续执行
        ├── rejected → 放弃该操作，调整方案
        └── timeout (60s) → 放弃该操作，send_message 报告
    config_files:
      - "package.json"
      - "tsconfig.json"
      - ".eslintrc.*"
      - "vite.config.*"
      - "pyproject.toml"
      - "docker-compose.yml"
      - ".env*"
    dangerous_cmds:
      - "npm publish"
      - "git push"
      - "rm -rf"
      - "DROP TABLE"
      - "DELETE FROM"
      - "docker build"

  # ═══ 超红灯（禁区）— 禁止执行 ═══
  prohibited:
    description: "绝对不允许的操作"
    operations:
      - "修改 workspace 之外的文件"
      - "安装全局 npm/pip 包"
      - "硬编码敏感信息到代码中"
    enforcement_layer: [1, 3]           # 平台物理层（工具移除）+ 行为层
    violation_handling: |
      立即 send_message(type="message", recipient="main",
        event_type="permission_violation",
        operation="{尝试的操作}")
      跳过该操作，继续 plan 中的下一步
```

---

## 4. 执行命令白名单

```yaml
command_whitelist:
  # 包管理
  - npm install
  - npm run build
  - npm run test
  - npm run lint
  - pnpm install
  - pnpm run build
  - pnpm run test
  - yarn install
  - yarn build
  - yarn test

  # 编译检查
  - tsc --noEmit
  - python -m py_compile
  - python -m pytest
  - python -m black --check

  # 代码质量
  - eslint
  - prettier --check
  - pylint
  - mypy

  # 项目特定（由 orchestrator 在 spawn 时追加）
  extra: []
```

---

## 5. 各 Agent 权限特化

```yaml
agent_specialization:
  pm-coder:
    execute_phase:
      blocked_tools: [delete_file]
      yellow_tools: [replace_in_file]
      command_whitelist: [全量]          # 见上 command_whitelist

  pm-runner:
    execute_phase:
      green_tools: [task, team_create, team_delete, send_message]
      yellow_tools: [write_to_file, replace_in_file]
      blocked_tools: [delete_file]
      command_whitelist: [clawhub]       # 仅 clawhub CLI

  pm-researcher:
    # researcher 始终 plan 模式，无 execute_phase
    explore_phase_only: true
    mode: "plan"
    blocked_tools: [write_to_file, replace_in_file, execute_command, delete_file]

  pm-writer:
    execute_phase:
      green_tools: [write_to_file]       # 文档创建是绿灯
      yellow_tools: [replace_in_file]
      blocked_tools: [delete_file, execute_command]

  pm-analyst:
    explore_phase_only: true
    mode: "plan"
    allowed_extra: [write_to_file]        # 允许创建分析文档

  pm-planner:
    explore_phase_only: true
    mode: "plan"
    allowed_extra: [write_to_file]        # 允许创建规划文档

  pm-designer:
    execute_phase:
      green_tools: [write_to_file, image_gen]
      yellow_tools: [replace_in_file]
      blocked_tools: [delete_file, execute_command]

  pm-orchestrator:
    mode: "default"                       # 不自动接受编辑
    allowed_tools: [read_file, search_file, search_content, list_dir, send_message, task]
    blocked_tools: [write_to_file, replace_in_file, execute_command, delete_file]

  pm-backlog-manager:
    mode: "default"
    allowed_extra: [write_to_file, replace_in_file]  # 允许管理 backlog 文档
```

---

## 6. 权限变更审计

所有权限相关的决策必须审计：

```yaml
audit_events:
  - event_type: risk_notification
    detail: {agent, operation, risk_level, reason}
    
  - event_type: risk_approval_request
    detail: {agent, operation, risk_level, impact, rollback}
    
  - event_type: approval_result
    detail: {approver, approved, reason}
    
  - event_type: permission_violation
    detail: {agent, attempted_operation, blocked_by_layer}
```

---

*Ironforge Permission Framework v1.0 — 2026-04-23*
