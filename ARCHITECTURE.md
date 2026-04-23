# AI PM Skills 架构详细说明

> v3.1 Ironforge — 三层架构 + 企业工程化（安全/稳定/可观测）+ 平台无关 + Skill独立运行 + 上下文自适应

## 1. 系统架构

### 1.1 整体架构图（v3 高解耦架构版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层                                       │
│                         (自然语言需求输入)                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     pm-orchestrator Harness（瘦身后）                         │
│                    (主控器：只做监听+同步+收集+决策)                           │
│                                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  阶段切换决策  │ │  打回路由    │ │  三角验证     │ │  复盘沉淀     │       │
│  │  - 审核拆解   │ │  - 最小回退  │ │  - 语义评估   │ │  - 模式识别   │       │
│  │  - 原型校验   │ │  - 路由决策  │ │  - 加权验收   │ │  - L1-L4分级 │       │
│  │  - 用户确认   │ │  - 次数限制  │ │  - 约束检查   │ │  - 经验沉淀   │       │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                        │
│  │  上下文同步   │ │  健康度监控  │ │  检查点管理   │                        │
│  │  - 全局监听   │ │  - 评分计算  │ │  - 快照创建   │                        │
│  │  - 结果收集   │ │  - 预警介入  │ │  - 崩溃恢复   │                        │
│  │  - 预准备链路 │ │  - 策略引擎  │ │  - 决策回溯   │                        │
│  └──────────────┘ └──────────────┘ └──────────────┘                        │
│                                                                              │
│  绑定Skill: pm-orchestrator/SKILL.md                                        │
│  工具: 团队管理 · 子Agent调度 · 消息通信 · 文件读写                           │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
     Phase 1 团队            Phase 2 团队             Phase 3 团队
           │                        │                        │
           ▼                        ▼                        ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  pm-analyst     │      │  pm-designer    │      │  pm-runner      │
│  Harness        │─────→│  Harness        │─────→│  Harness        │
│                 │      │                 │      │                 │
│ ┌─────────────┐│      │ ┌─────────────┐│      │ ┌─────────────┐│
│ │需求澄清      ││      │ │原型设计      ││      │ │架构讨论      ││
│ │Goal构建      ││      │ │线框图        ││      │ │Skills管理    ││
│ └─────────────┘│      │ └─────────────┘│      │ └──────┬──────┘│
│ ┌─────────────┐│      │ ┌─────────────┐│      │        │       │
│ │pm-planner   ││      │ │组件树       ││      │ ┌──────▼──────┐│
│ │颗粒化拆解    ││      │ │交互流       ││      │ │子Agent调度   ││
│ │依赖DAG      ││      │ │页面路由      ││      │ │coder×N      ││
│ │Skills分析   ││      │ └─────────────┘│      │ │researcher   ││
│ └─────────────┘│      └─────────────────┘      │ │writer       ││
└─────────────────┘                                │ └─────────────┘│
                                                   └─────────────────┘
                                   │
                    ┌──────────────────────────────────┐
                    │  Phase 4: 打回循环                 │
                    │  模块化验收 + 最小必要回退          │
                    │  打回决策权归 orchestrator          │
                    └──────────────────────────────────┘
                                   │
                    ┌──────────────────────────────────┐
                    │  Phase 5: 整合交付                 │
                    │  模块化讨论验收 + 复盘               │
                    └──────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              基础设施层                                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
│  │  上下文池     │ │ Skills仓库   │ │  HEARTBEAT   │ │  远程仓库    │       │
│  │ Context Pool │ │ Skill Registry│ │  v2 双格式   │ │  Skills源    │
│  │ +prototype/  │ │ +新Agent SK  │ │  YAML+MD    │ │  按平台适配  │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                        │
│  │ 🔒安全框架   │ │ 🛡️稳定性框架 │ │ 📊可观测框架 │                        │
│  │ 权限+审计    │ │ 检查点+熔断  │ │ 指标+追踪   │                        │
│  └──────────────┘ └──────────────┘ └──────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心概念：Skill vs Harness

| 维度       | Skill                        | Harness                                               |
| -------- | ---------------------------- | ----------------------------------------------------- |
| **本质**   | 知识包（SKILL.md + references/）  | 执行载体（运行环境配置）                                          |
| **类比**   | SOP 手册                       | 带工具箱的工作台                                              |
| **定义内容** | 角色定位、工作流程、输出模板、禁止事项          | spawn配置、工具绑定、通信协议、Skill加载策略                           |
| **文件位置** | `{skills_root}/pm-{name}/SKILL.md` | `harnesses/{agent}-harness.md`                        |
| **注入方式** | 通过 prompt 参数写入               | 通过平台工具的参数配置                                       |
| **生命周期** | 项目级（不随任务变化）                  | 任务级（每次创建子Agent重新配置）                                    |
| **平台绑定** | **无**（v3 平台无关）              | **有**（适配特定AI平台）                                      |
| **示例**   | `pm-coder/SKILL.md` 定义"编码规范" | `coder-harness.md` 定义"文件写入权限, 最多50轮" |

```
┌──────────────────────────────────────────────────┐
│                    Harness                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│  │ 运行环境    │  │ Skill绑定   │  │ 通信配置    │  │
│  │ 权限配置    │  │ 加载策略   │  │ HEARTBEAT  │  │
│  │ 模式/轮次   │  │ 注入方式   │  │ 消息通知    │  │
│  │ 团队名      │  │ references │  │ 接收方      │  │
│  └────────────┘  └────────────┘  └────────────┘  │
└──────────────────────────────────────────────────┘
         ↓ 绑定           ↓ 加载          ↓ 使用
┌────────────────┐ ┌──────────────┐ ┌──────────────┐
│  平台工具       │ │  SKILL.md    │ │  HEARTBEAT   │
│  (创建子Agent)  │ │  (行为规范)   │ │  (共享记忆)  │
└────────────────┘ └──────────────┘ └──────────────┘
```

### 1.3 平台抽象层（v3 新增）

v3 引入 `pm-core/platform-adapter.md`，将行为规范与特定AI平台解耦：

| 路径变量 | 说明 | 映射示例 |
|---------|------|---------|
| `{context_root}` | 项目上下文根目录 | `.workbuddy/`、`.cursor/`、`.cline/` |
| `{skills_root}` | Skills 安装目录 | `~/.workbuddy/skills/`、项目级 skills/ |

操作映射：

| 抽象操作 | 平台实现示例 |
|---------|-----------|
| 读取文件 {path} | `read_file(path)` |
| 写入文件 {path} | `write_to_file(path)` |
| 更新文件 {path} | `replace_in_file(path)` |
| 向 {recipient} 发送消息 | `send_message(recipient, ...)` |
| 创建子Agent {name} | `task(name, ...)` |
| 创建团队 {name} | `team_create(name)` |
| 清理团队 | `team_delete()` |

> SKILL.md 和 pm-core/ 中的所有文件只使用抽象操作，不直接调用平台API。

### 1.4 数据流图（v3 六阶段）

```
用户输入
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  orchestrator: 初始化上下文池 + 创建团队               │
└──────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────┐
│ Phase 0      │
│ pm-backlog   │
│ 需求池管理    │
│ 优先级排序    │
│ MVP定义      │
│ batch-plan   │
└──────┬───────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│ Phase 1      │     │              │
│ pm-analyst   │────→│ pm-planner   │
│ 需求澄清      │     │ 颗粒化拆解    │
│ goal.md      │     │ modules.md   │
│              │     │ DAG          │
└─────────────┘     │ skills-needed│
                    └──────┬───────┘
                           │
                    orchestrator 审核
                           │
                    ┌──────▼───────┐
                    │ Phase 2       │
                    │ pm-designer   │
                    │ 原型设计       │
                    │ wireframe.html│
                    │ component-tree│
                    └──────┬───────┘
                           │
                    orchestrator + 用户 原型校验
                           │
                    ┌──────▼───────┐
                    │ Phase 3       │
                    │ pm-runner     │
                    │ ┌───────────┐│
                    │ │架构讨论    ││
                    │ │Skills安装  ││
                    │ │功能开发    ││──→ pm-coder × N
                    │ │模块测试    ││──→ pm-researcher
                    │ │文档编写    ││──→ pm-writer
                    │ └───────────┘│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Phase 4       │
                    │ 打回循环       │
                    │ 模块化验收     │
                    │ 最小必要回退   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Phase 5       │
                    │ 整合交付       │
                    │ 模块化讨论验收  │
                    │ 复盘+经验沉淀  │
                    └──────────────┘
                           │
                           ▼
                      用户交付
```

## 2. 核心组件详解

### 2.1 上下文池（Context Pool）

#### 2.1.1 设计目标

- 提供全局共享的项目上下文
- 支持细粒度的访问控制
- 实现版本化的变更追踪
- 确保数据一致性

#### 2.1.2 数据结构

```typescript
interface ContextPool {
  // 项目级只读配置
  config: {
    product: ProductConfig;      // 产品定义
    requirements: Requirement[]; // 需求列表
    techStack: TechStack;        // 技术栈
    architecture: Architecture;  // 架构设计
  };

  // 动态决策记录
  decisions: Decision[];         // 关键决策历史

  // 任务执行状态
  tasks: {
    [taskId: string]: TaskProgress;
  };

  // 共享资源
  shared: {
    apiSchema?: OpenAPISpec;
    dbSchema?: SQLSchema;
    uiMockups?: DesignFile[];
    assets?: Asset[];
  };
}

interface TaskProgress {
  taskId: string;
  agent: string;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'blocked';
  progress: number;              // 0-100
  deliverables: Deliverable[];
  blockers: Blocker[];
  lastUpdate: Timestamp;
}
```

#### 2.1.3 文件映射

| 数据类型        | 文件路径                                        | 格式       | 访问模式                          | 维护者                     |
| ----------- | ------------------------------------------- | -------- | ----------------------------- | ----------------------- |
| **项目记忆**    | `{context_root}/HEARTBEAT.md`                | Markdown | orchestrator读写，子Agent只读       | orchestrator            |
| **任务记忆**    | `{context_root}/context_pool/progress/T{XXX}-heartbeat.md` | Markdown | orchestrator读写，子Agent读写（仅自己的） | 各子Agent                 |
| 产品定义        | `{context_root}/context_pool/product.md`     | Markdown | 只读                            | orchestrator            |
| 需求清单        | `{context_root}/context_pool/requirements.md` | Markdown | 只读                            | orchestrator            |
| 技术栈         | `{context_root}/context_pool/tech_stack.md`  | Markdown | 只读                            | orchestrator/researcher |
| 架构设计        | `{context_root}/context_pool/architecture.md` | Markdown | 只读                            | orchestrator            |
| 决策记录        | `{context_root}/context_pool/decisions.md`   | Markdown | 追加                            | orchestrator            |
| API规范       | `{context_root}/context_pool/shared/api-schema.json` | JSON     | 读写                            | coder/writer            |
| 数据库         | `{context_root}/context_pool/shared/db-schema.sql` | SQL      | 读写                            | coder                   |
| UI设计        | `{context_root}/context_pool/shared/ui-mockups/` | 图片/Figma | 读写                            | coder/writer            |
| HEARTBEAT模板 | `pm-core/templates/heartbeat-template.md`    | Markdown | 只读                            | -                       |

### 2.2 HEARTBEAT 机制

#### 2.2.1 设计目标

- 实时追踪项目整体状态
- 快速识别阻塞点和风险
- 提供决策历史追溯
- 支持项目复盘分析
- **跨会话记忆持久化**：Agent被重建后可通过HEARTBEAT快速恢复上下文
- **上下文压缩**：用结构化摘要替代无限增长的Chat History，防止上下文窗口爆掉
- **错误恢复**：任务出错时通过HEARTBEAT快速定位偏差，矫正执行方向

> **核心原则**：HEARTBEAT 是唯一的持久记忆载体。Chat History 只是临时的执行轨迹，可以随时丢弃。
> 类比：项目经理不看每一行 Git 提交记录（Chat History），只看每周周报（HEARTBEAT）。

#### 2.2.2 两级记忆架构

```
┌─────────────────────────────────────────────────────────────┐
│  项目级 HEARTBEAT ({context_root}/HEARTBEAT.md)              │
│  维护者: pm-orchestrator                                      │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 项目概览  │ │ 任务看板  │ │ 决策记录  │ │ 风险追踪  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
│  ┌──────────┐ ┌──────────┐                                  │
│  │ 文件索引  │ │ 变更日志  │                                  │
│  └──────────┘ └──────────┘                                  │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│ T001-heartbeat.md │ │ T002-heartbeat.md │ │ T003-heartbeat.md │
│ 维护者:researcher  │ │ 维护者:coder      │ │ 维护者:writer     │
│                   │ │                   │ │                   │
│ • 任务目标         │ │ • 任务目标         │ │ • 任务目标         │
│ • 执行进度         │ │ • 执行进度         │ │ • 执行进度         │
│ • 产出物清单       │ │ • 产出物清单       │ │ • 产出物清单       │
│ • 关键发现         │ │ • 代码统计         │ │ • 素材引用         │
│ • 依赖阻塞         │ │ • 依赖阻塞         │ │ • 一致性检查       │
└───────────────────┘ └───────────────────┘ └───────────────────┘
```

#### 2.2.3 记忆读写流程

```
子Agent启动时（记忆恢复）
    │
    ├──→ 读取 项目HEARTBEAT.md → 恢复全局上下文（替代读Chat History）
    ├──→ 读取 自己的任务HEARTBEAT → 恢复上次执行状态
    ├──→ 读取 上游任务HEARTBEAT → 获取依赖信息
    └──→ 读取 context_pool/*.md → 获取项目约束

子Agent执行中（记忆写入）
    │
    ├──→ 创建自己的任务HEARTBEAT（首次）
    ├──→ 更新执行进度（关键节点）
    └──→ 记录产出物（生成文件时）

子Agent完成/阻塞时（记忆同步）
    │
    ├──→ 更新任务HEARTBEAT状态
    └──→ 通知 orchestrator →
        └→ orchestrator: 更新项目HEARTBEAT看板

上下文快满时（强制压缩）
    │
    ├──→ 立即将当前状态 + 核心结论 压缩写入HEARTBEAT
    └──→ Chat History可以安全丢弃，HEARTBEAT已包含所有关键信息

任务出错时（错误恢复）
    │
    ├──→ 读取 HEARTBEAT → 定位偏差
    ├──→ 分析失败原因
    └──→ 更新 HEARTBEAT（记录错误 + 矫正措施）→ 重新执行
```

#### 2.2.4 上下文压缩与错误恢复

**压缩原则**：HEARTBEAT 中的内容必须是提炼过的结构化结论，不是对话记录的复制。

```
                  Chat History                     HEARTBEAT
            ┌───────────────────┐            ┌───────────────────┐
            │ 💬 冗长的对话细节   │  压缩写入  │ 📋 结构化摘要     │
            │ （会丢失，可丢弃）  │ ────────→ │ （持久化，不丢失）  │
            │                   │  丢弃细节   │                   │
            │ "我搜索了...发现..."│  保留结论   │ "选择A因为...理由" │
            │ "然后尝试了..."    │           │ "产出物: /path/xx" │
            │ "又调整了..."      │           │ "阻塞项: 需要确认" │
            └───────────────────┘            └───────────────────┘
```

**错误恢复流程**：

```
任务出错
    │
    ├── 1. 读取HEARTBEAT → 我做到哪了？产出了什么？
    ├── 2. 反思 → 对比目标vs现状，定位偏差原因
    ├── 3. 矫正 → 制定修正计划，不重复已完成的工作
    └── 4. 更新HEARTBEAT → 记录错误原因和矫正措施
```

#### 2.2.5 更新触发条件

| 事件类型        | 更新内容      | 触发者                 | 更新位置         |
| ----------- | --------- | ------------------- | ------------ |
| 任务状态变更      | 状态、进度、时间戳 | 子Agent              | 任务HEARTBEAT  |
| 任务状态变更      | 看板行更新     | orchestrator        | 项目HEARTBEAT  |
| 子Agent完成/阻塞 | 通知消息      | 子Agent              | → orchestrator |
| 新决策产生       | 决策记录      | orchestrator        | 项目HEARTBEAT  |
| 关键发现        | 发现记录      | 子Agent              | 任务HEARTBEAT  |
| 风险出现        | 风险项、应对措施  | orchestrator/子Agent | 项目HEARTBEAT  |
| 阻塞解除        | 阻塞项状态更新   | orchestrator        | 项目HEARTBEAT  |
| 里程碑达成       | 阶段完成标记    | orchestrator        | 项目HEARTBEAT  |
| 产出物生成       | 文件路径、说明   | 子Agent              | 任务HEARTBEAT  |

#### 2.2.6 状态流转

```
                    ┌──────────┐
         ┌─────────│  pending │◄────────┐
         │         └──────────┘         │
         │               │              │
    任务创建            派发             重试
         │               │              │
         ▼               ▼              │
   ┌──────────┐    ┌──────────┐         │
   │  failed  │◄───│ running  │─────────┘
   └──────────┘    └──────────┘
        ▲               │
        │失败           │完成
        │               ▼
        │          ┌──────────┐
        └──────────│completed │
                   └──────────┘
                        │
                   依赖未满足
                        ▼
                   ┌──────────┐
              ┌───│ blocked  │
              │   └──────────┘
              │        │
              └────────┘
                阻塞解除
```

### 2.3 Skills 管理系统

#### 2.3.1 Skills 分类

| 类别                 | 说明         | 示例                      |
| ------------------ | ---------- | ----------------------- |
| **Core Skills**    | 子Agent核心能力 | pm-coder, pm-researcher |
| **Domain Skills**  | 领域特定技能     | vue3, react, electron   |
| **Tool Skills**    | 工具集成       | github, notion, xlsx    |
| **Utility Skills** | 通用工具       | web-search, file-ops    |

#### 2.3.2 Skills 匹配算法

```python
def match_skills(task: Task) -> List[Skill]:
    """
    根据任务类型和描述匹配所需Skills
    """
    required_skills = []

    # 1. 基础Agent Skill
    required_skills.append(get_agent_skill(task.agent_type))

    # 2. 根据任务类型匹配
    if task.type == "frontend":
        if "vue" in task.description.lower():
            required_skills.append(get_skill("vue3"))
        elif "react" in task.description.lower():
            required_skills.append(get_skill("react"))

    # 3. 根据关键词匹配
    keywords = extract_keywords(task.description)
    for keyword in keywords:
        skill = find_skill_by_keyword(keyword)
        if skill:
            required_skills.append(skill)

    # 4. 去重并排序
    return deduplicate_and_sort(required_skills)
```

#### 2.3.3 Skills 存储结构

```
{skills_root}/                    # Skills 安装目录（路径变量，见 platform-adapter.md）
├── pm-orchestrator/              # 主控器 Skill（核心）
├── pm-backlog-manager/           # 需求池管理 Skill（核心）
├── pm-analyst/                   # 需求澄清 Skill（核心）
├── pm-planner/                   # 任务规划 Skill（核心）
├── pm-designer/                  # 原型设计 Skill（核心）
├── pm-runner/                    # 执行调度 Skill（核心）
├── pm-coder/                     # 编程执行 Skill（核心）
│   ├── SKILL.md                  # 行为规范（<300行，核心指令）
│   └── references/               # 渐进加载的参考资料
│       ├── heartbeat-ops.md
│       ├── code-standards.md
│       └── acceptance-criteria.md
├── pm-researcher/                # 信息检索 Skill（核心）
│   ├── SKILL.md
│   └── references/
│       └── report-templates.md
├── pm-writer/                    # 内容输出 Skill（核心）
│   ├── SKILL.md
│   └── references/
│       └── doc-templates.md
├── vue3/                         # Domain Skill（Phase 3从远程仓库安装）
├── electron/
└── ...

AI_PM_SKills/                     # 完整架构包
├── pm-core/                      # 🧠 共享内核（平台无关）
│   ├── context-protocol.md
│   ├── agent-lifecycle.md
│   ├── platform-adapter.md       # 平台抽象层
│   ├── references/               # 参考资料
│   └── templates/                # 模板
├── orchestrations/               # 🔄 编排模板（平台无关）
│   ├── code-only.yaml
│   ├── research-only.yaml
│   ├── analysis-only.yaml
│   ├── doc-only.yaml
│   ├── full-pipeline.yaml
│   └── custom/
├── harnesses/                    # ⚙️ Harness 定义（平台适配器）
│   └── ...
└── ...
```

### 2.4 Harness 生命周期管理

#### 2.4.1 子Agent Harness 生命周期

```
创建（团队创建 + 子Agent创建）
    │
    ├──→ 创建团队: 建立项目团队通信通道
    ├──→ 创建子Agent: 配置运行环境
    │    ├── 注入 Skill（通过 prompt 参数）
    │    ├── 绑定工具集（读写/消息通知等）
    │    └── 加入团队
    │
    ▼
运行（Running in Team）
    │
    ├──→ 读取 SKILL.md（行为规范）
    ├──→ 读取 HEARTBEAT（记忆恢复）
    ├──→ 执行任务
    ├──→ 更新任务 HEARTBEAT
    └──→ 通知 orchestrator
    │
    ▼
完成（通知 → 关闭）
    │
    ├──→ 通知 orchestrator 任务完成
    ├──→ orchestrator: 读取子Agent HEARTBEAT
    ├──→ orchestrator: 更新项目 HEARTBEAT
    │
    ▼
终止（关闭请求 + 团队清理）
    │
    ├──→ 请求子Agent关闭
    ├──→ 清理团队资源和通信通道
    └── 所有子Agent历史保存并移除
```

#### 2.4.2 通信协议

**子Agent → orchestrator**

```yaml
message_type: "task_complete" | "task_progress" | "task_blocked" | "task_failed"
task_id: "T001"
agent_name: "coder-T003"
payload:
  status: "completed"
  progress: 100
  deliverables:
    - path: "src/components/UserForm.vue"
  blockers: []
  suggestions: ["T005可以开始文档编写"]
```

**orchestrator → 子Agent**

```yaml
message_type: "task_update" | "decision" | "cancel"
payload:
  decision: "使用Vue3 + Electron"
  reason: "基于T001调研结论"
```

**orchestrator → 全体**

```yaml
message_type: "project_pause" | "project_cancel"
payload:
  reason: "用户要求暂停，等待确认方向"
```

## 3. 执行流程详解

### 3.1 Phase 0: 需求池管理 + MVP定义

#### 输入

- 用户自然语言描述的需求列表

#### 处理

1. **需求去重和分类**：识别重复需求、按功能领域分类
2. **优先级排序**：P0主流程核心 / P1重要可延后 / P2需暗地发觉
3. **MVP定义**：从P0需求中识别MVP范围，定义边界和验收标准
4. **批次交付计划**：基于优先级制定分批交付时间线

#### 输出

- `{context_root}/context_pool/backlog.md`
- `{context_root}/context_pool/mvp-definition.md`
- `{context_root}/context_pool/batch-plan.md`

### 3.2 Phase 1: 需求澄清 + 任务拆解

#### Phase 1a: 需求澄清

1. **意图识别**：提取产品类型、核心功能、目标用户
2. **范围界定**：明确In Scope / Out Scope
3. **约束提取**：技术栈偏好、时间限制、质量要求
4. **Goal构建**：结构化成功标准 + 硬/软约束

#### Phase 1b: 任务拆解

```python
def decompose_task(requirements: List[Requirement]) -> List[Task]:
    tasks = []

    for req in requirements:
        # 1. 识别任务类型
        task_type = classify_task(req)

        # 2. 确定执行Agent
        agent = select_agent(task_type)

        # 3. 分析依赖关系
        dependencies = analyze_dependencies(req, requirements)

        # 4. 创建任务
        task = Task(
            id=generate_task_id(),
            type=task_type,
            agent=agent,
            description=req.description,
            dependencies=dependencies,
            estimated_duration=estimate_duration(task_type, req.complexity)
        )
        tasks.append(task)

    # 5. 拓扑排序，确保依赖顺序
    return topological_sort(tasks)
```

#### 任务类型映射

| 需求关键词      | 任务类型          | 目标Agent         | 必需Skills        |
| ---------- | ------------- | --------------- | --------------- |
| 调研、对比、选型   | research      | pm-researcher   | web-search      |
| 设计、架构、规划   | design        | pm-orchestrator | -               |
| 前端、页面、UI   | frontend      | pm-coder        | vue3/react      |
| 后端、API、数据库 | backend       | pm-coder        | fastapi/express |
| 文档、PRD、说明  | documentation | pm-writer       | markdown        |
| 测试、验证、检查   | testing       | pm-coder        | jest/pytest     |

### 3.3 Phase 2: 原型设计（并行可选）

> 与 Phase 3 同时开始，原型完成后成为 Phase 3 的方向指标。

1. 读取 goal.md + modules.md
2. 设计交互流程（Mermaid 图）
3. 构建组件树（语义化命名）
4. 定义页面路由
5. 生成线框图（HTML+CSS 低保真可交互）
6. 自检：功能覆盖·完整流程·路由覆盖
7. 产出：prototype/ + 组件树文档

### 3.4 Phase 3: Skills管理 + 架构→开发→测试

#### Skills管理流程（本地优先 → 远程仓库补充 → 动态生成兜底）

```
1. 收集所有任务所需Skills
2. 检查本地Skills
   ├── 检查 {skills_root}/ 下已安装的 Skill
   └── 检查项目级 Skills
3. 生成缺失Skills清单
4. 从远程仓库搜索缺失Skills
   ├── 搜索关键词
   └── 评估结果（名称匹配/描述/热度/安全/稳定性）
5. 安装到 {skills_root}/（默认用户级）
6. 验证安装（SKILL.md存在 + 可见）
7. 远程仓库无匹配 → 动态生成临时Skill
```

#### 架构讨论 → 开发 → 测试

1. **架构讨论**：创建多个 coder 各提方案，上报 orchestrator 评审
2. **Skills 安装与验证**：按 skills-needed.md 安装
3. **DAG 批次调度开发**：按依赖图并行调度，初期以需求文档为基准开发，原型完成后切换为对齐原型开发
4. **模块测试（自动化验收）**：编写测试，自动化验收标准执行
5. **文档编写**：输出文档

### 3.5 Phase 4: 打回循环

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

### 3.6 Phase 5: 结果收集 + 整合交付

#### 收集机制

1. **主动推送**：子Agent完成任务后主动通知 orchestrator
2. **被动拉取**：orchestrator 读取子Agent HEARTBEAT
3. **阻塞通知**：子Agent遇到阻塞立即通知 orchestrator

#### 验证规则

- 输出格式是否符合约定
- 文件路径是否正确
- 依赖的共享数据是否更新
- 质量检查是否通过

#### 整合步骤

1. **收集所有交付物**
2. **一致性检查**：API schema与代码实现、数据库设计与API、文档与代码
3. **冲突解决**：命名冲突、版本冲突、逻辑冲突
4. **生成最终交付物**：源代码 + 文档 + 测试 + 部署配置

## 4. 异常处理

### 4.1 子Agent失败

| 失败类型   | 检测方式  | 处理策略          |
| ------ | ----- | ------------- |
| 执行超时   | 心跳超时  | 重试1次，仍失败则标记失败 |
| 输出格式错误 | 验证失败  | 返回子Agent要求修正  |
| 依赖缺失   | 运行时错误 | 补充依赖后重试       |
| 逻辑错误   | 测试失败  | 返回子Agent修复    |
| 资源不足   | 系统错误  | 扩容或降级处理       |

### 4.2 上下文冲突

| 冲突类型 | 检测时机   | 解决策略       |
| ---- | ------ | ---------- |
| 写冲突  | 文件锁检测  | 乐观锁 + 合并策略 |
| 版本冲突 | 提交时检测  | 人工介入或自动合并  |
| 逻辑冲突 | 一致性检查时 | 回滚 + 重新执行  |

### 4.3 网络/服务故障

| 故障类型        | 影响          | 应对           |
| ----------- | ----------- | ------------ |
| Skills仓库不可达 | 无法下载新Skills | 使用本地缓存，记录待下载 |
| 子Agent启动失败  | 任务无法执行      | 重试3次，失败后人工介入 |
| 上下文池写入失败    | 状态丢失风险      | 本地备份 + 定期同步  |

### 4.4 结构化恢复机制

当子 Agent 遇到已知失败模式时，按以下 SOP 自动恢复：

**恢复总则**：

1. 分类失败类型（参考 `pm-core/references/recovery-recipes.md`）
2. 查找对应的恢复配方
3. 执行恢复动作（最多重试 1 次）
4. 恢复失败 → 通知 orchestrator 升级处理
5. 更新 HEARTBEAT 记录恢复过程

**失败分类与恢复配方**：

| 分类码                  | 失败类型      | 恢复动作                         | 可自动恢复 |
| -------------------- | --------- | ---------------------------- | ----- |
| `context_overflow`   | 上下文窗口溢出   | 压缩写入 HEARTBEAT → 请求重启        | ✅     |
| `dependency_missing` | 上游依赖未满足   | 标记 blocked → 通知 orchestrator | ❌     |
| `file_conflict`      | 文件冲突      | 读取最新版本 → 智能合并                | ✅     |
| `tool_failure`       | 工具调用失败    | 分析错误 → 等待后重试                 | ✅     |
| `network_timeout`    | API/网络超时  | 等待后重试                        | ✅     |
| `skill_missing`      | Skill 未找到 | 通知 orchestrator 补装           | ❌     |
| `build_failure`      | 编译/测试失败   | 分析错误 → 修复代码                  | ✅     |
| `permission_denied`  | 权限不足      | 通知 orchestrator 人工介入         | ❌     |

**恢复台账**：每次恢复尝试在 HEARTBEAT "遇到的问题" 区域记录：

```markdown
| # | 时间 | 问题描述 | 恢复配方 | 尝试次数 | 结果 | 状态 |
|---|------|---------|---------|---------|------|------|
```

### 4.5 部分成功支持

系统支持部分成功/部分失败状态，带有结构化降级报告：

```yaml
# 部分成功消息格式
type: task_partial_success
fields:
  - task_id: "T003"
  - agent: "pm-coder"
  - completed_items: ["用户登录API", "注册API"]
  - incomplete_items: ["密码重置API（缺少邮箱服务配置）"]
  - degradation_report: "核心功能已完成，附加功能需等待配置"
  - suggestion: "建议接受部分交付，密码重置API作为后续任务"
```

## 4.6 事件驱动通信协议

### 设计原则

> **事件优于文本抓取**：Agent 间通信使用结构化事件，而非依赖自然语言解析。

所有消息体采用结构化格式：

```yaml
# 标准消息信封
message_envelope:
  type: "message" | "broadcast"
  recipient: "main" | "{agent-name}"
  summary: "一句话摘要"
  content:
    event_type: "task_complete" | "task_progress" | "task_blocked" | "task_failed" | "task_partial_success" | "recovery_attempt" | "decision_request"
    task_id: "T{XXX}"
    agent_name: "{agent-role}"
    timestamp: "{YYYY-MM-DD HH:mm}"
    payload: {类型特定数据}
```

### 事件类型定义

| 事件类型                   | 必填字段                                                   | 触发者     |
| ---------------------- | ------------------------------------------------------ | ------- |
| `task_complete`        | task_id, status, deliverables, suggestions             | 子 Agent |
| `task_progress`        | task_id, progress_pct, current_step                    | 子 Agent |
| `task_blocked`         | task_id, block_reason, needed_from                     | 子 Agent |
| `task_failed`          | task_id, error_kind, error_detail, recoverable         | 子 Agent |
| `task_partial_success` | task_id, completed_items, incomplete_items, suggestion | 子 Agent |
| `recovery_attempt`     | task_id, recipe_id, attempt_count, result              | 子 Agent |
| `decision_request`     | task_id, decision_description, options                 | 子 Agent |

## 4.7 策略引擎

orchestrator 内置自动化策略规则，减少 ad-hoc 判断：

```yaml
policies:
  auto_unblock:
    trigger: 上游任务 COMPLETED
    action: 检查所有 BLOCKED 任务，依赖全部满足则自动派发

  auto_recover:
    trigger: 子Agent FAILED
    action: 按恢复配方尝试恢复
    condition: 失败类型可恢复 且 尝试次数 < 2

  timeout_escalate:
    trigger: 子Agent RUNNING 超过预期时间 50%
    action: 主动检查 HEARTBEAT，无进展则通知用户

  auto_cleanup:
    trigger: 所有任务 COMPLETED 或 FAILED
    action: 关闭子Agent + 清理团队资源

  budget_warning:
    trigger: 项目总轮次 > 上限的 80%
    action: 评估剩余任务优先级，通知用户预算即将耗尽

  constraint_escalate:
    trigger: 子Agent 输出违反硬约束
    action: 立即通知用户，不自动恢复
```

### 4.8 目标驱动模型

> **核心理念**：Goal 是结构化一等公民，贯穿项目全生命周期。

#### 核心概念

| 要素                   | 说明                   |
| -------------------- | -------------------- |
| **success_criteria** | 加权成功标准，多维度衡量项目质量     |
| **constraints**      | 硬约束（违规立即升级）+ 软约束（偏好） |
| **context**          | 注入每次 LLM 调用的领域知识     |

#### Goal 生命周期

- **Phase 1**：需求澄清后立即构建 Goal → 写入 `{context_root}/context_pool/goal.md`
- **Phase 4**：Goal 的 success_criteria + constraints 注入每个子 Agent prompt
- **Phase 6**：用 success_criteria 做加权验收
- **全程**：硬约束违规 → 立即升级人工

> 详见：`harnesses/orchestrator-harness.md` — 目标驱动模型

### 4.9 三角验证模型

> **核心理念**：没有单一可靠的 Ground Truth，多信号收敛 = 可靠性。

#### 三层验证信号

| 信号层       | 特点       | 执行者              | 检查内容                            |
| --------- | -------- | ---------------- | ------------------------------- |
| **确定性规则** | 零歧义、快速   | 子Agent（自验证）      | 格式检查、编译/测试通过、关键词匹配              |
| **语义评估**  | 灵活但有幻觉风险 | orchestrator（验收） | 对比 Goal 的 success_criteria 加权打分 |
| **人工判断**  | 最高权威     | 用户（HITL）         | 关键决策确认、交付审批                     |

> 详见：`harnesses/orchestrator-harness.md` — 三角验证模型

### 4.10 三层 Prompt 洋葱模型

> **核心理念**：子 Agent 的上下文 = 三层叠加，确保始终知道"我是谁、做了什么、现在做什么"。

```
┌─────────────────────────────────────────────┐
│ Layer 1 — Identity（身份层）                  │
│ 来源: SKILL.md + Goal                        │
│ 内容: 角色定位、职责、能力边界、成功标准、约束   │
│ 特点: 静态，永不改变                          │
├─────────────────────────────────────────────┤
│ Layer 2 — Narrative（叙事层）                 │
│ 来源: HEARTBEAT + context_pool（动态构建）     │
│ 内容: 项目状态、已完成阶段、上游结论、当前进度   │
│ 特点: 每次子Agent启动时从文件自动拼接          │
├─────────────────────────────────────────────┤
│ Layer 3 — Focus（焦点层）                     │
│ 来源: orchestrator 的任务描述                  │
│ 内容: 具体任务ID、描述、验收标准、输出格式       │
│ 特点: 每次切换任务时替换                       │
└─────────────────────────────────────────────┘
```

> 详见：`harnesses/orchestrator-harness.md` — 三层 Prompt 洋葱模型

### 4.11 Skill 渐进式披露

> **核心理念**：控制 token 消耗，按需加载，避免一次性注入过多信息。

| 层级                        | 加载内容               | 时机                    | Token 成本         |
| ------------------------- | ------------------ | --------------------- | ---------------- |
| **Tier 1 — Catalog**      | name + description | 创建子Agent时注入 prompt      | ~50-100 tokens/个 |
| **Tier 2 — Instructions** | 完整 SKILL.md        | Agent 自主读取激活 | <5000 tokens     |
| **Tier 3 — Resources**    | 脚本、参考文档            | 指令中引用时按需读取    | 因情况而异            |

> 详见：`harnesses/orchestrator-harness.md` — Skill 渐进式披露

### 4.12 默认技能体系

所有子Agent共享的 6 个运行时行为技能（详见 `pm-core/references/default-skills.md`）：

| 默认技能                    | 目的         | 注入点         |
| ----------------------- | ---------- | ----------- |
| pm-note-taking          | 防止丢失跟踪     | 系统提示        |
| pm-progress-tracker     | 防止任务跳过/丢弃  | 迭代边界 + 完成钩子 |
| pm-context-preservation | 上下文快满时主动保存 | 系统提示        |
| pm-quality-monitor      | 定期自我评估质量   | 迭代边界        |
| pm-error-recovery       | 结构化恢复协议    | 系统提示        |
| pm-consistency-guard    | 产出物与约束一致   | 系统提示 + 完成钩子 |

### 4.13 边条件与条件路由

子Agent间的数据流支持 4 种边条件：

| 边类型         | 触发条件   | 使用场景       |
| ----------- | ------ | ---------- |
| On Success  | 源节点成功  | 正常流程推进     |
| On Failure  | 源节点失败  | 回退、错误恢复    |
| Conditional | 表达式为真  | 调研结果驱动不同分支 |
| Human Gate  | 需要人工确认 | 架构决策、交付审批  |

> 详见：`harnesses/orchestrator-harness.md` — 边条件与条件路由

### 4.14 预算控制

```yaml
budget:
  max_turns: 100                    # 项目总轮次上限
  per_task_max_turns:               # 各子Agent单任务上限
    coder: 50
    researcher: 40
    writer: 35
  over_budget_action:               # 超预算处理策略
    1. 评估剩余任务优先级
    2. 标记低优先级任务为 DEFERRED
    3. 通知用户预算状态
```

> 详见：`harnesses/orchestrator-harness.md` — 预算控制

### 4.15 子Agent委托机制

| 委托规则   | 说明                                  |
| ------ | ----------------------------------- |
| **允许** | pm-coder → pm-researcher（编码时需调研）    |
| **允许** | pm-writer → pm-researcher（撰写时需补充信息） |
| **禁止** | pm-researcher 不得委托（调研是终端操作）         |
| **禁止** | 嵌套委托（A→B→C 不允许）                     |

> 详见：`harnesses/orchestrator-harness.md` — 子Agent委托机制

### 4.16 演化循环

> **核心理念**：利用 HEARTBEAT 积累的项目数据，持续改进 Skill 和 Harness 规范。

#### 四阶段演化循环

| 阶段           | 说明                                     |
| ------------ | -------------------------------------- |
| **Execute**  | Worker Agent 按当前规范运行，产生 HEARTBEAT 日志   |
| **Evaluate** | 对比 Goal 的 success_criteria，判断产出质量      |
| **Diagnose** | 从 HEARTBEAT 提取结构化失败数据（失败类型、恢复频率、信号不一致） |
| **Improve**  | 更新 SKILL.md / Harness / 恢复配方 / 策略引擎    |

#### 演化限制

> 演化使系统在**已遇到过的那类问题**上更可靠，但不能让 Agent 对全新问题变聪明。
> 对于真正新颖的情况，需要人工介入——而每次人工介入都会成为下一个演化周期的燃料。

> 详见：`harnesses/orchestrator-harness.md` — 经验积累与持续学习

### 4.17 健康检查与监控

> **核心理念**：子 Agent 运行不是"发射后不管"。orchestrator 主动探查子 Agent 的状态和中间产物质量。

#### 四层监控体系

| 层级            | 监控对象        | 检查方式                       | 介入手段                       |
| ------------- | ----------- | -------------------------- | -------------------------- |
| **L1 存活检查**   | 子Agent是否在运行 | HEARTBEAT 最后更新时间           | 超时 → 策略引擎 timeout_escalate |
| **L2 进度检查**   | 子Agent是否在推进 | 实际进度 vs 预期进度               | 进度停滞 → 主动询问或调整             |
| **L3 产出质量检查** | 中间产物是否达标    | 确定性规则 + 语义评估               | 质量偏差 → 中断并调整方向             |
| **L4 约束合规检查** | 产出是否违反护栏    | 逐项核对 hard/soft constraints | 硬约束违规 → 立即中断               |

#### 健康度评分

orchestrator 为每个子Agent 维护 0-100 的健康度评分，因子包括：

- `progress_velocity`（进度速度，权重 0.3）
- `heartbeat_freshness`（心跳新鲜度，权重 0.25）
- `error_count`（错误计数，权重 0.2）
- `constraint_compliance`（约束合规，权重 0.25）

健康度等级：≥75 健康 | 50-74 关注 | 30-49 预警 | <30 严重

#### 里程碑检查点

复杂任务预设里程碑，每个里程碑定义 expected_output + quality_gate。orchestrator 在每个检查点自动触发质量检查，不通过则阻止继续。

详见：`pm-core/references/health-check-protocols.md`

### 4.18 检查点与崩溃恢复

> **核心理念**：AI平台已有会话恢复机制，但作为通用框架需要第二道防线——结构化检查点。

#### 三层崩溃恢复防线

```
第一层：HEARTBEAT 压缩恢复（已有）
  └─ 子Agent单体会话中断

第二层：检查点快照恢复（本模块）
  └─ orchestrator中断、多Agent异常、需要精确回滚

第三层：人工介入恢复（已有 ESCALATE）
  └─ 不可恢复的系统性故障
```

#### 检查点类型

| 类型       | 触发时机         | 恢复用途   |
| -------- | ------------ | ------ |
| **项目快照** | Phase 完成时    | 整个项目回滚 |
| **任务快照** | 子Agent里程碑完成时 | 单任务回滚  |
| **决策快照** | 关键决策后        | 决策回溯   |

存储位置：`{context_root}/checkpoints/`，按版本号管理，超出保留上限自动清理。

#### 崩溃场景应对

| 场景             | 恢复策略                       |
| -------------- | -------------------------- |
| 子Agent失联       | 任务快照恢复 → 重新启动          |
| orchestrator中断 | 项目快照恢复 → 重构上下文             |
| 上下文溢出          | HEARTBEAT 已包含压缩状态 → 重启     |
| 多Agent级联失败     | 最近项目快照全面回滚                 |
| 团队通道丢失         | 重建团队 → 从快照恢复所有任务 |

详见：`pm-core/references/checkpoint-recovery.md`

### 4.19 经验积累与持续学习

> **核心理念**：系统从每次执行中学习，重复模式沉淀为可复用知识资产。

#### 经验四级分类

| 级别     | 名称      | 沉淀位置                        | 触发条件            | 作用范围              |
| ------ | ------- | --------------------------- | --------------- | ----------------- |
| **L1** | 微模式     | `{context_root}/lessons-learned.md` | 同类问题 ≥2 次       | 当前项目              |
| **L2** | 规则强化    | Harness 策略引擎                | 微模式被 ≥3 个项目验证   | 所有项目              |
| **L3** | Skill增强 | SKILL.md / references/      | 需 Agent 遵守的行为规范 | 绑定该 Skill 的 Agent |
| **L4** | 独立Skill | 新建 `pm-{name}/`             | 跨领域通用           | 全局可用              |

#### 经验生命周期

```
孵化期（3项目/30天）→ 验证期（再出现则生效）→ 固化期（5+项目无负面影响）→ 淘汰期（10项目未触发）
```

#### Phase 7.5 复盘流程

orchestrator 在 Phase 5 整合交付后执行复盘：

1. **数据收集**：扫描所有任务HEARTBEAT的问题区 + 恢复台账
2. **模式识别**：聚类相似问题，识别三角验证不一致频率
3. **经验分级**：新模式 → L1，已验证 → L2/L3，通用 → L4
4. **更新资产**：按级别更新 lessons-learned / 策略引擎 / SKILL.md
5. **生成报告**：结构化复盘报告（成功标准达成率 + 恢复配方统计 + 新增经验 + 改进建议）

### 4.20 Coder Harness v2 — 六大核心开发策略

> **设计参考**：Claude Code 的六大工程策略，适配 AI_PM_Skills 的 Multi-Agent 架构。
> 将 Coder 从"接到任务就写代码"升级为"先规划后编码、权限分级管控、钩子自动验证"。

| 策略          | 核心理念         | 关键机制                             |
| ----------- | ------------ | -------------------------------- |
| **规划优先管制**  | 拒绝"上来就干"     | Phase A探索→plan.md→审批→Phase C编码   |
| **上下文工程**   | 上下文是有限内存     | L1常驻≤3000 + L2工作集≤15000 + L3冷引用  |
| **意图-执行解耦** | Layer 0 意图解析 | 歧义消除→范围锚定→风险预判→plan.md           |
| **风险分级权限**  | 操作危险度动态设卡    | 绿灯自主/黄灯通知/红灯审批/禁区禁止              |
| **交接棒机制**   | 长任务无缝续接      | HANDOFF.md + orchestrator重新启动 |
| **事件驱动钩子**  | 确定性规则约束AI行为  | 9个H1-H9钩子覆盖全生命周期                 |

> 详见：`harnesses/coder-harness.md`

### 4.21 Guides/Sensors 双闭环控制模型

> **设计参考**：毒舌产品经理 4.0 的"双闭环控制机制"。
> Agent = 模型 + Harness。Guides（前馈）在行动前注入标准，Sensors（反馈）在行动后检查结果。
> **⚠️ 设计变更**：推理型传感器由 orchestrator 在验收时执行，不在 Coder 自检范围内。

```
┌──────────────────────────────────────────────────────────────┐
│                   Harness = 模型 + 控制                       │
│                                                              │
│  ┌─────────────────────┐    ┌──────────────────────────┐    │
│  │  Guides（前馈控制）   │    │  Sensors（反馈控制）       │    │
│  │                     │    │                          │    │
│  │  • SKILL.md         │    │  计算型传感器（Coder自执行）│    │
│  │  • Goal 标准        │──→──│    • H1-H9 Hooks        │    │
│  │  • code-standards   │    │                          │    │
│  │  • review-protocol  │    │  轻量自检（Coder自执行）   │    │
│  │  • debug-protocol   │    │    • 关键词匹配          │    │
│  │                     │    │                          │    │
│  │                     │    │  推理型传感器（Orchestrator│    │
│  │  在行动之前注入标准    │    │    验收时执行）           │    │
│  │                     │    │    • 语义评估             │    │
│  │                     │    │    • Spec 深度对照         │    │
│  └─────────────────────┘    └──────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

| 闭环             | 作用          | 代表组件                                         | 执行者          |
| -------------- | ----------- | -------------------------------------------- | ------------ |
| **Guides 前馈**  | 提高"一次做对"的概率 | SKILL.md、Goal、code-standards、review-protocol | —            |
| **Sensors 反馈** | 发现偏差并自我修正   | 计算型H1-H9钩子、关键词匹配                             | Coder        |
| **Sensors 反馈** | 语义级验收审查     | 语义评估、Spec深度对照                                | Orchestrator |

> 详见：`harnesses/coder-harness.md` — Guides/Sensors 双闭环控制模型

### 4.22 两阶段 Code Review 自动闭环

> **⚠️ 设计变更**：推理型语义审查由 orchestrator 验收时执行（三角验证第二层），Coder 自检只做计算型检查。

```
编码完成
    │
    ▼
Stage 1: 功能完整性自检（轻量关键词匹配）
    ├── BLOCKING 问题 → 修复（最多2轮）→ 不通过 → task_blocked
    └── 通过 → Stage 2 代码质量检查（计算型 H7/H8/H9）
         ├── H7 测试通过
         ├── H8 Lint 通过
         ├── H9 完整性检查通过
         └── 全部通过 → 通知 orchestrator 完成
                              │
                              ▼
              ┌───────────────────────────────────┐
              │ Orchestrator 验收（异步执行）        │
              │                                    │
              │ Stage 3: 语义评估（推理型传感器）   │
              │   ├── 对比 Goal success_criteria   │
              │   ├── Spec 深度对照                │
              │   └── 代码质量语义审查              │
              │                                    │
              │ 置信度 ≥ 80% → COMPLETED          │
              │ 置信度 < 80% → 发回 coder 修复     │
              └───────────────────────────────────┘
```

| 约束                         | 说明              |
| -------------------------- | --------------- |
| Stage 1 不通过则阻断 Stage 2     | 功能完整性是前提        |
| 每个 Task 完成后立即审查            | 不攒到 Phase 结束    |
| Stage 1/2 由 Coder 自检       | 计算型检查 + 轻量关键词匹配 |
| Stage 3 由 Orchestrator 验收  | 语义级评估，三角验证第二层   |
| 优先级：设计图 > 设计规范 > 需求文档 > 代码 | 文档优先于实现         |

> 详见：`pm-coder/references/code-review-protocol.md`

### 4.23 四阶段系统性调试 SOP

| 阶段                | 核心活动                   | 关键原则   |
| ----------------- | ---------------------- | ------ |
| **Phase 1: 收集证据** | 读错误信息→判断稳定性→检查变更→追踪数据流 | 不只看摘要  |
| **Phase 2: 分析模式** | 找相似正常功能→识别差异→排除法缩小范围   | 对比找线索  |
| **Phase 3: 假设验证** | 形成最多3个假设→最小改动验证        | 不猜不试   |
| **Phase 4: 实施修复** | 一次只改一处→编译→功能→回归→清理     | 一次只改一处 |

**核心原则：** 不猜不试、一次只改一处、必须回归验证。
**止损机制：** 同一 bug 经 2 轮仍未修复 → 立即停止，升级 orchestrator 介入。

> 详见：`pm-coder/references/debugging-protocol.md`

---

## 5. 扩展设计

### 5.1 添加新子Agent

1. 创建 `pm-{name}/SKILL.md` — 行为规范（平台无关）
2. 创建 `pm-{name}/standalone-prompt.md` — 独立运行精简prompt
3. 创建 `pm-{name}/references/` — 渐进加载的参考资料
4. 创建 `harnesses/{name}-harness.md` — 执行载体定义（平台适配器）
5. 在 `pm-orchestrator` 中注册：
   - 任务类型映射
   - 必需Skills
   - 创建子Agent的 prompt 模板
6. 更新 `orchestrations/` 中相关编排模板

### 5.2 自定义Skills

```yaml
# custom-skill.yaml
name: my-custom-skill
description: |
  自定义Skill描述
triggers:
  - keyword1
  - keyword2
actions:
  - type: script
    file: scripts/action.py
  - type: reference
    file: references/guide.md
```

### 5.3 插件机制

预留扩展点：

- 自定义任务拆解策略
- 自定义Skills源
- 自定义结果整合器
- 自定义通知渠道

---

## 6. v3.1 Ironforge 企业工程化框架

> 详见 `ENTERPRISE_FRAMEWORK.md` 获取完整设计文档

### 6.1 三大支柱总览

| 支柱 | 代号 | 核心变化 | 新增文件 |
|------|------|---------|---------|
| 🔒 安全加固 | Fortify | "告诉Agent不要做" → "让Agent做不到" | security/ 3个文件 |
| 🛡️ 稳定性加固 | Stabilize | 文档级恢复 → 确定性检查点恢复 | stability/ 3个文件 |
| 📊 可观测性加固 | Observe | Markdown给人看 → 机器+人双格式 | observability/ 3个文件 |

### 6.2 安全加固架构

三层权限执行模型：

```
Layer 1: 平台物理层（不可绕过）
  └─ spawn mode=plan → 写/改/执行 物理不可用
  └─ blocked_tools → 危险工具直接移除

Layer 2: Harness 逻辑层（语义约束）
  └─ 黄灯操作 → 执行前通知 → orchestrator 可拦截

Layer 3: SKILL.md 行为层（自律约束）
  └─ 编码规范、禁止事项 → "道德准则"兜底
```

敏感信息防护：写入前正则扫描 + 受保护路径 + 审计日志（JSONL追加+trace_id）。

### 6.3 稳定性加固架构

检查点与回滚：

```
自动检查点 → 每个 plan step 完成后
验证点检查点 → 推理验证点通过后（回滚优先使用）
交接检查点 → Agent间交接时

回滚5步：定位安全检查点 → 文件还原（git优先）→ 状态还原 → 重新执行 → 审计记录
```

三级熔断模型：

```
Agent级：closed → open（停止派发）→ half_open（试探1次）
任务级：重试预算 + 指数退避
项目级：轮次>80%且完成率<30% → 全局暂停
```

幂等性规范：replace_in_file 0匹配=跳过 + 命令pre_check + 文件操作安全策略。

### 6.4 可观测性加固架构

HEARTBEAT v2 双格式：

```yaml
--- 
# YAML front matter（机器可解析）
heartbeat_version: "2.0"
trace_id: "proj-20260424-a3f2"
status: "running"
progress: 60
health_score: 82
---

# Markdown body（人类可读）
## 已完成步骤
- [x] Step 1: 创建组件文件
```

指标三类：Counter（只增）+ Gauge（可增减）+ Histogram（分布）。

分布式追踪：trace_id 贯穿审计/指标/检查点/HEARTBEAT，span_id 定位Agent+任务。

### 6.5 Harness 分层架构（v3.1 核心改造）

```
harnesses/
├── base/                  ← 公共层（7个文件，所有Agent共享）
│   ├── permission-framework.md
│   ├── security-hooks.md
│   ├── audit-logging.md
│   ├── checkpoint-protocol.md
│   ├── handoff-protocol.md
│   ├── context-engineering.md
│   └── observability-config.md
│
├── coder-harness.md       ← 继承base + 特化逻辑
├── runner-harness.md      ← 继承base + 特化逻辑
└── ... 其余7个同理
```

**每个 Harness 只保留特化逻辑**：权限覆盖 + 审计事件覆盖 + P1稳定性覆盖 + 上下文预算覆盖。

**维护成本**：改一处生效全部（如安全策略更新 → 只改 base/security-hooks.md）。
