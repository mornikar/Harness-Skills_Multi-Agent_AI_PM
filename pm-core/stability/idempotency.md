# 幂等性规范

> **版本**：v3.1 Ironforge
> **定位**：pm-core/stability 层 — 操作可安全重试
> **原则**：同一操作执行一次和执行多次的结果完全相同

---

## 一、核心哲学

```
当前状态                              目标状态
──────────                            ──────────
Agent 中断后恢复 → 不确定已做了什么     Agent 恢复 → 检查 + 安全重试
文件可能被重复创建/覆盖                创建前先检查，覆盖需确认
命令可能被重复执行                     执行前检查前置条件
回滚后重试 → 可能重复操作              每步操作天然幂等
```

**定义**：操作 f 是幂等的，当且仅当 `f(f(x)) = f(x)` —— 执行一次和执行多次效果相同。

---

## 二、文件操作幂等性

### 2.1 write_to_file（创建文件）

| 属性 | 说明 |
|------|------|
| **默认行为** | 创建前先检查文件是否存在 |
| **幂等等级** | 条件幂等（需要检查前置条件） |

```yaml
write_to_file:
  rule: "创建前先检查文件是否存在"
  
  procedure:
    step_1:
      action: "read_file(path)"
      
    step_2:
      condition: "文件不存在"
      action: "write_to_file 创建"
      risk_level: "green"
      
    step_3:
      condition: "文件已存在"
      strategy:
        a:
          context: "plan.md 标记为'新建'"
          action: "确认覆盖（需审批，risk_level=yellow）"
          
        b:
          context: "plan.md 标记为'修改'"
          action: "改用 replace_in_file（更安全）"
          
        c:
          context: "上下文恢复场景（HANDOFF中有记录）"
          action: "直接覆盖（有交接记录保证安全性）"
          
        d:
          context: "文件内容与待写入内容完全一致"
          action: "跳过（幂等成功，无需操作）"
```

### 2.2 replace_in_file（修改文件）

| 属性 | 说明 |
|------|------|
| **默认行为** | 替换前验证 old_str 仍存在且唯一 |
| **幂等等级** | 自然幂等（old_str 不存在 = 已替换） |

```yaml
replace_in_file:
  rule: "替换前验证 old_str 仍存在且唯一"
  
  procedure:
    step_1:
      action: "search_content(pattern=old_str, path=target_file)"
      
    step_2:
      condition: "恰好1个匹配"
      action: "执行替换"
      risk_level: "green"
      
    step_3:
      condition: "0个匹配"
      action: "old_str已被替换或不存在 → 跳过（幂等成功）"
      risk_level: "null"
      note: "这是最关键的幂等保障：重复执行不会报错"
      
    step_4:
      condition: "多个匹配"
      action: "缩小 old_str 范围增加上下文，重试"
      risk_level: "yellow"
      
  # 增强幂等性的技巧
  idempotent_techniques:
    - "old_str 包含足够的上下文（前后各3行）"
    - "优先使用函数签名+注释作为 old_str 定位"
    - "避免使用可能重复的短字符串"
```

### 2.3 delete_file（删除文件）

| 属性 | 说明 |
|------|------|
| **默认行为** | 删除前确认文件存在且已备份 |
| **幂等等级** | 条件幂等（文件不存在 = 已删除） |

```yaml
delete_file:
  rule: "删除前确认文件存在且已备份"
  
  procedure:
    step_1:
      action: "read_file(path) → 确认存在"
      
    step_2:
      condition: "文件不存在"
      action: "跳过（幂等成功）"
      
    step_3:
      condition: "文件存在"
      action: "备份到 {context_root}/checkpoints/deleted/"
      
    step_4:
      action: "执行删除"
      risk_level: "red"
      
    step_5:
      action: "记录审计日志"
```

### 2.4 文件操作幂等性速查表

| 操作 | 幂等性 | 重复执行结果 | 前置检查 |
|------|--------|------------|---------|
| write_to_file（新建） | 条件幂等 | 覆盖或跳过 | 检查文件是否存在 |
| write_to_file（恢复） | 幂等 | 覆盖为相同内容 | 比较内容是否一致 |
| replace_in_file | 自然幂等 | 跳过（old_str已不存在） | 验证old_str存在且唯一 |
| delete_file | 幂等 | 跳过（文件已不存在） | 确认文件存在+备份 |
| read_file | 天然幂等 | 读取相同内容 | 无需检查 |

---

## 三、命令执行幂等性

### 3.1 命令分类

| 类别 | 命令 | 幂等性 | 前置检查 |
|------|------|--------|---------|
| **包管理** | `npm install` | ✅ 幂等 | `node_modules/.package-lock.json 存在 → 跳过` |
| **构建** | `npm run build` | ✅ 幂等 | 无（每次重新构建，重复无害） |
| **测试** | `npm run test` | ✅ 幂等 | 无（每次重新运行） |
| **代码检查** | `npm run lint` | ✅ 幂等 | 无 |
| **类型检查** | `tsc --noEmit` | ✅ 幂等 | 无 |
| **Python测试** | `python -m pytest` | ✅ 幂等 | 无 |
| **Python格式** | `python -m black --check` | ✅ 幂等 | 无 |
| **数据库迁移** | `prisma migrate deploy` | ⚠️ 条件幂等 | `检查 migration 是否已执行` |
| **Git提交** | `git commit` | ❌ 非幂等 | `检查是否有未提交变更` |
| **Git推送** | `git push` | ⚠️ 条件幂等 | `检查远程是否有新commit` |
| **部署** | `deploy` | ❌ 非幂等 | `检查当前部署版本` |

### 3.2 非幂等命令处理

```yaml
non_idempotent_commands:
  git_commit:
    pre_check: |
      1. git status → 是否有未提交变更？
      2. git log → 是否已有相同 message 的 commit？
    strategy: |
      有变更 + 无重复commit → 执行提交
      无变更 → 跳过
      有重复commit → 跳过（可能上次已提交）
      
  git_push:
    pre_check: |
      1. git status → 是否有未推送的commit？
      2. git fetch → 远程是否有新commit？
    strategy: |
      有本地commit + 远程无新commit → 执行推送
      远程有新commit → 先 pull --rebase
      无本地commit → 跳过
      
  database_migration:
    pre_check: |
      1. 查询 migration 状态表
      2. 确认该 migration 是否已执行
    strategy: |
      未执行 → 执行迁移
      已执行 → 跳过
    design_requirement: "迁移脚本必须包含 IF NOT EXISTS 等幂等设计"
      
  deploy:
    pre_check: |
      1. 查询当前部署版本
      2. 与目标版本比较
    strategy: |
      版本不同 → 执行部署
      版本相同 → 跳过
```

---

## 四、恢复场景的幂等保障

### 4.1 检查点恢复后重试

```yaml
checkpoint_recovery_retry:
  principle: "恢复后从检查点下一步开始，已完成的步骤不再重复"
  
  procedure:
    step_1:
      action: "读取检查点，确定已完成的步骤"
      
    step_2:
      action: "验证已完成的步骤（文件hash校验）"
      
    step_3:
      condition: "文件hash一致"
      action: "确认步骤已完成，跳过"
      
    step_4:
      condition: "文件hash不一致"
      action: "回滚该步骤，重新执行"
      
  # 幂等保障
  guarantee: "每个步骤的操作都是幂等的，重试不会产生副作用"
```

### 4.2 Agent 交接后重试

```yaml
handoff_retry:
  principle: "新Agent通过HANDOFF了解已完成的工作，不会重复"
  
  procedure:
    step_1:
      action: "读取 HANDOFF.md"
      
    step_2:
      action: "验证 HANDOFF 中记录的已完成步骤"
      
    step_3:
      action: "从 HANDOFF 记录的下一步开始执行"
      
  # 幂等保障
  guarantee: "HANDOFF 包含 files_created 和 files_modified 列表，新Agent可据此判断"
```

### 4.3 熔断恢复后重试

```yaml
circuit_breaker_retry:
  principle: "HALF_OPEN 试探成功后，从最近检查点恢复"
  
  procedure:
    step_1:
      action: "读取最近 verification_passed 检查点"
      
    step_2:
      action: "验证检查点状态（文件hash校验）"
      
    step_3:
      action: "从检查点的下一步重新执行"
      
  # 幂等保障
  guarantee: "检查点记录了精确的文件状态，重试可确定性地恢复"
```

---

## 五、最佳实践

### 5.1 编写幂等操作的检查清单

- [ ] `write_to_file`：是否检查了文件已存在？
- [ ] `replace_in_file`：`old_str` 是否足够唯一？
- [ ] `delete_file`：是否确认了文件存在？
- [ ] 命令执行：是否有前置检查（pre_check）？
- [ ] 恢复场景：操作是否能安全重试？

### 5.2 反模式（非幂等模式）

```yaml
anti_patterns:
  - name: "无条件创建"
    bad: "write_to_file(path, content)  # 直接创建，可能覆盖"
    good: "if not exists(path) then write_to_file(path, content)"
    
  - name: "模糊替换"
    bad: "replace_in_file(old_str='}', new_str='}\n// comment')  # old_str不唯一"
    good: "replace_in_file(old_str='function foo() {', new_str='function foo() {\n// comment')"
    
  - name: "无条件删除"
    bad: "delete_file(path)  # 不检查存在性"
    good: "if exists(path) then backup_and_delete(path)"
    
  - name: "无检查命令"
    bad: "git commit -m 'feat: add login'  # 可能重复提交"
    good: "if has_unstaged_changes() then git commit -m 'feat: add login'"
```

### 5.3 各 Agent 幂等性要求

| Agent | 关键幂等操作 | 特别注意 |
|-------|-------------|---------|
| coder | 文件创建/修改/删除 | replace_in_file 的 old_str 唯一性 |
| runner | Agent spawn/任务派发 | 避免重复 spawn 同一任务 |
| analyst | 需求文档输出 | 文件创建前检查已存在 |
| planner | 任务拆解文档 | 文件创建前检查已存在 |
| designer | 原型文件输出 | 文件创建前检查已存在 |
| researcher | 调研报告输出 | 文件创建前检查已存在 |
| writer | 文档输出 | 文件创建前检查已存在 |
| orchestrator | 全局状态更新 | 状态更新操作幂等 |
| backlog-manager | 需求池排序 | 排序操作天然幂等 |

---

## 六、与审计日志的联动

| 事件类型 | 触发时机 | 详情字段 |
|---------|---------|---------|
| `idempotent_skip` | 幂等检查发现操作已完成，跳过 | `{operation, path, reason}` |
| `idempotent_retry` | 幂等重试成功 | `{operation, path, attempt}` |
| `idempotent_conflict` | 幂等检查发现冲突 | `{operation, path, expected, actual}` |

---

*v3.1 Ironforge — idempotency.md — 2026-04-24*
