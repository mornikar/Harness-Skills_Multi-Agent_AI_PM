# 上下文工程（Context Engineering）— Harness 公共层

> 从原 coder-harness 模块二抽取并泛化。
> 定义所有 Agent 共享的上下文分层、预算控制和噪声过滤规范。

---

## 1. 上下文分层策略

```yaml
context_layers:
  # ═══ Layer 1: 常驻上下文（Hot Memory）═══
  layer_1_hot:
    budget: "≤ 3000 tokens"
    contents:
      - "角色身份 + 核心职责（~200 tokens）"
      - "当前任务的 Goal success_criteria + constraints（~500 tokens）"
      - "可用 Skills Catalog（Tier 1 列表，~300 tokens）"
      - "风险分级权限表（~200 tokens）"
      - "当前执行阶段标识（~50 tokens）"
    rule: "精简到极致，只保留决策所需的最少信息"

  # ═══ Layer 2: 按需上下文（Working Memory）═══
  layer_2_working:
    budget: "≤ 15000 tokens"
    contents:
      - "完整 SKILL.md（~2000 tokens）"
      - "项目 HEARTBEAT 状态摘要（~1000 tokens）"
      - "上游任务 HEARTBEAT 核心结论（~500 tokens/task）"
      - "context_pool/tech_stack.md（~1000 tokens）"
      - "context_pool/architecture.md（~1000 tokens）"
      - "plan.md（当前任务规划，~1500 tokens）"
      - "Domain Skills（按需，每个 ~2000 tokens）"
    rule: "启动时批量加载，执行中如感觉冗余则压缩"

  # ═══ Layer 3: 引用上下文（Cold Reference）═══
  layer_3_cold:
    contents:
      - "references/ 目录下的所有文件"
      - "pm-core/references/recovery-recipes.md"
    rule: "只在 prompt 中给出路径提示，Agent 自行判断何时读取"
```

---

## 2. 噪声过滤规范

```yaml
noise_filtering:
  # 执行命令时的输出截断
  execute_command_output:
    max_lines: 50
    truncation_message: "... (输出已截断，保留最后 N 行)"
    smart_extraction:
      test_output:
        extract_pattern: "(PASS|FAIL|ERROR|passed|failed)"
        summary_template: "测试: {passed} passed, {failed} failed"
      build_output:
        extract_pattern: "(error|Error|ERROR|warning|Warning)"
        summary_template: "构建: {errors} errors, {warnings} warnings"
      install_output:
        extract_pattern: "(added|removed|changed|up to date)"
        summary_template: "安装: {added} added, {removed} removed"

  # 文件读取时的选择性加载
  file_reading:
    large_file_threshold: 500             # 超过500行的文件，只读相关部分
    strategy: "先 search_content 定位行号 → read_file(offset, limit)"
    skip_patterns:
      - "*.min.js"
      - "*.map"
      - "node_modules/**"
      - "dist/**"
      - "*.lock"
```

---

## 3. 上下文预算控制

```yaml
context_budget:
  # 轮次消耗预估（粗粒度）
  estimated_tokens_per_turn:
    read_file_small: 500
    read_file_large: 3000
    search_content: 800
    write_to_file: 1500
    replace_in_file: 1000
    execute_command: 2000
    send_message: 300

  # 上下文健康度监控
  health_check:
    trigger: "每完成一个 plan step 后"
    action: |
      1. 评估已用轮次 / max_turns
      2. 如使用率 > 60% → 压缩 Layer 2
      3. 如使用率 > 80% → 启动交接流程（HANDOFF）
    escalation: "使用率 > 90% → 立即 HANDOFF，不再执行新操作"
    
  # 默认交接触发
  handoff_trigger: "70% 轮次使用"
```

---

*Ironforge Base: Context Engineering v1.0 — 2026-04-23*
