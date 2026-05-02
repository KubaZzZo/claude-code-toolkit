---
name: dev-workflow
description: 项目开发总控工作流，按任务复杂度分派不同角色 Agent，控制成本
---

# Dev Workflow

接收用户任务后，按以下规则分派给对应角色 Agent。

## 角色与模型

| 角色 | 模型 | 职责 |
|------|------|------|
| pm-orchestrator | haiku | 需求分析、流程调度 |
| researcher | haiku | 查代码、查文档 |
| github-researcher | haiku | 搜索 GitHub 参考实现（本地无方案时） |
| architect | opus | 技术方案（复杂任务） |
| developer | sonnet | 代码实现 |
| tdd-engineer | sonnet | 写测试（实现前）、验证覆盖率（实现后） |
| reviewer | haiku | 代码质量审查 |
| qa-engineer | haiku | 测试与验证 |
| security-reviewer | opus | 安全审查（安全敏感任务） |
| performance-reviewer | haiku | 性能审查（DB/循环/缓存相关代码） |
| doc-writer | haiku | 更新文档（改动涉及公开 API 或功能时） |

## 调度规则

- 小任务：`researcher → developer → tdd-engineer(green) → qa-engineer`
- 普通功能：`researcher → architect → tdd-engineer(red) → developer → tdd-engineer(green) → reviewer → qa-engineer`
- 安全敏感：在上面基础上加 `security-reviewer`（在 reviewer 之前）
- 纯分析：只用 `researcher`

## tdd-engineer 触发规则

- **red 阶段**：在 developer 之前，写失败测试
- **green 阶段**：在 developer 之后，验证测试通过 + 覆盖率 ≥ 80%
- 简单改动（改名、文案）可跳过 red 阶段，只做 green 验证

## github-researcher 触发条件（满足任一即触发）

- researcher 在本地找不到现成实现模式
- 任务引入全新技术或库
- 需要了解业界最佳实践
- 触发位置：在 researcher 之后、architect 之前

## performance-reviewer 触发条件（满足任一即触发）

- 改动涉及数据库查询（ORM/SQL）
- 改动包含循环处理大数据集
- 改动涉及缓存逻辑
- 改动涉及外部 API 批量调用
- 触发位置：在 reviewer 之后、qa-engineer 之前

## doc-writer 触发条件（满足任一即触发）

- 改动涉及公开 API 接口（新增/修改/删除）
- 改动影响用户可见的功能行为
- 触发位置：在 qa-engineer 之后

---

## 门禁检查 (Gate Checks)

每个 Agent 完成后必须通过对应的门禁检查才能进入下一阶段。检查由 pm-orchestrator 执行。

| 阶段 | 门禁条件 | 检查方式 |
|------|----------|----------|
| researcher | relevant_files 非空，每个文件路径存在 | `test -f <path>` |
| github-researcher | references 非空，至少 1 个有效 URL | grep 检查 URL 格式 |
| architect | approach 非空，steps 至少 1 个，boundaries 明确 | 检查 JSON 字段完整性 |
| tdd-engineer(red) | tests_written ≥ 1，phase=red 时 verdict=fail（预期的失败） | pytest 运行确认失败 |
| developer | changed_files 非空，无 open_questions 阻塞项 | 检查 JSON 字段 |
| tdd-engineer(green) | 全部测试通过，覆盖率 ≥ 80% | `pytest` + 覆盖率工具 |
| reviewer | verdict ≠ block，无 critical 问题 | 检查 JSON verdict 字段 |
| security-reviewer | verdict ≠ block，无 critical 问题 | 检查 JSON verdict 字段 |
| performance-reviewer | verdict ≠ block | 检查 JSON verdict 字段 |
| qa-engineer | tests_run 非空，verdict=pass，无新增回归 | 检查 JSON 字段 |
| doc-writer | updated_files 非空（如果触发条件满足） | 检查文件存在 |

**门禁失败处理**：阻塞流水线，进入异常处理流程。

---

## 异常处理 (Exception Handling)

当 Agent 返回 `verdict: block` 或调用失败时，按以下链条处理：

### 重试规则

```
Agent 失败 → 最多重试 2 次（共 3 次尝试）
  - 第 1 次重试：原样重试，给 Agent 补充错误上下文
  - 第 2 次重试：降级策略（换更简单的方案/放宽约束）
```

### 降级策略

| Agent | 第 2 次重试降级策略 |
|-------|---------------------|
| architect | 跳过可选步骤，只做最小方案 |
| developer | 跳过边缘情况，只实现核心路径 |
| tdd-engineer | 只覆盖核心逻辑，覆盖率降到 70% |
| reviewer | 只看 critical + high 问题 |
| security-reviewer | 只看 OWASP Top 5 |

### 升级规则

```
3 次尝试后仍失败 → 生成升级报告 → 停止流水线 → 报告用户
```

### 升级报告格式

```json
{
  "escalation": {
    "failed_agent": "",
    "attempts": 0,
    "last_error": "",
    "impact": "影响后续哪些阶段",
    "recommendation": "建议人工介入的方式"
  }
}
```

### 非阻塞失败处理

- `verdict: warn` → 记录警告，继续流水线
- `verdict: partial` → pm-orchestrator 判断是否阻塞，默认继续但标记

---

## 对抗式审查 (Adversarial Review)

### reviewer 强制规则

1. **最小问题数**：每次审查必须找到至少 2 个实质问题（severity ≥ medium），否则标记为审查不充分
2. **文件验证**：至少对 3 个代码声明进行文件验证（读取实际文件确认），不能仅凭 plan/描述判断
3. **对抗姿态**：reviewer 的职责是找问题，不是通过审查。假设代码有 bug 直到找到证据证明没有

### 问题回应要求

developer/architect 处理 reviewer 反馈时：
- **每个问题必须逐一回应**，不允许静默忽略
- 接受：说明如何修复
- 拒绝：说明为什么不同意，必须有技术理由
- 推迟：说明何时处理、为什么不是现在

### 回溯审计

第二轮审查（如有）必须：
- 列出上一轮所有问题
- 逐条检查是否真正修复
- 未修复的标记为 `regression`

---

## 执行步骤

1. 调用 `pm-orchestrator` 分析任务，得到执行计划（含门禁清单）
2. 按计划顺序调用对应 Agent，传入上一步输出
3. 每步完成后执行门禁检查，失败则进入异常处理
4. 任何步骤触发升级时停止并报告用户
5. 完成后输出最终摘要

## 输出格式

```
## 任务摘要
[任务类型、复杂度、参与角色]

## 各角色结论
[每个角色的 summary + 门禁结果]

## 审查发现
[reviewer findings 数量、处理情况]

## 异常记录（如有）
[重试次数、降级情况、升级报告]

## 最终结论
[通过 / 需修复 / 阻塞，以及下一步建议]
```
