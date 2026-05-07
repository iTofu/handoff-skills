---
name: handoff-resume
description: Use when user wants to resume work from a previously saved handoff document, typically after running /compact, or when starting a session that continues earlier work. Triggers on "/handoff-resume", "从 handoff 恢复", "继续上次的工作", "resume handoff", "compact 之后继续".
---

# Handoff Resume

## 概述

加载先前由 `handoff-save` 写入 `<worktree-root>/.handoff/` 的 handoff 文件，让新会话进入之前的上下文——**在用户明确批准方向之前不得执行任何修改操作**。

## 文件选择

根据是否带参数决定行为。

### 不带参数

1. 找到当前 worktree 根：`git rev-parse --show-toplevel`，不在 git 仓库时回退到当前工作目录
2. 找到当前 branch：`git branch --show-current`
3. 列出 `.handoff/<branch-slug>--*.md`（按当前 branch 过滤）
4. 按文件名中的时间戳倒序排列
5. 根据匹配数量分支：
   - **0 个** → 告知用户找不到，停止。提示用户可改用 `/handoff-resume <关键词>` 或 `/handoff-resume <绝对路径>` 扩大范围
   - **1 个** → 向用户显示文件名，询问 `是这个吗？`，等待回复。**不要自动加载。**
   - **多个** → 展示前 5 条 `主题 — 时间戳 — 大小`，请用户选择

**展示文件路径时**：必须使用 Markdown 链接格式 `[<文件名>](<绝对路径>)`，让 Claude Code UI 渲染成可点击链接。链接文本用文件名（不含目录），URL 用完整绝对路径。同样适用于"是这个吗？"的文件展示和多文件列表。

### 带参数

- 参数是绝对路径 → 向用户确认后加载
- 参数是 `.handoff/` 内的文件名 → 向用户确认后加载
- 参数是关键词 → 在当前 worktree 的 `.handoff/*.md` 全集（不限 branch）中模糊匹配，列出匹配项请用户选择

**跨 worktree**：不要主动搜索其他 worktree。如果用户需要从别的 worktree 恢复，必须自己传绝对路径。

## 确认门规则（核心纪律）

这是本 skill 最重要的部分。**用户明确批准方向之前，禁止执行任何修改动作。**

用户确认要加载哪个文件之后，读取文件，然后**在调用任何 Edit / Write / NotebookEdit / 写入型 Bash 命令之前**：

1. 用 3-5 句话向用户复述你的理解，覆盖：
   - 当时在做什么（任务目标）
   - 工作中断在哪里（handoff 第 6 节）
   - 最关键的决策或约束
2. 把 handoff 第 7 节"候选下一步"原样列出。如果你认为有新的选项值得提出，加在后面并明确标注 `新增选项`
3. 明确询问：`请问要按哪个方向继续，还是有其他想法？`
4. **等待**用户回复。不得继续执行。

用户给出明确方向之后：
- 确认门已关闭，**回归正常的 Agent 行为**
- 不要在后续每次写操作前额外加确认。本规则只在恢复入口生效一次，不是会话级的常态约束

## Red Flags —— 立即停止

读完 handoff 文件、用户尚未明确批准方向之前，如果出现以下任一情况，说明你已经违反本 skill：

- 调用了 `Edit` / `Write` / `NotebookEdit` 或写入型 `Bash` 命令
- 看到 handoff 提到的问题就"开始动手修"
- 觉得"选项 A 很明显"就直接照做
- 说"我先按 X 进行"却没等用户回复
- 觉得"handoff 写得很清楚"而跳过了复述步骤

任何一条命中：停下，致歉，从确认门规则的第 1 步重新开始。

## 常见错误

| 错误 | 修正 |
|---|---|
| 只匹配到 1 个文件就自动加载 | 即使只有 1 个，也要先把文件名给用户看并询问 |
| 跳过 3-5 句的复述 | 复述是用户在你动手前发现误解的最后一道防线 |
| 把第 7 节当成执行队列 | 那是讨论用的选项清单，不是 to-do |
| 后续每次写操作都加确认 | 确认门只在恢复入口生效，之后按用户平时的设置走 |
| 没问就跨 worktree 搜 | 当前 worktree 内搜索，跨 worktree 必须用户传路径 |

## 边界情况

- **`.handoff/` 不存在** → 告知用户，停止。不要创建（创建是 save skill 的职责）
- **handoff 文件陈旧**（如时间戳超过 7 天、branch 已大幅推进）→ 仍然加载，但在复述里明确指出陈旧度，让用户决定是否跳过部分章节
- **handoff 引用的文件已不存在** → 在复述里标记，不要默默忽略
- **不在 git 仓库** → 仍可工作；把当前工作目录视为 worktree 根，跳过 branch 过滤，列出 `.handoff/` 内全部文件
