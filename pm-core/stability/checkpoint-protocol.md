# 检查点与回滚协议

> **版本**：v3.1 Ironforge
> **定位**：pm-core/stability 层 — 确定性检查点恢复
> **原则**：从"文档级恢复配方"到"每个步骤有快照，崩溃后可确定性恢复"

---

## 一、核心哲学

```
当前状态                              目标状态
──────────                            ──────────
恢复配方是文档 → Agent理解执行          检查点是结构化数据 → 程序化恢复
崩溃后靠记忆重建                       崩溃后回滚到最近安全检查点
验证失败只能重做                       验证失败 → 回滚 → 从检查点重试
Agent交接信息可能丢失                   检查点包含完整状态快照
```

---

## 二、检查点类型

### 2.1 自动检查点（Auto Checkpoint）

| 属性 | 说明 |
|------|------|
| **触发** | 每个 plan step 完成后 |
| **存储** | `{context_root}/checkpoints/T{XXX}.jsonl` |
| **格式** | JSONL（追加写入） |
| **重要性** | 标准 |

```json
{
  "step": 3,
  "timestamp": "2026-04-23T14:30:00Z",
  "status": "step_completed",
  "files_created": ["src/components/UserForm.vue"],
  "files_modified": ["src/router/index.ts"],
  "file_hashes": {
    "src/components/UserForm.vue": "sha256:abc123...",
    "src/router/index.ts": "sha256:def456..."
  },
  "heartbeat_snapshot": {
    "status": "running",
    "progress": 60,
    "health_score": 82
  },
  "verification_result": null
}
```

### 2.2 验证点检查点（Verification Checkpoint）

| 属性 | 说明 |
|------|------|
| **触发** | 推理验证点（RC-P3-02等）通过后 |
| **重要性** | high — 回滚优先使用验证点检查点 |
| **额外字段** | `verification_id`, `verification_detail` |

```json
{
  "step": 5,
  "timestamp": "2026-04-23T15:00:00Z",
  "status": "verification_passed",
  "files_created": ["src/components/UserForm.vue", "src/views/UserPage.vue"],
  "files_modified": ["src/router/index.ts", "src/store/user.ts"],
  "file_hashes": {
    "src/components/UserForm.vue": "sha256:abc123...",
    "src/views/UserPage.vue": "sha256:ghi789...",
    "src/router/index.ts": "sha256:jkl012...",
    "src/store/user.ts": "sha256:mno345..."
  },
  "heartbeat_snapshot": {
    "status": "running",
    "progress": 80,
    "health_score": 90
  },
  "verification_result": "passed",
  "verification_id": "RC-P3-02",
  "verification_detail": "代码与原型一致性检查通过"
}
```

### 2.3 交接检查点（Handoff Checkpoint）

| 属性 | 说明 |
|------|------|
| **触发** | Agent 交接时（HANDOFF 写入前） |
| **重要性** | critical — 确保交接信息不丢失 |
| **额外字段** | `handoff_to`, `handoff_reason` |

---

## 三、检查点写入协议

### 3.1 写入流程

```
Plan Step 完成
      │
      ▼
┌─────────────────┐
│ 1. 计算文件哈希  │  对所有 files_created + files_modified 计算 sha256
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. 拍摄心跳快照  │  读取当前 HEARTBEAT 状态
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 3. 构建检查点    │  组装 JSON 对象
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. 追加写入      │  append 到 T{XXX}.jsonl
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 5. 审计记录      │  写入审计日志 checkpoint_created 事件
└─────────────────┘
```

### 3.2 写入规范

| 规范 | 说明 |
|------|------|
| **原子性** | 每个检查点是一个完整的 JSON 行，不可跨行 |
| **追加写入** | 只 append，不修改历史条目 |
| **哈希计算** | 使用 sha256，对文件完整内容计算 |
| **不可变** | 检查点写入后不可修改 |

### 3.3 文件哈希计算

```python
# 伪代码
import hashlib

def compute_file_hash(file_path: str) -> str:
    content = read_file(file_path)
    return f"sha256:{hashlib.sha256(content.encode()).hexdigest()[:16]}"
```

---

## 四、回滚机制

### 4.1 回滚触发条件

| 条件 | 严重度 | 动作 |
|------|--------|------|
| 验证点不通过，重试2次仍失败 | HIGH | 回滚到最近 verification_passed 检查点 |
| Agent 崩溃且无法自动恢复 | CRITICAL | 回滚到最近检查点 + 交接给新Agent |
| 文件损坏（hash校验失败） | CRITICAL | 回滚到最近 hash 一致的检查点 |
| 人工触发回滚 | MANUAL | 回滚到指定检查点 |

### 4.2 回滚流程

```
回滚触发
    │
    ▼
┌──────────────────────────┐
│ Step 1: 定位安全检查点     │
│ 找到最近的                  │
│ status=verification_passed │
│ 的检查点                    │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Step 2: 文件还原           │
│ 遍历当前检查点之后的       │
│ 所有检查点：               │
│ - files_created → 删除    │
│ - files_modified →        │
│   恢复hash对应版本         │
│ 优先: git checkout        │
│ 降级: 逆向操作             │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Step 3: 状态还原           │
│ 恢复 HEARTBEAT 到         │
│ 检查点的 snapshot          │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Step 4: 重新执行           │
│ 从检查点的下一步开始       │
│ 重新执行 plan              │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ Step 5: 审计记录           │
│ 写入审计日志               │
│ rollback 事件              │
└──────────────────────────┘
```

### 4.3 文件还原策略

```
还原策略选择（按优先级）：

1. Git Checkout（如果项目在 git 管理下）
   git checkout <commit_hash> -- <file_path>
   → 最可靠，但需要 git 历史

2. 检查点恢复（如果存储了完整文件内容）
   从检查点存储的文件快照恢复
   → 需要检查点包含完整内容

3. 逆向操作（降级方案）
   - files_created → delete_file（删除新创建的文件）
   - files_modified → replace_in_file（用旧内容替换）
   → 依赖检查点的 file_hashes 做一致性校验
```

### 4.4 回滚安全保障

| 保障 | 说明 |
|------|------|
| **回滚前备份** | 回滚操作前，先备份当前状态作为新检查点 |
| **回滚验证** | 回滚后验证文件 hash 与检查点一致 |
| **回滚次数限制** | 同一任务最多回滚2次，超过则上报 orchestrator |
| **审计记录** | 每次回滚都有完整的审计日志 |

---

## 五、检查点清理

### 5.1 清理策略

```
任务 COMPLETED 后：

保留：
  ├── 第一个检查点（任务起始状态）
  ├── 最后一个检查点（任务最终状态）
  └── 所有 verification_passed 检查点（关键恢复点）

删除：
  └── 中间的 step_completed 检查点（减少存储占用）
```

### 5.2 清理时机

| 时机 | 动作 |
|------|------|
| 任务 COMPLETED | 执行清理策略 |
| 项目完成 | 保留所有 verification_passed，删除其他 |
| 存储超过 100MB | 触发紧急清理，只保留最近3天的检查点 |

---

## 六、与 Harness 的集成

### 6.1 Harness 侧的检查点配置

每个 Harness 可通过 `checkpoint_override` 覆盖默认检查点行为：

```yaml
# 在特化层声明
specialization:
  checkpoint_override:
    # 附加触发条件（在默认的"每step完成后"之外）
    additional_trigger: "每次 replace_in_file 后"
    
    # 检查点频率覆盖
    frequency: "every_step"           # every_step | every_n_steps | verification_only
    
    # 保留策略覆盖
    retention: "all"                   # all | minimal | verification_only
    
    # 回滚策略覆盖
    rollback_strategy: "git_first"     # git_first | checkpoint_first | reverse_only
```

### 6.2 各 Agent 默认检查点策略

| Agent | 默认频率 | 保留策略 | 回滚策略 | 特殊说明 |
|-------|---------|---------|---------|---------|
| coder | every_step | all | git_first | 文件修改密集，每步检查点 |
| runner | every_step | minimal | git_first | 需要快速回滚 |
| analyst | verification_only | verification_only | reverse_only | 只读为主，检查点少 |
| planner | verification_only | verification_only | reverse_only | 只读为主 |
| designer | every_step | all | git_first | 原型文件修改密集 |
| researcher | verification_only | verification_only | reverse_only | 纯只读 |
| writer | every_step | all | git_first | 文档修改密集 |
| orchestrator | every_step | minimal | git_first | 需要追踪全局状态 |
| backlog-manager | verification_only | verification_only | reverse_only | 只读为主 |

---

## 七、与审计日志的联动

每次检查点操作都会生成审计事件：

| 事件类型 | 触发时机 | 详情字段 |
|---------|---------|---------|
| `checkpoint_created` | 检查点写入 | `{step, status, file_count, storage_size}` |
| `checkpoint_rollback` | 回滚执行 | `{from_step, to_step, files_restored, reason}` |
| `checkpoint_cleanup` | 检查点清理 | `{retained_count, deleted_count, space_freed}` |
| `checkpoint_verify` | 回滚后验证 | `{step, hash_match, files_verified}` |

---

## 八、与熔断机制的联动

检查点失败计数参与熔断决策：

```yaml
circuit_breaker_integration:
  # 连续检查点验证失败 → 触发Agent级熔断
  checkpoint_failure:
    threshold: 2                       # 连续2次检查点验证失败
    action: "触发 agent_level 熔断"
    
  # 回滚次数过多 → 触发任务级熔断
  rollback_exhaust:
    threshold: 2                       # 同一任务回滚2次
    action: "触发 task_level 熔断（exhaust_action）"
```

---

*v3.1 Ironforge — checkpoint-protocol.md — 2026-04-24*
