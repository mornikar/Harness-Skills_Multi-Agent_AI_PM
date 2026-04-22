# AI产品经理多Agent Skill体系

> v2 架构重构版 — 五阶段流程 + 三段式解耦 + Harness 高解耦方向盘

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                orchestrator（瘦身后：只做监听+同步+收集）          │
│                                                                  │
│  职责：上下文同步 · 结果收集 · 全局监听 · 阶段切换决策            │
│        跨阶段打回路由 · 用户交互 · 复盘与经验沉淀                 │
│                                                                  │
│  不做：需求澄清、任务拆解、Skills安装、Agent调度、编码、文档       │
└───────────────────────────────┬─────────────────────────────────┘
                                │
   ┌────────────────────────────┼────────────────────────────┐
   ▼                            ▼                            ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  第一阶段     │        │  第二阶段     │        │  第三阶段     │
│  需求→分析→拆解│       │  原型+UI定型  │        │  架构→开发→测试│
│              │        │              │        │              │
│ pm-analyst   │──────→ │ pm-designer  │──────→ │ pm-runner    │
│ (需求澄清)   │        │ (原型设计)    │        │ (调度执行)    │
│      ↓       │        │      ↓       │        │  ↓  ↓  ↓     │
│ pm-planner   │        │  原型验收     │        │ coder review │
│ (颗粒化拆解) │        │ (模块化校准)  │        │ researcher   │
│      ↓       │        │              │        │ writer       │
│ Skills分析   │        │              │        │              │
│ +依赖图DAG   │        │              │        │              │
└──────────────┘        └──────────────┘        └──────────────┘
       ↑                       ↑                       ↑
       │          ┌────────────────────────┐           │
       └──────────│  第四阶段：打回循环      │───────────┘
                  │  模块化验收+最小回退     │
                  └────────────────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │  第五阶段：整合交付      │
                  │  模块化讨论验收+复盘     │
                  └────────────────────────┘
```

## 核心特性

### 1. 五阶段流程

| 阶段 | 名称 | 主导 Agent | 关键产出 |
|------|------|-----------|---------|
| P1 | 需求→分析→拆解 | pm-analyst → pm-planner | goal.md + modules.md + dependency-dag.md |
| P2 | 原型+UI定型 | pm-designer | 可交互原型 + 组件树 + 页面路由 |
| P3 | 架构→开发→测试 | pm-runner → subAgents | 可运行代码 + 模块测试 + 文档 |
| P4 | 打回循环 | orchestrator（决策） | 最小回退到必要阶段 |
| P5 | 整合交付 | orchestrator | 模块化验收 + 复盘报告 |

### 2. 三段式解耦

orchestrator 不再大包大揽，职责分发到三个阶段 Agent：

- **pm-analyst**：需求澄清 + Goal 构建（追问 ≥3 轮）
- **pm-planner**：颗粒化拆解 + 依赖 DAG + Skills 分析
- **pm-runner**：调度执行 + 策略引擎 + 健康监控

### 3. 原型作为校准基石（第二阶段创新）

编码前先做原型，原型经 orchestrator + 用户确认后才进入开发，避免"编码完了发现方向不对"的浪费。

### 4. 模块化验收 + 打回循环

- 验收以模块为单位（不是任务级），更有实际意义
- 跨阶段打回由 orchestrator 决策路由，规则是"最小必要回退点"
- pm-runner 只能上报，不能自行决定打回

### 5. Harness = 高解耦方向盘

每个 Agent 的 Harness 有差异化配置（mode、max_turns、allowed_tools、constraints），详见 `harnesses/README.md`。

### 6. 上下文池（Context Pool）

全局共享的项目上下文，细粒度访问权限控制：

| 文件 | orchestrator | analyst | planner | designer | runner | subAgents |
|------|-------------|---------|---------|----------|--------|-----------|
| goal.md | 读写 | 读写 | 只读 | 只读 | 只读 | — |
| product.md | 读写 | 只读 | 只读 | 只读 | — | — |
| modules.md | 读写 | — | 读写 | 只读 | 只读 | — |
| dependency-dag.md | 读写 | — | 读写 | — | 读写 | — |
| prototype/ | 读写 | — | — | 读写 | 只读 | 只读 |
| architecture.md | 读写 | — | — | — | 读写 | 只读 |
| progress/*.md | 读写 | — | — | — | 读写 | 读写 |
| shared/* | 读写 | 只读 | 只读 | 只读 | 读写 | 读写 |

### 7. HEARTBEAT 双层记忆

- **项目级**：全局进度、关键决策、风险追踪
- **任务级**：单任务状态、上下文、中间结果

## 目录结构

```
AI_PM_SKills/
├── README.md                    # 本文件
├── ARCHITECTURE.md              # 架构详细说明
├── QUICKSTART.md               # 快速开始指南
│
├── pm-orchestrator/            # 主控器（瘦身后）
│   └── SKILL.md
│
├── pm-analyst/                 # 需求澄清 Agent（新增）
│   └── SKILL.md
│
├── pm-planner/                 # 颗粒化拆解 Agent（新增）
│   └── SKILL.md
│
├── pm-designer/                # 原型设计 Agent（新增）
│   └── SKILL.md
│
├── pm-runner/                  # 调度执行 Agent（新增）
│   └── SKILL.md
│
├── pm-coder/                   # 编程执行 Agent
│   └── SKILL.md
│
├── pm-researcher/              # 信息检索 Agent
│   └── SKILL.md
│
├── pm-writer/                  # 内容输出 Agent
│   └── SKILL.md
│
├── harnesses/                  # Harness 配置（方向盘）
│   ├── README.md               # Harness 概念说明
│   ├── orchestrator-harness.md
│   ├── analyst-harness.md      # 新增
│   ├── planner-harness.md      # 新增
│   ├── designer-harness.md     # 新增
│   ├── runner-harness.md       # 新增
│   └── coder-harness.md
│
├── shared/                     # 共享资源
│   └── references/             # 参考资料
│       ├── coding-standards.md
│       ├── api-design-guide.md
│       └── security-checklist.md
│
└── scripts/                    # 工具脚本
```

## Agent 角色体系

| Agent | 核心职责 | 触发词 | 阶段 |
|-------|---------|--------|------|
| pm-orchestrator | 上下文同步·结果收集·阶段切换·打回路由 | 开发、实现、做一个、搭建 | 全局 |
| pm-analyst | 需求澄清·约束提取·Goal构建 | 需求澄清、范围确认、Goal定义 | P1 |
| pm-planner | 颗粒化拆解·依赖DAG·Skills分析 | 任务拆解、模块化、规划 | P1 |
| pm-designer | 原型设计·组件树·交互流程 | 原型设计、UI设计、线框图 | P2 |
| pm-runner | 调度执行·策略引擎·健康监控 | 调度、执行、运行 | P3 |
| pm-coder | 编码·调试·重构·测试 | 编码、实现、开发、调试 | P3 |
| pm-researcher | 技术调研·竞品分析·方案选型 | 调研、对比、选型、分析 | P3 |
| pm-writer | PRD·技术文档·API文档·CHANGELOG | 文档、PRD、撰写、编写 | P3 |

## 五阶段工作流程

### 第一阶段：需求→分析→拆解

```
用户输入需求
    ↓
orchestrator spawn pm-analyst
    ↓
┌────────────────────────────────────────┐
│ pm-analyst 需求澄清                     │
│ - 读取 product.md 了解项目背景           │
│ - 与用户追问 ≥3 轮                      │
│ - 界定范围、提取约束                     │
│ - 构建 Goal 对象（goal.md）             │
│ - 输出：product.md + goal.md            │
└────────────────────────────────────────┘
    ↓
orchestrator 审核 goal.md → 通过则 spawn pm-planner
    ↓
┌────────────────────────────────────────┐
│ pm-planner 颗粒化拆解                   │
│ - 读取 goal.md + requirements.md        │
│ - 提取功能点 → 模块化分组                │
│ - 每模块功能点 ≤5，含验收标准            │
│ - 构建依赖 DAG（无环、单根、可达）       │
│ - Skills 需求分析 → 安装清单             │
│ - 输出：modules.md + dependency-dag.md  │
│         + tasks.md + skills-needed.md   │
└────────────────────────────────────────┘
    ↓
orchestrator 审核拆解结果 → 决定是否进入 P2
```

### 第二阶段：原型+UI定型

```
orchestrator spawn pm-designer
    ↓
┌────────────────────────────────────────┐
│ pm-designer 原型设计                    │
│ - 读取 goal.md + modules.md            │
│ - 设计交互流程（Mermaid 图）            │
│ - 构建组件树（语义化命名）               │
│ - 定义页面路由                          │
│ - 生成线框图（HTML+CSS 低保真可交互）    │
│ - 自检：功能覆盖·完整流程·路由覆盖       │
│ - 输出：prototype/ + 组件树文档          │
└────────────────────────────────────────┘
    ↓
orchestrator + 用户 原型验收
    ↓
原型成为开发与验收的校准基石 → 进入 P3
```

### 第三阶段：架构→开发→测试（阻塞，等 P1+P2 走完）

```
orchestrator spawn pm-runner
    ↓
┌────────────────────────────────────────┐
│ pm-runner 调度执行                      │
│                                        │
│ 1. 技术架构讨论                         │
│    - spawn 多个 coder 各提方案          │
│    - 上报 orchestrator 评审             │
│                                        │
│ 2. Skills 安装与验证                    │
│    - 按 skills-needed.md 安装           │
│                                        │
│ 3. DAG 批次调度开发                     │
│    - 按依赖图并行 spawn coder           │
│    - 每批次 spawn 提示注入原型对齐要求   │
│    - 策略引擎自动处理阻塞/恢复           │
│                                        │
│ 4. 模块测试                            │
│    - spawn coder 编写测试               │
│                                        │
│ 5. 文档编写                            │
│    - spawn writer 输出文档              │
│                                        │
│ 关键约束：                              │
│ - 不能自行决定打回，只能上报             │
│ - 子Agent 健康度 < 30 必须通知          │
│   orchestrator                         │
└────────────────────────────────────────┘
    ↓
orchestrator 监听 + 预准备
(可利用 P2 期间预分析技术方案)
```

### 第四阶段：打回循环

```
模块验收不通过
    ↓
┌────────────────────────────────────────┐
│ 打回路由决策（orchestrator 全局视角）    │
│                                        │
│ 原型不通过      → 回到 P1 需求澄清      │
│ 架构不可行      → 回到 P2 原型调整      │
│ 代码与原型不符  → 回到 P3 架构调整      │
│ 测试不通过      → 回到 P3 开发修复      │
│ 模块验收不通过  → 回到最小必要回退点     │
│                                        │
│ 规则：每次打回退到"最小必要回退点"       │
│       不可退回全部                      │
│ pm-runner 只能上报，不能自行决定打回     │
└────────────────────────────────────────┘
```

### 第五阶段：整合交付

```
┌────────────────────────────────────────┐
│ 模块化讨论验收                          │
│ - 多 Agent 以模块为单位交叉验收          │
│ - orchestrator 整合检查                 │
│ - 用户最终确认                          │
│                                        │
│ 复盘与经验沉淀                          │
│ - 经验 L1-L4 积累（微模式→独立 Skills） │
│ - 更新 HEARTBEAT 项目记忆               │
│ - 输出复盘报告                          │
└────────────────────────────────────────┘
```

## 使用场景

### 场景1：从0到1开发新产品

```
用户：帮我做一个个人知识管理工具，支持Markdown编辑和全文搜索

P1: pm-analyst 澄清需求 → pm-planner 拆解为模块
    ├── 编辑器模块、搜索模块、存储模块、UI框架模块
    └── 依赖DAG：存储→编辑器→搜索→UI框架

P2: pm-designer 设计原型
    ├── 编辑器页面、搜索页面、设置页面
    └── 组件树 + 交互流程 → 用户确认

P3: pm-runner 调度开发
    ├── T001 → pm-coder: 存储模块（IndexedDB）
    ├── T002 → pm-coder: 编辑器核心（依赖T001）
    ├── T003 → pm-researcher: 搜索技术选型
    ├── T004 → pm-coder: 搜索索引（依赖T001,T003）
    ├── T005 → pm-coder: UI框架（依赖T002）
    └── T006 → pm-writer: 用户手册

P4/P5: 模块验收 + 整合交付
```

### 场景2：功能迭代

```
用户：给现有App添加用户认证功能

P1: pm-analyst 澄清 → pm-planner 拆解
    ├── 认证模块（JWT）、登录页面、权限中间件
    └── DAG：认证→中间件→登录页

P2: pm-designer 设计登录页原型 + 权限流程图

P3: pm-runner 调度
    ├── T001 → pm-coder: JWT认证API
    ├── T002 → pm-coder: 权限中间件（依赖T001）
    ├── T003 → pm-coder: 登录页面（依赖T001）
    └── T004 → pm-writer: API文档更新
```

## 安装使用

### 1. 安装到 WorkBuddy

```powershell
# 复制到 WorkBuddy Skills 目录
Copy-Item -Path "AI_PM_SKills\*" `
  -Destination "$env:USERPROFILE\.workbuddy\skills\" -Recurse -Force
```

### 2. 开始使用

直接对 AI 说出你的需求：

- "帮我做一个记账App"
- "开发一个Markdown编辑器"
- "实现用户认证功能"

pm-orchestrator 会自动触发五阶段流程。

## 上下文池文件结构

```
.workbuddy/context_pool/
├── product.md              # 产品定义（analyst 写，其余只读）
├── goal.md                 # Goal 对象（analyst 写，其余只读）
├── requirements.md         # 需求清单（analyst 写，其余只读）
├── modules.md              # 模块清单（planner 写，其余只读）
├── dependency-dag.md       # 依赖DAG（planner 写，runner 读）
├── tasks.md                # 任务列表（planner 写，runner 读）
├── skills-needed.md        # Skills需求（planner 写，runner 读）
├── architecture.md         # 架构设计（runner 写，subAgents 读）
├── prototype/              # 原型文件（designer 写，runner/coder 读）
│   ├── wireframe.html
│   ├── component-tree.md
│   └── routes.md
├── decisions.md            # 决策记录（orchestrator 读写）
├── progress/               # 任务进度（runner/subAgents 读写）
│   ├── T001.md
│   └── ...
└── shared/                 # 共享数据
    ├── api-schema.json
    ├── db-schema.sql
    └── ui-mockups/
```

## HEARTBEAT.md 格式

```markdown
# Project Heartbeat

## 项目信息
- **项目名称**: MyApp
- **创建时间**: 2026-04-22
- **当前阶段**: P3（架构→开发→测试）
- **最后更新**: 2026-04-22 14:30

## 阶段状态

| 阶段 | 状态 | 主导Agent | 完成时间 |
|------|------|----------|---------|
| P1 | ✅ 完成 | pm-analyst → pm-planner | 10:00 |
| P2 | ✅ 完成 | pm-designer | 11:30 |
| P3 | 🔄 进行中 | pm-runner | — |
| P4 | ⏳ 待定 | orchestrator | — |
| P5 | ⏳ 待定 | orchestrator | — |

## 任务状态看板

| 任务ID | 模块 | Agent | 状态 | 进度 | 阻塞项 | 健康度 |
|-------|------|-------|------|------|--------|--------|
| T001 | 存储 | pm-coder | completed | 100% | — | 95 |
| T002 | 编辑器 | pm-coder | running | 60% | — | 80 |
| T003 | 搜索 | pm-coder | pending | 0% | 依赖T001 | — |

## 关键决策
- [2026-04-22] 选择Vue3 + Electron作为技术栈

## 风险与问题
- [MEDIUM] Electron打包体积较大 → 考虑使用Tauri替代

## 经验沉淀
- [L1] IndexedDB 异步操作需包装为 Promise
```

## 扩展开发

### 添加新的子Agent

1. 创建 `pm-{agent-name}/SKILL.md`
2. 在 `harnesses/` 下创建对应 `{agent-name}-harness.md`
3. 在 `pm-runner/SKILL.md` 中注册子Agent spawn 配置
4. 更新本 README 的 Agent 角色体系表

### 添加共享资源

1. 放入 `shared/references/`
2. 在相关 Agent 的 SKILL.md 中添加引用说明

## 最佳实践

1. **原型先行**：绝不跳过 P2，原型是校准基石
2. **模块粒度**：单模块功能点 ≤5，含验收标准
3. **DAG 无环**：依赖关系必须形成有向无环图
4. **最小回退**：打回时退到最小必要回退点，不退回全部
5. **策略引擎**：pm-runner 策略自动处理阻塞/恢复，不打断流程
6. **打回上报**：pm-runner 不能自行决定打回，只能上报 orchestrator
7. **追问 ≥3 轮**：pm-analyst 必须至少追问 3 轮才可结束需求澄清

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| v2.0 | 2026-04-22 | 架构重构：五阶段流程+三段式解耦+4个新Agent+Harness方向盘 |
| v1.0 | 2026-04-20 | 初始版本，7阶段流程+4个核心Agent |

## 许可证

MIT License
