# Agent 生命周期规范（Agent Lifecycle）

> v3 核心新增协议。定义每个 Skill 独立运行时必须遵循的启动→执行→交付流程。
> 确保所有 Agent 无论独立运行还是编排运行，都有一致的行为规范。
>
> 路径变量和操作映射见 `pm-core/platform-adapter.md`。

---

## 1. 生命周期三阶段

```yaml
agent_lifecycle:
  # ─── 阶段1：启动 ───
  boot:
    step_1_context_discovery:
      action: "读取 pm-core/context-protocol → 发现上下文等级"
      detail: |
        扫描 {context_pool}/ 目录
        检查关键文件是否存在
        确定 FULL / PARTIAL / MINIMAL 等级
    
    step_2_skill_loading:
      action: "读取自身 SKILL.md → 确定行为模式"
      detail: |
        如有 standalone-prompt.md → 加载独立运行prompt
        如无 → 加载标准 SKILL.md（编排模式）
    
    step_3_mode_selection:
      action: "根据上下文等级，选择工作流程分支"
      detail: |
        MINIMAL → 跳过文档对齐，直接接收用户指令
        PARTIAL → 对齐已有文档，缺失部分从用户输入推导
        FULL → 严格对齐所有文档，按流程执行
  
  # ─── 阶段2：执行 ───
  execute:
    principles:
      - "按 SKILL.md 工作流程执行（根据上下文等级选择分支）"
      - "遵循 default-skills 行为纪律（DS1-DS6）"
      - "出错时按 recovery-recipes 恢复"
      - "定期更新 HEARTBEAT（如已创建）"
    
    adaptation:
      MINIMAL:
        - "不要求 goal.md / requirements.md 等前置产出"
        - "自建轻量验收标准"
        - "直接向用户汇报进展"
        - "不使用消息上报（无上游Agent）"
      
      PARTIAL:
        - "已有文档正常对齐"
        - "缺失文档从用户输入推导"
        - "可通知已知Agent"
      
      FULL:
        - "严格对齐所有前置文档"
        - "按 Goal success_criteria 验收"
        - "通过消息通知上游Agent"
  
  # ─── 阶段3：交付 ───
  deliver:
    MINIMAL:
      - "产出物写入工作目录"
      - "直接向用户汇报结果"
      - "说明修改了什么、为什么修改"
      - "不创建 HEARTBEAT（轻量）"
    
    PARTIAL:
      - "产出物写入工作目录"
      - "通知关联Agent（如有）"
      - "可选创建 HEARTBEAT"
    
    FULL:
      - "产出物写入 {context_pool}/"
      - "通知上游Agent"
      - "必须更新 HEARTBEAT"
      - "产出物必须对齐 goal.md 验收标准"
```

---

## 2. 独立运行 vs 编排运行

| 维度 | 独立运行 | 编排运行 |
|------|---------|---------|
| **触发方式** | 用户直接调用 | orchestrator / runner 调度 |
| **上下文等级** | MINIMAL（默认）| FULL（默认） |
| **Prompt来源** | standalone-prompt.md | SKILL.md + 平台执行配置 |
| **验收标准** | 自建轻量标准 | Goal success_criteria |
| **通信方式** | 直接向用户汇报 | 消息上报 |
| **HEARTBEAT** | 可选 | 必须 |
| **上下文写入** | 工作目录 | {context_pool}/ |
| **错误升级** | 通知用户 | 消息上报 orchestrator |

---

## 3. 模式切换

Agent 在运行过程中可动态切换模式：

```yaml
mode_switching:
  # 独立 → 编排（用户后续决定走全流程）
  standalone_to_orchestrated:
    trigger: "用户在独立运行后决定走全流程"
    action: |
      1. 保存当前产出到 {context_pool}/
      2. 通知 orchestrator 接管
      3. orchestrator 从当前状态继续编排
  
  # 编排 → 独立（流程中断，但某个Agent需继续工作）
  orchestrated_to_standalone:
    trigger: "编排流程中断/暂停，但Agent需完成当前任务"
    action: |
      1. 切换为 MINIMAL 模式
      2. 直接向用户汇报
      3. 产出物写入工作目录（非 {context_pool}/）
```

---

## 4. 启动检查清单

每个 Agent 启动时，隐式执行以下检查：

```yaml
boot_checklist:
  - ✅ "上下文等级已确定（FULL/PARTIAL/MINIMAL）"
  - ✅ "工作流程分支已选择"
  - ✅ "验收标准已明确（对齐文档 or 自建）"
  - ✅ "输出目标已确定（{context_pool}/ or 工作目录）"
  - ✅ "通信方式已确定（消息上报 or 直接汇报用户）"
```
