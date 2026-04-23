# 🚀 AI PM Skills 快速入门指南

> 5分钟上手，让AI产品经理团队为你干活。
> v3.1 Ironforge — 企业工程化（安全/稳定/可观测）+ Skill独立运行 + 按需编排 + 上下文自适应

---

## 一、最小可行流程（5分钟上手）

你只需要说一句话，剩下的交给系统。

```
你："帮我做一个待办事项Web应用"
```

系统自动完成：

| 步骤 | 发生了什么 | 你需要做什么 |
|:----:|-----------|------------|
| ① | **orchestrator 自动触发** — 识别到"做一个"，加载 pm-orchestrator Skill | 无 |
| ② | **需求澄清** — orchestrator 向你确认产品类型、核心功能、技术偏好 | 回答几个问题 |
| ③ | **自动拆解+执行** — 拆成调研→设计→编码→文档等子任务，多个Agent并行干活 | 等待 |
| ④ | **关键节点确认** — 架构方案、最终交付 | 看一眼，点头或提意见 |
| ⑤ | **交付** — 代码+文档+测试，一步到位 | 验收 |

> **核心理念**：你是产品经理，AI是你的团队。你说需求，团队执行，你只在关键决策点介入。

---

## 二、三种使用方式

### 方式1：单Skill直接使用（最轻量）

```
你："帮我改个Bug"       → 加载 pm-coder → 独立运行 → 交付
你："调研下XX技术"      → 加载 pm-researcher → 独立运行 → 交付
你："帮我排一下需求优先级" → 加载 pm-backlog-manager → 独立运行 → 交付
```

每个 Skill 启动时自动扫描上下文，三级自适应：
- **FULL** — 所有前置文档齐全，严格对齐流程
- **PARTIAL** — 部分文档存在，对齐已有、缺失从用户推导
- **MINIMAL** — 无前置文档，直接接收用户指令，轻量运行

### 方式2：预定义编排（中等复杂度）

```
你："梳理需求+拆解任务" → analysis-only编排 → pm-analyst → pm-planner
你："写个PRD"          → doc-only编排 → pm-writer
```

### 方式3：全链路编排（完整项目）

```
你："从0做个App" → full-pipeline编排 → Phase 0→1→2→3→4→5
```

---

## 三、第一次使用前的准备

### 环境检查清单

| # | 检查项 | 验证方法 | 预期结果 |
|---|--------|---------|---------|
| 1 | AI编程助手已安装 | 打开客户端 | 能正常对话 |
| 2 | 9个核心 Skill 目录存在 | 查看项目目录 | 每个 `pm-*/` 下有 SKILL.md |
| 3 | pm-core 目录完整 | 查看项目目录 | `pm-core/references/` 和 `pm-core/templates/` 存在 |
| 4 | Harness 定义存在 | 查看 `harnesses/` 目录 | 9个 .md 文件（含 README） |
| 5 | 编排模板存在 | 查看 `orchestrations/` 目录 | 5个预定义编排 + custom/ 目录 |

### 目录结构速览

```
你的项目/
├── pm-orchestrator/       # 🎯 主控器（项目经理）
│   └── SKILL.md
├── pm-backlog-manager/    # 📋 需求池管理（Phase 0 守门人）
│   └── SKILL.md
├── pm-analyst/            # 🔍 需求澄清专家
│   └── SKILL.md
├── pm-planner/            # 📊 任务规划专家
│   └── SKILL.md
├── pm-designer/           # 🎨 原型设计专家（并行可选）
│   └── SKILL.md
├── pm-runner/             # ⚙️ 执行调度专家
│   └── SKILL.md
├── pm-coder/              # 💻 编码专家（程序员）
│   ├── SKILL.md
│   └── references/        # 详细规范（按需加载）
├── pm-researcher/         # 🔎 调研专家（分析师）
│   ├── SKILL.md
│   └── references/
├── pm-writer/             # 📝 文档专家（技术写作）
│   ├── SKILL.md
│   └── references/
├── pm-core/               # 🧠 共享内核
│   ├── context-protocol.md
│   ├── agent-lifecycle.md
│   ├── platform-adapter.md  # 平台抽象层
│   ├── security/            # 🔒 安全协议（v3.1）
│   │   ├── permission-framework.md
│   │   ├── secrets-protection.md
│   │   └── audit-protocol.md
│   ├── stability/           # 🛡️ 稳定性协议（v3.1）
│   │   ├── checkpoint-protocol.md
│   │   ├── circuit-breaker.md
│   │   └── idempotency.md
│   ├── observability/       # 📊 可观测性协议（v3.1）
│   │   ├── heartbeat-v2.md
│   │   ├── metrics-protocol.md
│   │   └── trace-protocol.md
│   ├── references/          # 恢复配方、默认技能、健康检查协议等
│   └── templates/           # HEARTBEAT v2模板
├── orchestrations/        # 🔄 编排模板
│   ├── code-only.yaml
│   ├── research-only.yaml
│   ├── analysis-only.yaml
│   ├── doc-only.yaml
│   ├── full-pipeline.yaml
│   └── custom/             # 自定义编排
├── harnesses/             # ⚙️ 执行载体配置
│   ├── base/               # 🆕 公共层（v3.1 Ironforge）
│   │   ├── permission-framework.md
│   │   ├── security-hooks.md
│   │   ├── audit-logging.md
│   │   ├── checkpoint-protocol.md
│   │   ├── handoff-protocol.md
│   │   ├── context-engineering.md
│   │   └── observability-config.md
│   └── ... 9个Agent Harness
└── ARCHITECTURE.md        # 架构详细文档
```

---

## 四、关键概念速查

### Skill vs Harness

| | Skill | Harness |
|-|-------|---------|
| **一句话** | SOP手册——"做什么、怎么做" | 工作台——"用什么工具、在哪做" |
| **类比** | 厨师的菜谱 | 厨房的灶台+锅碗瓢盆 |
| **文件** | `pm-coder/SKILL.md` | `harnesses/coder-harness.md` |
| **变不变** | 项目级，基本不变 | 任务级，每次创建子Agent重新配置 |

### 上下文自适应

每个 Skill 启动时自动扫描上下文，决定运行模式：

| 等级 | 条件 | 行为 |
|------|------|------|
| **FULL** | 所有前置文档齐全 | 严格对齐，按流程验收 |
| **PARTIAL** | 部分文档存在 | 对齐已有，缺失从用户推导 |
| **MINIMAL** | 无前置文档 | 直接接收用户指令，轻量运行 |

> 平台抽象层详见 `pm-core/platform-adapter.md`（定义 `{context_root}`、`{skills_root}` 等路径变量和操作映射）。

### HEARTBEAT v2 是什么（v3.1 升级）

**类比：周报 + 自动化监控仪表盘。**

- 项目经理不看每一行 Git 提交（Chat History），只看每周周报（HEARTBEAT）
- 两级结构：
  - **项目级** `{context_root}/HEARTBEAT.md` — 全局看板，orchestrator 维护
  - **任务级** `T001-heartbeat.md` — 各子Agent维护自己的任务进度
- 关键特性：**即使对话丢失，HEARTBEAT 也在**，Agent 重启后通过它恢复状态
- **v3.1 新增**：
  - **YAML front matter**（机器可解析）：status/progress/health_score/trace_id/turn_usage
  - **Markdown body**（人类可读）：任务步骤/产出物/阻塞项
  - **5条自动化监控规则**：任务停滞/健康度下降/轮次超支/错误聚集/阻塞超时
  - **trace_id 贯穿**：从项目到审计日志到指标，统一追踪

### 三角验证是什么

**类比：老板-质检-客户三道验收。**

| 验证层 | 类比 | 谁来做 | 做什么 |
|--------|------|--------|--------|
| 确定性规则 | 质检员量尺寸 | 子Agent自验证 | 格式检查、编译通过、测试通过 |
| 语义评估 | 老板看成品 | orchestrator | 对比成功标准打分，≥80%才算过 |
| 人工判断 | 客户签字 | 你 | 关键决策确认、最终交付审批 |

> 三层信号都通过，才算真正完成。任何一层不通过都会被打回。

---

## 五、orchestrator 内部流程图（全链路模式）

```
用户说需求
    │
    ▼
┌──────────────────────────────────┐
│ Phase 0: 需求池管理              │ 👤 用户介入：确认MVP范围
│ · 需求去重、分类、优先级排序      │
│ · MVP定义 + 批次交付计划         │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Phase 1: 需求澄清+拆解           │ 👤 用户介入：确认需求范围
│ · 需求澄清 → Goal构建            │
│ · 颗粒化拆解 → DAG + Skills分析  │
└──────────────┬───────────────────┘
               │
    ┌──────────┴──────────┐
    ▼                     ▼
┌────────────────┐  ┌────────────────┐
│ Phase 2: 原型  │  │ Phase 3: 开发  │
│ 设计（并行可选）│  │ 架构→编码→测试 │
│ 方向指标       │  │ 👤 架构审批    │
└────────────────┘  └───────┬────────┘
                            ▼
               ┌────────────────────────┐
               │ Phase 4: 打回循环（按需）│
               │ 最小必要回退            │
               └────────────┬───────────┘
                            ▼
               ┌────────────────────────┐
               │ Phase 5: 整合交付       │ 👤 用户介入：最终交付确认
               │ 复盘+经验沉淀           │
               └────────────────────────┘
                            ▼
                     使用者人工验收

👤 = 用户需要介入的节点
```

---

## 六、子Agent协作示例

### 场景：待办事项Web应用

用户说："帮我做一个待办事项Web应用，Vue3，支持增删改查。"

#### 任务拆解

| 任务ID | 描述 | 负责Agent | 依赖 |
|--------|------|----------|------|
| T001 | 技术调研（Vue3生态+最佳实践） | pm-researcher | - |
| T002 | 架构设计 | pm-orchestrator | T001 |
| T003 | 前端开发 | pm-coder | T002 |
| T004 | 后端开发 | pm-coder | T002 |
| T005 | 文档编写 | pm-writer | T003, T004 |

#### 执行时序

```
时间线 ──────────────────────────────────────────────────→

批次1:  [T001 技术调研]
              │
              ▼ T001完成，通知 orchestrator
              
批次2:  [T002 架构设计]
              │
              ▼ T002完成
              
批次3:  [T003 前端开发] ║ [T004 后端开发]     ← 并行执行
              │                    │
              ▼ T003完成           ▼ T004完成
              
批次4:  [T005 文档编写]
              │
              ▼ T005完成
              
交付 ← 整合所有产出物
```

---

## 七、故障排查

### 1. 子Agent 失败了怎么办？

| 情况 | 系统自动处理 | 如果自动处理也失败 |
|------|------------|------------------|
| 编译失败 | Coder 分析错误→修复→重试（最多2次） | 通知 orchestrator → 升级给你 |
| 工具调用报错 | 等待5秒→重试1次 | 通知 orchestrator |
| 上下文溢出 | 自动压缩到 HEARTBEAT → 请求重启 | 你确认重启 |
| 网络超时 | 等待→重试 | 通知 orchestrator |
| 敏感信息写入 | 🔒 写入前自动扫描→阻止→替换为环境变量 | 审计日志记录 |
| Agent连续失败 | 🛡️ 熔断机制→停止派发→30min后试探 | 通知 orchestrator → 你决定 |
| 验证不通过 | 🛡️ 自动回滚到最近安全检查点 | 通知 orchestrator |

**恢复优先级**：系统先自动恢复1次 → 失败才通知你 → 你决定下一步。

### 2. 上下文溢出怎么办？

系统内置 **HANDOFF（交接棒）机制**：

```
子Agent 感知上下文快满
    │
    ├── 冻结：停止新操作
    ├── 压缩：生成 HANDOFF.md（已做什么/正在做什么/下一步建议）
    ├── 同步：更新 HEARTBEAT
    └── 通知：告知 orchestrator
    
orchestrator 收到后：
    ├── 读取 HANDOFF.md
    ├── 启动新的 Agent
    └── 新 Agent 从断点无缝接手
```

**你不需要做任何事**，系统自动完成交接。

### 3. Skills 安装失败怎么办？

| 阶段 | 失败 | 回退方案 |
|------|------|---------|
| 远程仓库搜索无结果 | 远程仓库没有这个 Skill | 动态生成临时 Skill（基于任务描述自动创建） |
| 安装失败 | 网络问题/版本冲突 | 重试1次，仍失败则动态生成 |
| 动态生成也不行 | 极端情况 | 通知你手动处理 |

**三层兜底**：远程仓库安装 → 动态生成 → 人工介入。

### 4. 其他常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 子Agent卡住不动 | 可能是任务描述有歧义 | 检查 HEARTBEAT 的阻塞项，orchestrator 会自动超时升级 |
| 两个Agent改了同一个文件 | 文件所有权未明确 | orchestrator 在任务拆解时分配文件所有权，冲突时自动合并 |
| 调研结论格式不统一 | 不同Agent输出格式差异 | 系统统一使用结构化消息信封 + 调研报告模板 |
| 项目预算快用完 | 轮次消耗超过80% | 策略引擎自动预警，低优先级任务标记为 DEFERRED |

---

## 八、文件结构速查

### 核心文件

| 文件 | 用途 |
|------|------|
| `ARCHITECTURE.md` | 系统架构详细说明（技术参考） |
| `ARCHITECTURE_SPEC.md` | 架构说明书（v3规范） |
| `QUICKSTART.md` | 本文件（快速入门） |
| `README.md` | 项目简介 |

### Skill 定义

| 文件 | 用途 |
|------|------|
| `pm-orchestrator/SKILL.md` | 主控器行为规范 |
| `pm-backlog-manager/SKILL.md` | 需求池管理行为规范 |
| `pm-analyst/SKILL.md` | 需求澄清行为规范 |
| `pm-planner/SKILL.md` | 任务规划行为规范 |
| `pm-designer/SKILL.md` | 原型设计行为规范 |
| `pm-runner/SKILL.md` | 执行调度行为规范 |
| `pm-coder/SKILL.md` | 编码专家行为规范 |
| `pm-researcher/SKILL.md` | 调研专家行为规范 |
| `pm-writer/SKILL.md` | 文档专家行为规范 |

### 共享内核（pm-core）

| 文件 | 用途 |
|------|------|
| `pm-core/context-protocol.md` | 上下文自适应协议（FULL/PARTIAL/MINIMAL） |
| `pm-core/agent-lifecycle.md` | Agent 生命周期规范 |
| `pm-core/platform-adapter.md` | 平台抽象层（路径变量+操作映射） |
| `pm-core/security/permission-framework.md` | 🔒 权限强制执行框架（v3.1） |
| `pm-core/security/secrets-protection.md` | 🔒 敏感信息防护规则（v3.1） |
| `pm-core/security/audit-protocol.md` | 🔒 审计日志协议（v3.1） |
| `pm-core/stability/checkpoint-protocol.md` | 🛡️ 检查点与回滚（v3.1） |
| `pm-core/stability/circuit-breaker.md` | 🛡️ 三级熔断机制（v3.1） |
| `pm-core/stability/idempotency.md` | 🛡️ 幂等性规范（v3.1） |
| `pm-core/observability/heartbeat-v2.md` | 📊 HEARTBEAT v2协议（v3.1） |
| `pm-core/observability/metrics-protocol.md` | 📊 指标收集协议（v3.1） |
| `pm-core/observability/trace-protocol.md` | 📊 分布式追踪协议（v3.1） |
| `pm-core/references/recovery-recipes.md` | 失败恢复配方（8种失败类型+恢复步骤） |
| `pm-core/references/default-skills.md` | 6个默认运行时行为技能 |
| `pm-core/references/health-check-protocols.md` | 四层健康检查协议 |
| `pm-core/references/checkpoint-recovery.md` | 检查点与崩溃恢复方案 |
| `pm-core/references/message-protocol.md` | Agent间通信协议规范 |
| `pm-core/templates/heartbeat-template.md` | HEARTBEAT v2初始化模板（含YAML front matter） |

### Harness 定义（执行载体）

| 文件 | 用途 |
|------|------|
| `harnesses/README.md` | Skill vs Harness 概念说明 |
| `harnesses/orchestrator-harness.md` | 主控器运行环境配置 |
| `harnesses/backlog-manager-harness.md` | 需求池管理运行环境配置 |
| `harnesses/coder-harness.md` | 编码Agent运行环境配置（含6大策略） |
| `harnesses/researcher-harness.md` | 调研Agent运行环境配置 |
| `harnesses/writer-harness.md` | 文档Agent运行环境配置 |

---

## 九、进阶阅读

想深入了解？按以下顺序阅读：

1. **`README.md`** — 项目概览和完整使用场景
2. **`ARCHITECTURE.md`** — 完整架构设计（数据流、状态机、通信协议、Ironforge框架）
3. **`ARCHITECTURE_SPEC.md`** — v3.1架构说明书（核心创新点+八大Agent角色+Ironforge三大支柱）
4. **`ENTERPRISE_FRAMEWORK.md`** — Ironforge企业工程化框架完整设计文档
5. **`harnesses/orchestrator-harness.md`** — 主控器详细配置（策略引擎、健康检查、检查点恢复）
6. **`harnesses/coder-harness.md`** — 编码Agent六大策略（规划管制、上下文工程、风险权限、交接棒、钩子）
7. **`pm-core/security/`** — 安全协议三件套（权限框架+敏感信息防护+审计日志）
8. **`pm-core/stability/`** — 稳定性协议三件套（检查点+熔断+幂等性）
9. **`pm-core/observability/`** — 可观测性协议三件套（HEARTBEAT v2+指标+追踪）
10. **`pm-core/platform-adapter.md`** — 平台抽象层设计

---

> 💡 **一句话总结**：你说需求 → orchestrator 拆解 → 多个Agent并行执行 → 你在关键点确认 → 交付。中间出了问题，系统先自己恢复（🔒安全防护 + 🛡️检查点回滚 + 熔断隔离），恢复不了再找你。每个Skill也能独立运行——改Bug、写文档、做调研，一句话就搞定。
