# 上下文自适应协议（Context Protocol）

> v3 核心新增协议。定义了"有上下文时对齐，没有时降级"的自适应机制。
> 使每个 Skill 可以独立运行，不依赖前置流程产出。
>
> 路径变量定义见 `pm-core/platform-adapter.md`。

---

## 1. 上下文发现

每个 Agent 启动时，自动执行上下文发现：

```yaml
discovery:
  step_1:
    action: "扫描工作目录"
    scan_paths:
      - "{context_pool}/"
      - "{context_root}/"
      - "docs/"
      - "src/"
    
  step_2:
    action: "检查关键上下文文件"
    check_files:
      - path: "{context_pool}/goal.md"
        provides: "目标定义 + 成功标准 + 约束条件"
      - path: "{context_pool}/requirements.md"
        provides: "功能需求清单 + 非功能需求"
      - path: "{context_pool}/product.md"
        provides: "产品定义 + 目标用户 + 核心价值"
      - path: "{context_pool}/tasks.md"
        provides: "任务拆解 + 模块列表 + 依赖DAG"
      - path: "{context_pool}/modules.md"
        provides: "模块定义 + 功能点 + 验收标准"
      - path: "{context_pool}/dependency-dag.md"
        provides: "依赖关系图"
      - path: "{context_pool}/prototype/"
        provides: "原型文件（方向指标）"
      - path: "{context_pool}/backlog.md"
        provides: "需求池 + 优先级分类"
      - path: "{context_pool}/mvp-definition.md"
        provides: "MVP范围 + 验收标准"
    
  step_3:
    action: "汇总发现结果"
    output: "available_contexts[] — 已发现的上下文文件列表"
```

---

## 2. 上下文等级

根据发现结果，确定运行模式：

### FULL — 全链路模式

```yaml
context_level: FULL
requires:
  - goal.md
  - requirements.md
  - tasks.md
  - modules.md
behavior:
  - "严格对齐所有文档，按流程执行"
  - "对照 goal.md 的 success_criteria 验收"
  - "对照 modules.md 的验收标准自检"
  - "对照 prototype/ 对齐UI/交互"
use_when: "在编排流程内运行，前置Agent已产出完整文档"
```

### PARTIAL — 部分模式

```yaml
context_level: PARTIAL
requires:
  - "至少一个 context 文件存在"
behavior:
  - "已有文档正常对齐"
  - "缺失文档从用户输入推导"
  - "不因缺少文档而拒绝执行"
  - "在 HEARTBEAT 记录哪些文档缺失"
use_when: "部分前置Agent已完成，或用户手动提供了部分文档"
```

### MINIMAL — 轻量模式

```yaml
context_level: MINIMAL
requires: []
behavior:
  - "用户指令即需求，跳过文档对齐步骤"
  - "不要求 goal.md / requirements.md 等前置产出"
  - "自建轻量验收标准（基于用户指令推导）"
  - "直接向用户汇报结果，不依赖消息上报"
use_when: "独立运行，无编排流程，无前置Agent产出"
```

---

## 3. 降级规则

```yaml
degradation:
  FULL_to_PARTIAL:
    trigger: "缺少部分 context 文件"
    action: "已有文档正常对齐，缺失文档从用户输入推导"
    example: "有 goal.md 但没有 tasks.md → 按 goal.md 对齐，任务拆解自行推导"
  
  PARTIAL_to_MINIMAL:
    trigger: "缺少所有 context 文件"
    action: "用户指令即需求，跳过文档对齐步骤"
    example: "新项目目录，没有任何上下文池 → 直接接收用户指令"
```

---

## 4. 升级规则

```yaml
upgrade:
  MINIMAL_to_PARTIAL:
    trigger: "运行过程中产生了 context 文件"
    action: "自动升级上下文等级，后续步骤对齐新产出"
    example: "pm-analyst 产出了 goal.md → 后续 pm-coder 按 goal.md 对齐"
  
  PARTIAL_to_FULL:
    trigger: "所有必需 context 文件齐全"
    action: "升级为全链路模式，严格对齐所有文档"
    example: "编排流程中所有前置Agent已完成 → 按完整标准验收"
```

---

## 5. 各 Skill 的上下文等级要求

| Skill | 支持独立运行 | 最低上下文等级 | FULL 需要的文档 |
|-------|:-----------:|:------------:|---------------|
| pm-coder | ✅ | MINIMAL | goal.md + requirements.md + tasks.md + modules.md |
| pm-researcher | ✅ | MINIMAL | goal.md + requirements.md |
| pm-writer | ✅ | MINIMAL | goal.md + requirements.md + 产品定义 |
| pm-analyst | ✅ | MINIMAL | — (自身产出 goal.md) |
| pm-planner | ✅ | PARTIAL | goal.md + requirements.md |
| pm-designer | ✅ | PARTIAL | goal.md + requirements.md + modules.md |
| pm-backlog-manager | ✅ | MINIMAL | — (自身产出 backlog.md) |
| pm-runner | ✅ | PARTIAL | tasks.md + modules.md + dependency-dag.md |
| pm-orchestrator | ✅ | MINIMAL | — (编排全流程) |

---

## 6. 上下文协议注入方式

### 独立运行时

Agent 启动时自动执行：
```
1. 扫描 {context_pool}/ → 确定上下文等级
2. 根据 SKILL.md 的 standalone 配置 → 确定行为模式
3. 选择对应的工作流程分支（MINIMAL/PARTIAL/FULL）
```

### 编排运行时

编排模板指定上下文等级：
```yaml
# orchestrations/code-only.yaml
flow:
  phase_1:
    agent: pm-coder
    context_level: MINIMAL  # 编排模板可显式指定
```

如果编排模板未指定，Agent 自行发现上下文等级。

---

## 7. 与 v2 的兼容性

v2 的全链路流程自动满足 FULL 上下文等级：
- Phase 0 产出 backlog.md + mvp-definition.md
- Phase 1 产出 goal.md + requirements.md + tasks.md + modules.md
- Phase 2 产出 prototype/
- Phase 3 所有子Agent自动获得 FULL 上下文

v3 新增的独立运行模式使用 MINIMAL/PARTIAL，不破坏 v2 流程。
