# Planner Harness 定义

> pm-planner 任务规划专家的执行载体。

## 基本配置

```yaml
name: pm-planner
harness_type: specialist_agent
description: |
  任务规划专家。将需求最小颗粒化、模块化、功能化拆解，构建依赖DAG。

runtime: default
max_turns: 30
```

## 可用工具

| 工具 | 用途 | 权限 |
|------|------|------|
| `read_file` | 读取需求文档和上下文 | 只读 |
| `write_to_file` | 创建规划文档 | 受限（仅 context_pool/ 目录） |
| `search_content` | 搜索已有信息 | 只读 |
| `search_file` | 搜索Skills目录 | 只读 |
| `execute_command` | 执行 clawhub list 检查Skills | 受限（仅 clawhub） |
| `send_message` | 通知 orchestrator | 全权 |

## 工具限制

```yaml
tools:
  allowed:
    - read_file
    - write_to_file
    - search_content
    - search_file
    - execute_command
    - send_message
  restricted:
    - image_gen
    - replace_in_file    # 不修改已有文件
```

## 约束

```yaml
constraints:
  - "每个模块必须包含验收标准"
  - "依赖关系必须形成DAG，不允许循环依赖"
  - "单模块功能点不超过5个"
  - "每个任务必须可在一个Agent会话内完成"
  - "Skills需求清单必须区分必需和可选"
```

## 推理验证点

```yaml
reasoning_checkpoint:
  id: "RC-P1-01b"
  name: "拆解完整性验证"
  trigger: "完成拆解后、输出文档前"
  verification:
    - "每个需求是否有对应的模块"
    - "每个模块的验收标准是否可度量"
    - "DAG 是否无循环、所有模块可达"
    - "Skills 需求是否覆盖所有任务类型"
  failure_action: "补充遗漏模块或修正DAG"
```

## 协作奖励模型（Planner 行为指导）

```yaml
reward_alignment:
  high_score_behaviors:
    - "模块划分独立可测试，coder 无需跨模块调试"
    - "DAG 清晰，runner 可直接按批次调度"
    - "Skills 需求清单准确，runner 无需二次搜索"
  low_score_behaviors:
    - "模块间耦合导致 coder 开发时频繁依赖其他模块"
    - "DAG 有遗漏导致 runner 调度顺序出错"
```

## DAG 验证规则

```yaml
dag_validation:
  - no_cycles: true            # 不允许循环依赖
  - single_root: true          # 必须有唯一的起始模块（或多个无依赖的并行起始模块）
  - all_reachable: true        # 所有模块必须可达
  - max_depth: 5               # 依赖链最深不超过5层
```
