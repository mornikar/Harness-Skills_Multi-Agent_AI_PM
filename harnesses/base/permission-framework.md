# 权限强制执行（Permission Framework）— Harness 公共层

> 从 `pm-core/security/permission-framework.md` 衍生。
> 定义所有 Agent 共享的权限执行配置和 spawn 模板。
> 各 Agent 特化层通过 `permission_override` 增减。

---

## 1. 三层权限执行模型

```yaml
permission_execution_model:
  layer_1_platform:
    description: "平台物理层 — 利用 spawn mode 物理阻断"
    rule: "Agent 的工具列表里根本没有被移除的工具，想做也做不到"
    
  layer_2_harness:
    description: "Harness 逻辑层 — 通知后执行，有监督窗口"
    rule: "允许执行但必须通知 orchestrator，orchestrator 可介入拦截"
    
  layer_3_skill:
    description: "SKILL.md 行为层 — 自律约束，作为兜底"
    rule: "编码规范和禁止事项，靠 Agent 理解执行"
```

---

## 2. 默认 Spawn 配置模板

```yaml
default_spawn_config:
  # ═══════════════════════════════════════
  # Phase A: 探索阶段 — 物理只读
  # ═══════════════════════════════════════
  explore_phase:
    mode: "plan"                           # WorkBuddy 原生只读模式
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

  # ═══════════════════════════════════════
  # Phase C: 执行阶段 — 分级放权
  # ═══════════════════════════════════════
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
      []                                   # 默认无额外禁止，特化层按需添加
```

---

## 3. 默认权限分级标尺

```yaml
default_permission_levels:
  green:
    auto_approve: true
    operations: [read_file, search_file, search_content, list_dir, send_message, write_to_file]
    
  yellow:
    auto_approve: false
    require_notification: true
    operations: [replace_in_file, execute_command(whitelist)]
    
  red:
    auto_approve: false
    require_approval: true
    operations: [delete_file, execute_command(non-whitelist), replace_in_file(config_files)]
    
  prohibited:
    operations: [workspace外操作, 全局包安装, 硬编码敏感信息]
```

---

## 4. 默认命令白名单

```yaml
default_command_whitelist:
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
  - tsc --noEmit
  - python -m py_compile
  - python -m pytest
  - python -m black --check
  - eslint
  - prettier --check
  - pylint
  - mypy
```

---

## 5. 通知协议

```yaml
notification_protocol:
  # 黄灯操作通知
  yellow_notification: |
    send_message(type="message", recipient="main",
      event_type="risk_notification",
      level="yellow",
      operation="{具体操作}",
      reason="{为什么需要这个操作}")
    
  # 红灯操作审批请求
  red_approval_request: |
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
      
  # 权限违规报告
  violation_report: |
    send_message(type="message", recipient="main",
      event_type="permission_violation",
      operation="{尝试的操作}")
    跳过该操作，继续 plan 中的下一步
```

---

## 6. 各 Agent 默认特化

```yaml
agent_defaults:
  pm-coder:
    execute_phase:
      blocked_tools: [delete_file]
      max_turns: 40
      
  pm-runner:
    execute_phase:
      green_add: [task, team_create, team_delete]
      blocked_tools: [delete_file]
      max_turns: 80
      
  pm-researcher:
    explore_phase_only: true
    mode: "plan"
    max_turns: 10
      
  pm-writer:
    execute_phase:
      blocked_tools: [delete_file, execute_command]
      max_turns: 35
      
  pm-analyst:
    explore_phase_only: true
    mode: "plan"
    allowed_extra: [write_to_file]
      
  pm-planner:
    explore_phase_only: true
    mode: "plan"
    allowed_extra: [write_to_file]
    max_turns: 15
      
  pm-designer:
    execute_phase:
      green_add: [image_gen]
      blocked_tools: [delete_file, execute_command]
      max_turns: 20
      
  pm-orchestrator:
    mode: "default"
    blocked_tools: [write_to_file, replace_in_file, execute_command, delete_file]
      
  pm-backlog-manager:
    mode: "default"
    allowed_extra: [write_to_file, replace_in_file]
```

---

*Ironforge Base: Permission Framework v1.0 — 2026-04-23*
