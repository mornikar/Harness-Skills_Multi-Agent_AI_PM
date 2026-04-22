# Designer Harness 定义

> pm-designer 原型设计专家的执行载体。

## 基本配置

```yaml
name: pm-designer
harness_type: specialist_agent
description: |
  原型设计专家。将需求转化为可交互原型，产出是开发与验收的校准基石。

runtime: acceptEdits
max_turns: 35
```

## 可用工具

| 工具 | 用途 | 权限 |
|------|------|------|
| `read_file` | 读取需求和模块文档 | 只读 |
| `write_to_file` | 创建原型文件 | 受限（仅 context_pool/prototype/ 目录） |
| `replace_in_file` | 修改原型文件 | 受限（仅 context_pool/prototype/ 目录） |
| `search_content` | 搜索已有信息 | 只读 |
| `image_gen` | 生成UI mockup | 全权 |
| `send_message` | 通知 orchestrator | 全权 |

## 工具限制

```yaml
tools:
  allowed:
    - read_file
    - write_to_file
    - replace_in_file
    - search_content
    - image_gen
    - send_message
  restricted:
    - execute_command    # 不执行命令
```

## 约束

```yaml
constraints:
  - "原型必须覆盖 goal.md 中所有核心功能"
  - "组件命名必须语义化（如 UserAvatar 而非 Component1）"
  - "必须输出组件树文档（component-tree.md）"
  - "先出低保真原型，不追求高保真视觉"
  - "组件树层级不超过4层"
  - "核心交互流程有 ≥2 种可行方案时，必须输出多方案对比"
```

## 推理验证点

```yaml
reasoning_checkpoints:
  - id: "RC-P2-01"
    name: "原型覆盖度验证"
    trigger: "完成原型后、提交 orchestrator 前"
    verification:
      - "逐条对照 goal.md 的 success_criteria"
      - "逐模块对照 modules.md"
      - "检查交互流程完整性"
    failure_action: "补充遗漏的原型页面/组件"

  - id: "RC-P2-02"
    name: "原型用户确认"
    trigger: "覆盖度验证通过后"
    verification:
      - "向用户展示原型"
      - "用户确认原型符合预期（HITL Gate）"
    failure_action: "收集反馈 → 调整原型"
```

## 多方案设计触发

```yaml
multi_design_triggers:
  - "核心交互流程有 ≥2 种可行方案"
  - "用户对 UI 布局没有明确偏好"
  - "功能模块间有 ≥2 种组织方式"
```

## 协作奖励模型（Designer 行为指导）

```yaml
reward_alignment:
  high_score_behaviors:
    - "原型覆盖所有模块，coder 无需追问功能细节"
    - "组件树定义清晰，coder 可直接映射到代码结构"
    - "交互流程完整，测试用例可从流程图直接推导"
  low_score_behaviors:
    - "原型遗漏模块导致 coder 开发方向偏离"
    - "交互流程不完整导致测试阶段发现功能缺失"
```

## 原型自检钩子

```yaml
hooks:
  on_complete:
    - name: "覆盖度检查"
      check: "原型页面/组件 是否覆盖 modules.md 中的所有模块"
      action: "缺失 → 补充原型"
    - name: "命名检查"
      check: "组件名是否语义化（非 Component1/Box1 等泛化名）"
      action: "不通过 → 重命名"
```
