# 平台适配器（Platform Adapter）

> v3 新增。定义 Skill 体系与宿主平台之间的抽象层。
> 所有 SKILL.md、pm-core、orchestrations 通过变量和操作描述与平台解耦。
> 每个平台提供自己的适配映射。

---

## 1. 路径变量映射

| 变量名 | 含义 | WorkBuddy | Cursor | Claude Code |
|--------|------|-----------|--------|-------------|
| `{context_root}` | 项目上下文目录 | `.workbuddy` | `.cursor` | `.claude` |
| `{skills_root}` | Skill 安装根目录 | `~/.workbuddy/skills` | `~/.cursor/skills` | `~/.claude/skills` |
| `{context_pool}` | 上下文池 | `{context_root}/context_pool` | `{context_root}/context_pool` | `{context_root}/context_pool` |
| `{checkpoints}` | 检查点目录 | `{context_root}/checkpoints` | `{context_root}/checkpoints` | `{context_root}/checkpoints` |
| `{heartbeat}` | 项目心跳文件 | `{context_root}/HEARTBEAT.md` | `{context_root}/HEARTBEAT.md` | `{context_root}/HEARTBEAT.md` |
| `{prototype}` | 原型目录 | `{context_pool}/prototype` | `{context_pool}/prototype` | `{context_pool}/prototype` |

---

## 2. 操作映射

SKILL.md 和 pm-core 中使用**操作描述**（左列），各平台映射为具体工具（右列）：

| 操作描述 | 含义 | WorkBuddy | Cursor | Claude Code |
|---------|------|-----------|--------|-------------|
| 读取文件 `{path}` | 读取指定文件内容 | `read_file` | Read tool | `Read` |
| 创建/写入文件 `{path}` | 创建或覆盖写入文件 | `write_to_file` | Write tool | `Write` |
| 修改文件 `{path}` 中的指定内容 | 精确替换文件中的部分内容 | `replace_in_file` | Edit tool | `Edit` |
| 列出目录 `{path}` 内容 | 列出目录下的文件和子目录 | `list_dir` | ls / dir | `LS` |
| 搜索匹配 `{pattern}` 的内容 | 在文件中搜索正则匹配 | `search_content` | Search tool | `Grep` |
| 搜索文件 `{pattern}` | 按文件名模式搜索文件 | `search_file` | Find tool | `Glob` |
| 执行命令 `{cmd}` | 在终端执行命令 | `execute_command` | Terminal | `Bash` |
| 向 `{recipient}` 发送消息 | Agent 间通信 | `send_message` | (via subagent) | (via subagent) |
| 创建/启动 Agent `{name}` | 生成子Agent执行任务 | `Task tool + name` | Agent spawn | `Task tool` |
| 通过 Skill 管理器搜索/安装 `{name}` | 搜索和安装 Skill | `clawhub search/install` | 手动安装 | 手动安装 |
| 通过 Skill 管理器列出已安装 | 列出已安装 Skill | `clawhub list` | ls skills dir | ls skills dir |
| 图片生成 `{description}` | 根据描述生成图片 | `image_gen` | (MCP tool) | (MCP tool) |

---

## 3. 使用规范

### SKILL.md 中的写法

```markdown
✅ 正确：读取文件 `{context_pool}/goal.md`
❌ 错误：read_file(".workbuddy/context_pool/goal.md")

✅ 正确：向 orchestrator 发送消息
❌ 错误：send_message(type="message", recipient="main", ...)

✅ 正确：创建 Agent `pm-coder`
❌ 错误：task(name="pm-coder", mode="acceptEdits", ...)

✅ 正确：通过 Skill 管理器搜索 `vue3`
❌ 错误：clawhub search "vue3"
```

### Harness 中的写法

Harness 是特定平台的执行配置，**可以且应该**使用平台专有工具名和配置。

```markdown
✅ harnesses/coder-harness.md 中：
read_file("pm-coder/SKILL.md")
mode: "acceptEdits"
max_turns: 50
```

---

## 4. 各平台适配指南

### WorkBuddy

WorkBuddy 是当前主要适配平台，harnesses/ 目录提供完整适配。

- 变量映射：见上表
- 操作映射：见上表
- 执行配置：harnesses/*.md
- Skill 管理器：clawhub CLI
- Agent 创建：Task tool（team mode）
- Agent 通信：send_message tool

### Cursor

Cursor 使用 `.cursor/rules` 和 MCP 工具进行适配。

- 变量映射：见上表
- Skill 管理器：手动安装到 `~/.cursor/skills/`
- Agent 创建：Cursor Agent
- Agent 通信：通过文件系统（上下文池读写）

### Claude Code

Claude Code 使用 `.claude/` 目录和内置工具进行适配。

- 变量映射：见上表
- Skill 管理器：手动安装到 `~/.claude/skills/`
- Agent 创建：Task tool
- Agent 通信：通过文件系统（上下文池读写）

---

## 5. 新平台适配清单

为新的 Agent 平台适配时，需要：

1. 确定路径变量映射（5 个变量）
2. 确定操作映射（12 个操作）
3. 为每个 Skill 创建执行配置（可选，类似 harnesses/）
4. 在平台配置中注册 Skill 目录
