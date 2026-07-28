---
name: handoff-save
description: Use when user requests a handoff document before /compact, asks to save current session state for later resumption, or says phrases like "给我一份 handoff", "保存 handoff", "compact 前", "记录当前进度", "save handoff", "before compact". Designed for the handoff → /compact → resume workflow.
---

# Handoff Save

## 概述

把当前会话的状态持久化为 markdown 文件，写入当前 worktree 内，便于将来的会话（在 `/compact` 之后或新对话里）恢复上下文而不丢失关键信息。

本 skill 只负责写文件。`/compact` 由用户自己执行。恢复由配套的 `handoff-resume` skill 处理。

## 输出位置

文件路径：`<worktree-root>/.handoff/<branch-slug>--<topic-slug>--<YYYYMMDD-HHMMSS>.md`

- `<worktree-root>` —— `git rev-parse --show-toplevel`，**没用 `git worktree` 的话就是仓库根目录，无需额外配置**。不在 git 仓库时回退到当前工作目录
- `<branch-slug>` —— `git branch --show-current`，把 `/` 和空白字符替换为 `-`
- `<topic-slug>` —— 由当前任务概要生成的 2-4 个英文 kebab-case 单词（如 `jwt-refactor`、`pr-review-fix`）
- 分隔符必须是**双横线** `--`，避免 branch 或 topic 内部的单 `-` 破坏解析
- 时间戳使用本地时间，格式 `YYYYMMDD-HHMMSS`

如果 `.handoff/` 目录不存在，先创建它。

## .gitignore 检查（强制）

写入文件前，检查 `.handoff/` 是否已被 git 忽略：

```bash
git check-ignore -q .handoff/ 2>/dev/null && echo IGNORED || echo NOT_IGNORED
```

- `IGNORED` → 静默继续
- `NOT_IGNORED` → 文件照样写入，但写入后向用户提示：
  > `.handoff/` 没有被 git 忽略。建议在 `.gitignore` 中添加 `.handoff/`。是否需要我加上？
- 未经用户明确批准，不要修改 `.gitignore`

## 文档模板

每一节都要填。如果某节确实没有内容，写 `（无）`，不要省略——读者需要知道这一节被认真考虑过。

```markdown
# Handoff: <topic>

- **保存时间**: <ISO 8601 本地时间戳>
- **Branch**: <当前 branch>
- **Worktree**: <绝对路径>
- **保存时的 CWD**: <pwd>

## 1. 任务目标
一句话目标 + 为什么做（动机/约束/截止）。

## 2. 关键决策
逐条列出。每条格式：
- **决策**: <做了什么选择>
- **理由**: <为什么这样选>
- **否决的备选**: <考虑过但没选的方案 + 不选的原因>

## 3. 进度状态
### 已完成
- ...
### 进行中
- ...（标注完成度，例如"3/5 文件已改"）
### 未开始
- ...

## 4. 代码现场
正在改的文件、行号、关键片段、读过但没改的相关文件。
- `<path>:<line>` —— 简述当前状态
- 必要时贴关键片段（10 行以内）

## 5. 已尝试但失败的路径
防止 resume 后重复踩坑。
- 尝试: <做法> | 失败原因: <为什么不行>

## 6. 中断时的状态
**描述性陈述句，禁止使用祈使语气。** 描述当时停在哪一步、刚做完什么、下一步本来想做什么。

## 7. 候选下一步（仅供参考，需用户确认）
**禁止使用"do X"、"修改 Y"、"立即执行"等祈使语气。** 全部写成选项：
- 选项 A: <描述> —— 适用于 <场景>
- 选项 B: <描述>
- 选项 C: <描述>

## 8. 开放问题 / 等待用户输入
- ...（resume 时新会话需要从这里恢复对话）

## 9. 环境快照
- **Git branch**: <branch>
- **Git status**: <未提交改动数 + 大致内容>
- **Worktrees**（`git worktree list`）: 多个时全部列出并标注当前；只有一个（默认情况，未用 `git worktree add`）写"仅当前一个"即可
- **后台进程**: 启动了哪些（dev server、watcher 等）
- **已加载外部资源**: 重要的 URL、已 fetch 的文档、已读的 MCP 资源
- **当前 TodoList**: 复制 TodoWrite 当前的全部条目
- **持久授权**（进行中流程的落盘授权凭证，供 resume 端双源核验；没有就写（无））:
  - 流程: <哪个 skill 的哪个流程，例：pr-review-loop PR #14 评审循环自动挡>
  - 凭证: <文件绝对路径> 中 <字段> == <期望值>（例：`<WT>/.pr-review-loop/pr-14.json` 中 `mode` == `auto`）
  - 重入方式: <一句话，例：重跑 /pr-review-loop 续跑该 PR>
  - 授权范围: <该授权覆盖的动作边界>

## 10. 隐含假设 与 用户偏好/约束
- **隐含假设**（resume 时应验证）: ...
- **对话中浮现的偏好/约束**: ...
```

## 写入规则

1. **不要编造** —— 只记录会话中真实发生过的内容。某字段未知就写 `（未知）`，不要猜。
2. **第 6、7 节禁止祈使语气** —— 这两节的设计目的是阻止恢复端的 AI 自动开干，必须保持描述性。
3. **优先具体而非概括** —— 写文件路径而不是"那个 auth 模块"，写行号而不是"大概在顶部"。
4. **保留关键工具输出原文** —— 影响过决策的测试失败、错误信息、文档片段，相关时整段贴入。
5. **不要省略章节** —— 空就写 `（无）`。
6. **持久授权只认落盘凭证** —— 第 9 节该字段只能记录此刻真实存在的机器可读文件（路径 + 字段 + 期望值），且须由流程自身的 opt-in 契约写入。用户在对话里说过"行、继续"不算——那是会话级授权，随会话结束失效，resume 端会重新问。给不出路径就写 `（无）`。

## 收尾输出

写完文件后，向用户输出：

1. 已保存的文件路径，**必须使用 Markdown 链接格式** `[<文件名>](<绝对路径>)`，让 Claude Code UI 把它渲染成可点击链接——裸文本路径不可点击
   - 链接文本用文件名即可（例如 `feature-auth--jwt-refactor--20260507-153022.md`）
   - 链接 URL 必须是完整绝对路径（开头 `/`）

2. **`.gitignore` 提示——只在 `NOT_IGNORED` 时输出**，具体提示文字见上文 ".gitignore 检查" 段的 `NOT_IGNORED` 模板。`IGNORED` 时**保持静默，不要输出任何 gitignore 相关字句**（"已被 .gitignore 覆盖" / "无需额外处理" 这种确认句也不要写）。

3. **恢复指令建议**——给出可直接复制粘贴的 resume 命令，文件名用刚写的那个：

   先判断 skill 安装形态（用 `test -d` 一条链路查，避免多次 stat 和 stdout 污染）：

   ```bash
   if test -d ~/.claude/plugins/cache/handoff-skills/handoff-resume; then echo PLUGIN; elif test -d ~/.claude/skills/handoff-resume; then echo SKILL_DIRECT; else echo UNKNOWN; fi
   ```

   **格式要求**：命令必须**单独占一行**，跟前缀 "恢复时运行：" 分开。前缀末尾加冒号 + 换行，命令独立成行——这样窄终端不会折行错位，用户也能整行选中复制。

   - `PLUGIN`（plugin marketplace 装的）→ 输出：

     ```
     恢复时运行：
     /handoff-resume:handoff-resume <文件名>
     ```

   - `SKILL_DIRECT`（直接装到 `~/.claude/skills/`，例如 `npx skills` 装的）→ 没有 slash 命令，用自然语言触发：

     ```
     恢复时对 agent 说：
     从 handoff 恢复 <文件名>

     （或 "/handoff-resume <文件名>" 也行，如果你手动加了同名 command file）
     ```

   - `UNKNOWN`（两个都没有 / 没装 resume skill）→ 两条都列出：

     ```
     恢复时（取决于安装方式）：

     plugin marketplace 装：
     /handoff-resume:handoff-resume <文件名>

     ~/.claude/skills/ 直装，对 agent 说：
     从 handoff 恢复 <文件名>
     ```

4. 收尾提示：`保存完成。你可以现在运行 /compact，之后用上面的命令恢复。`

## 不适用场景

- 纯只读的问答会话，没有任何决策或修改 —— 没东西可交接
- 用户准备继续在当前会话里干活、不会跑 `/compact`
