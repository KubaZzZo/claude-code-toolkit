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

- 小任务：`researcher → tdd-engineer(red) → developer → tdd-engineer(green) → qa-engineer`
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

## 执行步骤

1. 调用 `pm-orchestrator` 分析任务，得到执行计划
2. 按计划顺序调用对应 Agent，传入上一步输出
3. 任何步骤返回 `verdict: block` 时停止并报告
4. 完成后输出最终摘要

## 输出格式

```
## 任务摘要
[任务类型、复杂度、参与角色]

## 各角色结论
[每个角色的 summary]

## 最终结论
[通过 / 需修复 / 阻塞，以及下一步建议]
```
