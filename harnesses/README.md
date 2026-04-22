# Harness 定义目录（v2 架构重构版）

> Harness 是子Agent的**执行载体**，定义运行环境、绑定Skills、通信配置。
> v2 核心变革：Harness 现在是**高解耦的方向盘**，每个 Agent 有不同的方向盘参数。
> 与 Skill（知识包/行为规范）互补：Skill 定义"怎么做"，Harness 定义"在哪做、用什么做、不能做什么"。

## 文件说明

| 文件 | 对应Agent | 说明 |
|------|----------|------|
| `orchestrator-harness.md` | pm-orchestrator | 瘦身后主控器：监听+同步+收集+决策 |
| `analyst-harness.md` | pm-analyst | **新增** 需求澄清专家 |
| `planner-harness.md` | pm-planner | **新增** 任务规划专家 |
| `designer-harness.md` | pm-designer | **新增** 原型设计专家 |
| `runner-harness.md` | pm-runner | **新增** 执行调度专家（继承旧orchestrator调度职责） |
| `coder-harness.md` | pm-coder | 编程执行子Agent（v2: 必须对齐原型） |
| `researcher-harness.md` | pm-researcher | 信息检索子Agent |
| `writer-harness.md` | pm-writer | 内容输出子Agent |

## 核心概念：Harness = 高解耦的方向盘

> **核心隐喻**：Harness 不是统一的模具，而是每个 Agent 专属的方向盘。
> 方向盘控制着：权限边界、工具集、约束条件、策略规则、钩子集。
> 不同角色的方向盘完全不同，确保每个 Agent 只能做它该做的事。

### 方向盘差异化配置

| Agent | 方向盘模式 | 核心权限 | 核心约束 |
|-------|----------|---------|---------|
| **pm-analyst** | default（需用户交互） | 只读+创建需求文档 | 不产出代码；至少3轮追问 |
| **pm-planner** | default | 只读+创建规划文档+clawhub检查 | DAG无循环；模块≤5功能点 |
| **pm-designer** | acceptEdits | 读写原型文件+image_gen | 覆盖全部核心功能；语义化命名 |
| **pm-runner** | acceptEdits | spawn+send_message+clawhub | 不自行决定打回；对齐原型 |
| **pm-coder** | acceptEdits | 读写代码+测试+调试 | 规划优先；对齐原型；H1-H9钩子 |
| **pm-researcher** | acceptEdits | 只读+搜索+创建报告 | 调研是终端操作，不可委托 |
| **pm-writer** | acceptEdits | 只读+创建文档 | 文档与上游一致 |

### Skill vs Harness

| 维度 | Skill | Harness |
|------|-------|---------|
| **本质** | 知识包（SKILL.md + references/） | 执行载体（方向盘） |
| **类比** | SOP 手册 | 带定制工具箱的工作台 |
| **定义内容** | 角色定位、工作流程、输出模板、禁止事项 | spawn配置、工具绑定、通信协议、Skill加载策略 |
| **文件位置** | `{agent}/SKILL.md` | `harnesses/{agent}-harness.md` |
| **注入方式** | 通过 prompt 参数写入 | 通过 task 工具的参数配置 |
| **生命周期** | 项目级（不随任务变化） | 任务级（每次 spawn 重新配置） |

### Guides/Sensors 双闭环融入方向盘

```
┌──────────────────────────────────────────────────────────┐
│               Harness = 方向盘 + 双闭环                   │
│                                                          │
│  Guides（前馈控制）          Sensors（反馈控制）            │
│  在行动之前注入标准            在行动之后检查结果            │
│                                                          │
│  • SKILL.md                 • 计算型传感器（H1-H9钩子）    │
│  • Goal 标准                • 轻量自检（关键词匹配）        │
│  • code-standards           • 推理型传感器（orch验收）      │
│  • review-protocol                                       │
│  • 原型对齐约束（v2新增）                                  │
└──────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────┐
│                    Harness                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│  │ 运行环境    │  │ Skill绑定   │  │ 通信配置    │  │
│  │ spawn参数   │  │ 加载策略   │  │ HEARTBEAT  │  │
│  │ mode/turns │  │ 注入方式   │  │ send_msg   │  │
│  │ team_name  │  │ references │  │ recipient  │  │
│  └────────────┘  └────────────┘  └────────────┘  │
└──────────────────────────────────────────────────┘
         ↓ 绑定           ↓ 加载          ↓ 使用
┌────────────────┐ ┌──────────────┐ ┌──────────────┐
│  task 工具      │ │  SKILL.md    │ │  HEARTBEAT   │
│  (spawn子Agent) │ │  (行为规范)   │ │  (共享记忆)  │
└────────────────┘ └──────────────┘ └──────────────┘
```
