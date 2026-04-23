# Harness 公共层（Base Layer）

> Ironforge 企业工程化框架核心组件。
> v3.1 新增。将9个Harness中的公共逻辑抽取到统一层，改一处生效全局。
>
> 每个 Agent 的 Harness 通过 `inherits` 声明引用这些公共文件，
> 只在特化层中声明自己独有的配置和逻辑。

---

## 架构说明

```
harnesses/
├── base/                              # ← 你在这里
│   ├── README.md                      # 分层架构说明（本文件）
│   ├── permission-framework.md        # 权限强制执行（三层模型 + spawn配置）
│   ├── security-hooks.md             # 安全钩子（敏感信息扫描 + 受保护路径）
│   ├── audit-logging.md              # 审计日志（统一格式和事件定义）
│   ├── checkpoint-protocol.md        # 检查点（自动保存 + 回滚规范）
│   ├── handoff-protocol.md           # 交接棒（触发条件 + 流程 + 模板）
│   ├── context-engineering.md        # 上下文工程（分层 + 预算 + 噪声过滤）
│   └── observability-config.md       # 可观测性（HEARTBEAT + 通信事件）
│
├── coder-harness.md                   # 继承 base/ + coder 特化
├── runner-harness.md                  # 继承 base/ + runner 特化
└── ...                                # 其他7个Agent的特化Harness
```

---

## 继承机制

```yaml
# 每个 Harness 通过 inherits 声明引用公共层
inherits:
  - base/permission-framework.md
  - base/security-hooks.md
  - base/audit-logging.md
  - base/checkpoint-protocol.md
  - base/handoff-protocol.md
  - base/context-engineering.md
  - base/observability-config.md

# 然后只声明特化配置
specialization:
  permission_override: { ... }         # 权限覆盖
  hooks_override: { ... }              # 钩子覆盖
  agent_specific: { ... }              # Agent特有逻辑
```

---

## 覆盖规则

1. **公共层定义默认值**，特化层可以覆盖
2. **特化层的 `add` 追加，`remove` 移除**
3. **公共层必须覆盖所有 Agent 的最小公共集**
4. **特化层不允许降低安全级别**（只能更严格，不能更宽松）

```yaml
# ✅ 正确：特化层增加安全约束
permission_override:
  blocked_tools:
    add: [delete_file]                 # coder额外禁用delete_file

# ❌ 错误：特化层降低安全级别
permission_override:
  blocked_tools:
    remove: [execute_command]          # 不允许从blocked列表移除
```

---

## 维护原则

- **修改公共层**：影响所有继承的Harness，需全量回归验证
- **修改特化层**：只影响对应Agent，局部验证即可
- **新增公共功能**：在 base/ 添加文件，各Harness的 inherits 追加引用
- **删除公共功能**：从 base/ 移除，各Harness的 inherits 移除引用

---

*Ironforge Base Layer v1.0 — 2026-04-23*
