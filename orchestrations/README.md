# Orchestrations — 编排模板目录（v3 架构）

> 编排层是 v3 的核心创新：预定义的流程模板，按需组合 Skills。
> 编排是**可选的**，用户可以只用单个 Skill，也可以选择编排模板一键启动特定流程。

## 核心理念

```
v2: 流程驱动 → Skill 绑在固定流程上
v3: 能力驱动 → Skill 独立可用 → 编排是可选的组合层
```

## 预定义编排

| 编排模板 | 文件 | 使用场景 | 加载的 Skills |
|---------|------|---------|-------------|
| **code-only** | `code-only.yaml` | 改Bug、加功能、重构 | pm-coder |
| **research-only** | `research-only.yaml` | 技术调研、竞品分析 | pm-researcher |
| **analysis-only** | `analysis-only.yaml` | 需求梳理+任务拆解 | pm-analyst + pm-planner |
| **doc-only** | `doc-only.yaml` | 写PRD、技术文档 | pm-writer |
| **full-pipeline** | `full-pipeline.yaml` | 从0到1完整开发 | 全部9个 Skills |

## 自定义编排

用户可在 `custom/` 目录下创建自己的编排模板：

```yaml
# custom/my-workflow.yaml 示例
name: my-workflow
description: "我的自定义流程"
skills_required:
  - pm-analyst
  - pm-coder
flow:
  phase_1:
    agent: pm-analyst
    input: user_requirements
    output: [goal.md, requirements.md]
  phase_2:
    agent: pm-coder
    input: [goal.md, user_instruction]
    context_level: PARTIAL
    output: code_changes
```

## 编排模板格式规范

```yaml
name: "编排名称"
description: "一句话描述"
trigger_words: ["触发词1", "触发词2"]  # 用于自动匹配

skills_required:
  - skill-name-1
  - skill-name-2

flow:
  phase_N:
    agent: skill-name
    input: [输入来源]
    context_level: MINIMAL | PARTIAL | FULL  # 可选，默认自动发现
    output: [产出物列表]
    spawns: [子Agent列表]  # 仅 runner 需要
    parallel: true | false  # 是否并行
```
