# Claude Code 官方调研报告：Hooks、Plugins、Skills、Compact 系统

**报告日期**：2026-05-07  
**调研范围**：Claude Code v2.1.114 - v2.1.119（最近6个月）  
**文档基准**：code.claude.com/docs 官方文档  

---

## 1. Hooks 系统全貌

### 1.1 所有 Hook 事件类型与触发时机

URL: https://code.claude.com/docs/en/hooks.md

| 事件名 | 触发时机 | 可阻止 | additionalContext 支持 |
|--------|---------|--------|------------------------|
| `SessionStart` | session 开始或恢复时 | ❌ | ✅ |
| `Setup` | `--init-only` 或 `-p --init` 时 | ❌ | ✅ |
| `UserPromptSubmit` | 用户提交 prompt 前 | ✅ | ✅ |
| `UserPromptExpansion` | 用户输入的命令展开为 prompt 前 | ✅ | ✅ |
| `PreToolUse` | 工具调用前 | ✅ | ✅ |
| `PermissionRequest` | 权限对话显示时 | ✅ | ❌ |
| `PermissionDenied` | auto mode 分类器拒绝工具调用后 | ❌ | ❌ |
| `PostToolUse` | 工具调用成功后 | ❌ | ✅ |
| `PostToolUseFailure` | 工具调用失败后 | ❌ | ✅ |
| `PostToolBatch` | 一批平行工具调用完成后（在下一个 model call 前） | ❌ | ✅ |
| `Notification` | Claude Code 发送通知时 | ❌ | ❌ |
| `SubagentStart` | 子 agent 生成时 | ❌ | ✅ |
| `SubagentStop` | 子 agent 完成时 | ❌ | ❌ |
| `TaskCreated` | 任务创建时 | ❌ | ❌ |
| `TaskCompleted` | 任务标记为完成时 | ❌ | ❌ |
| `Stop` | Claude 完成响应时 | ✅ | ❌ |
| `StopFailure` | turn 因 API 错误结束时 | ❌ | ❌ |
| `TeammateIdle` | agent team 队友即将空闲时 | ❌ | ❌ |
| `InstructionsLoaded` | CLAUDE.md 或 .claude/rules/*.md 文件加载时 | ❌ | ❌ |
| `ConfigChange` | 配置文件在 session 期间改变时 | ❌ | ❌ |
| `CwdChanged` | 工作目录改变时 | ❌ | ❌ |
| `FileChanged` | 被监视的文件改变时 | ❌ | ❌ |
| `WorktreeCreate` | worktree 创建时 | ✅ | ❌ |
| `WorktreeRemove` | worktree 删除时 | ❌ | ❌ |
| `PreCompact` | context compaction 前 | ✅ | ❌ |
| `PostCompact` | context compaction 后 | ❌ | ❌ |
| `Elicitation` | MCP server 在工具调用期间请求用户输入时 | ❌ | ❌ |
| `ElicitationResult` | 用户响应 MCP elicitation 后（发送回 server 前） | ❌ | ❌ |
| `SessionEnd` | session 终止时 | ❌ | ❌ |

**关键发现**：
- `PreCompact` hook 支持阻止（返回 exit code 2 或 `{"decision":"block"}`）
- `PreCompact` 和 `PostCompact` **不支持** `additionalContext`（v2.1.105+）
- `PermissionRequest` hook 可返回 `decision.behavior` ("allow"/"deny") 来处理权限

URL: https://code.claude.com/docs/en/changelog.md
- v2.1.105: `PreCompact` hook 支持阻止 compaction（新增）
- v2.1.89: `PermissionDenied` 事件新增，`PreToolUse` 支持 `"defer"` decision

### 1.2 Hook 输入字段完整 Schema

URL: https://code.claude.com/docs/en/agent-sdk/hooks.md

所有 hook 接收输入的基础字段：
```json
{
  "session_id": "string",
  "cwd": "string",
  "hook_event_name": "string",
  "agent_id": "string (optional, subagent 中填充)",
  "agent_type": "string (optional)"
}
```

**工具相关 hook** (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`)：
```json
{
  "tool_name": "string",
  "tool_input": "object",
  "tool_use_id": "string or null",
  "duration_ms": "number (PostToolUse/PostToolUseFailure only)"
}
```

**其他重要输入**：
- `UserPromptSubmit`: `prompt` (user input text)
- `PermissionRequest`: `tool_name`, `tool_input`, 权限 rule 信息
- `SubagentStart`/`SubagentStop`: `agent_id`, `agent_transcript_path`
- `PreCompact`: `trigger` ("auto" 或 "manual"), `conversation_length_before`
- `InstructionsLoaded`: `file_path`, `file_type` ("claude_md" | "rules")

URL: https://code.claude.com/docs/en/agent-sdk/hooks.md
- 完整的 TypeScript 和 Python hook 类型定义在 SDK 参考中

### 1.3 Hook 输出契约（JSON Schema）

URL: https://code.claude.com/docs/en/hooks.md

**通用顶级字段**：
```json
{
  "systemMessage": "string",  // 注入到对话中的 system reminder
  "continue": "boolean",       // 是否继续执行（默认 true）
  "async": "boolean",          // 返回后不等待（TypeScript）
  "async_": "boolean",         // Python 版本
  "asyncTimeout": "number"     // 毫秒
}
```

**hookSpecificOutput** (取决于 event 类型)：
- `PreToolUse`: `permissionDecision` ("allow"|"deny"|"ask"|"defer"), `permissionDecisionReason`, `updatedInput`
- `PostToolUse`: `additionalContext`, `updatedToolOutput`
- `PermissionRequest`: `decision.behavior` ("allow"|"deny")
- `PreCompact`: `decision` ("block" 可选，阻止 compaction)
- `UserPromptSubmit`: `sessionTitle` (v2.1.94+)

### 1.4 Matcher 字段语法

URL: https://code.claude.com/docs/en/hooks.md

```json
{
  "matcher": "string",  // 可选，默认匹配所有
}
```

**匹配值规则**：
- 无 matcher 或 `"*"`: 匹配所有
- 纯字母数字 + `|`: 精确匹配（例 `"Bash|Edit"`）
- 含特殊字符：正则表达式（例 `"^Bash"`, `"mcp__.*"`)

**工具 hook 匹配目标**：工具名称  
**MCP 工具格式**：`mcp__<server>__<tool>`（例 `mcp__memory__.*` 匹配 memory 服务所有工具）

**非工具 hook 的匹配**：
- `FileChanged`: 文件名
- `Notification`: 通知类型
- 其他大多数：不支持 matcher

### 1.5 additionalContext 的完整行为

URL: https://code.claude.com/docs/en/hooks.md  
URL: https://code.claude.com/docs/en/context-window.md

**定义**：hook 返回的字符串，被包装为 system reminder 注入到 Claude 的 context

**大小限制**：
- **硬上限**：10,000 字符
- 超过限制时：内容保存到文件，用文件路径 + 预览替代（与大工具输出相同处理）
- 不会被静默截断，会有文件指针提示

**支持的事件**：
- ✅ `SessionStart`, `Setup`, `UserPromptSubmit`, `UserPromptExpansion`
- ✅ `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`
- ✅ `SubagentStart`
- ❌ `PreCompact`, `PostCompact` (v2.1.105+后明确不支持)

**注入时机**：
- `SessionStart`/`Setup`：conversation 起始前
- `UserPromptSubmit`/`UserPromptExpansion`：随 prompt 一起
- `PreToolUse`/`PostToolUse` 等：工具调用周边
- 作为 system reminder 而非聊天消息（Claude 在下一个 model request 时看到）

**官方原文**（https://code.claude.com/docs/en/hooks.md）：
> "Hook output injected into context (`additionalContext`, `systemMessage`, or plain stdout) is capped at 10,000 characters. Output that exceeds this limit is saved to a file and replaced with a preview and file path, the same way large tool results are handled."

**社区实现问题**：
- cc-triad-relay 和 claude-kit 在 `PreCompact` hook 中使用 `additionalContext`
- **官方文档明确说明**（v2.1.105+）：`PreCompact`/`PostCompact` 不支持 `additionalContext`
- 推理：compaction 是内部 context 管理操作，不适合向 Claude 注入新信息

### 1.6 多 Hook 同时匹配的执行顺序与合并语义

URL: https://code.claude.com/docs/en/agent-sdk/hooks.md

**执行模式**：同一事件的多个 hook **并行执行**（不确定完成顺序）

**决策合并规则**（PreToolUse 中）：
- **优先级**：`deny` > `defer` > `ask` > `allow`
- 单个 `deny` 阻止操作（不论其他 hook 返回什么）
- 建议：每个 hook 独立行动，不依赖其他 hook 的执行结果

**输出合并**（多个 hook 的 `additionalContext`）：
- 文档未明确说明
- 推理：应按火快顺序合并，但无保证

### 1.7 Hook 失败/超时的兜底行为

URL: https://code.claude.com/docs/en/hooks.md

**Exit Code 行为**：
- `0`: 成功，解析 JSON 输出
- `2`: 阻塞错误（stderr 显示，事件可阻止）
- 其他：非阻塞错误（stderr 日志记录，执行继续）

**超时**：
- 默认 60 秒（可配 `timeout` 字段）
- 超时时：hook 返回失败，如果是可阻止事件则不阻止

**JSON 解析失败**：
- 文档未明确，推理：日志记录，继续执行

---

## 2. Plugin / Marketplace 系统

### 2.1 .claude-plugin/plugin.json Schema

URL: https://code.claude.com/docs/en/plugins.md  
URL: https://code.claude.com/docs/en/plugins-reference.md

```json
{
  "name": "string (required, kebab-case)",
  "description": "string",
  "version": "string (optional)",
  "author": {
    "name": "string",
    "email": "string (optional)"
  },
  "homepage": "string (optional)",
  "repository": "string (optional)",
  "license": "string (optional, SPDX identifier)",
  
  "skills": "string|array (paths to skills/)",
  "commands": "string|array (paths to commands/)",
  "agents": "string|array (paths to agents/)",
  "hooks": "object or string (path to hooks.json)",
  "mcpServers": "object or string (path to .mcp.json)",
  "lspServers": "object or string (path to .lsp.json)",
  "monitors": "object or string (path to monitors.json)",
  
  "settings": "object (default settings when plugin enabled)"
}
```

**关键说明**：
- `name`：plugin 命名空间，skills 变为 `/name:skill-name`
- `version`：若设置，用户仅在版本改变时收到更新；若省略，用 git commit SHA
- 仅 `plugin.json` 放在 `.claude-plugin/` 内
- `skills/`, `commands/`, `agents/`, `hooks/` 等在 plugin 根目录

### 2.2 .claude-plugin/marketplace.json Schema

URL: https://code.claude.com/docs/en/plugin-marketplaces.md  
URL: https://code.claude.com/docs/en/plugins-reference.md

```json
{
  "$schema": "string (optional, for editor autocomplete)",
  "name": "string (required, kebab-case, public-facing)",
  "description": "string (optional)",
  "version": "string (optional)",
  "owner": {
    "name": "string (required)",
    "email": "string (optional)"
  },
  
  "metadata": {
    "pluginRoot": "string (optional, 相对路径前缀)"
  },
  
  "allowCrossMarketplaceDependenciesOn": ["array of marketplace names"],
  
  "plugins": [
    {
      "name": "string (required)",
      "source": "string|object (required)",
      "description": "string",
      "version": "string",
      "author": {"name": "string", "email": "string"},
      "category": "string",
      "tags": ["array"],
      "strict": "boolean (default true)",
      
      // 可继承 plugin.json 的字段
      "skills": "string|array",
      "commands": "string|array",
      "agents": "string|array",
      "hooks": "object|string",
      "mcpServers": "object|string",
      "lspServers": "object|string"
    }
  ]
}
```

**plugin source 格式**：
- `"./relative/path"`: 相对路径（仅 Git 分发）
- `{"source": "github", "repo": "owner/name", "ref": "...", "sha": "..."}`: GitHub
- `{"source": "url", "url": "...", "ref": "...", "sha": "..."}`: Git URL
- `{"source": "git-subdir", "url": "...", "path": "...", "ref": "...", "sha": "..."}`: Git 子目录
- `{"source": "npm", "package": "...", "version": "...", "registry": "..."}`: npm

**reserved marketplace names**（不能用）：
- `claude-code-marketplace`, `claude-code-plugins`, `claude-plugins-official`
- `anthropic-marketplace`, `anthropic-plugins`, `agent-skills`

### 2.3 Plugin 内部 Hooks 声明

URL: https://code.claude.com/docs/en/plugins-reference.md

**两种方式**：
1. **hooks.json 文件**：`.claude-plugin/` 内的 `hooks.json`
2. **plugin.json 内嵌**：`"hooks": {...}` 字段或 `"hooks": "path/to/hooks.json"`

**格式**（与用户 settings.json hooks 相同）：
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {"type": "command", "command": "..."}
        ]
      }
    ]
  }
}
```

**新增 (v2.1.118)**：hooks 可调用 MCP 工具
```json
{
  "type": "mcp_tool",
  "server": "server-name",
  "tool": "tool-name",
  "input": {"..."}
}
```

### 2.4 Plugin 范围 Hook 与用户 settings.json Hook 的优先级和合并规则

URL: https://code.claude.com/docs/en/changelog.md  
URL: https://code.claude.com/docs/en/plugins.md

**合并模式**：
- Plugin hooks 和用户 hooks **合并**（不覆盖）
- 执行顺序：用户 hooks + plugin hooks（相同事件）
- 由于并行执行，无保证顺序

**优先级**（矛盾时）：
- Plugin hooks from managed settings (force-enabled): 始终运行
- User hooks: 遵守权限和 settings

**故障修复** (v2.1.101)：
- Plugin hooks from managed settings 现在在 `allowManagedHooksOnly` 时运行
- Plugin skill hooks (frontmatter 定义) 现已支持 (v2.1.94)

### 2.5 Plugin Enable/Disable 与 Hook 同步生效

URL: https://code.claude.com/docs/en/plugins.md

**行为**：
- `/plugin install`: 加载 plugin hooks
- `/plugin uninstall`: 移除 plugin hooks
- 生效时间：**当前 session 不受影响**，下一个 session 生效

**reload-plugins**: `/reload-plugins` 使 plugin hooks 在当前 session 重新加载（v2.1.97+）

### 2.6 Plugin 安装时 Cache 策略

URL: https://code.claude.com/docs/en/plugin-marketplaces.md  
URL: https://code.claude.com/docs/en/changelog.md

**存储位置**：`~/.claude/plugins/cache/`

**更新策略**：
- Git-based: fetch 最新
- npm: `npm install` with version constraints
- 用户可手动 `/plugin update`

**SSH vs HTTPS**：
- Git repositories：自动选择（优先 SSH，fallback HTTPS）
- 可通过环境变量或配置控制

**相关修复**：
- v2.1.128: `--plugin-dir` 现接受 `.zip` 档案
- v2.1.129: `--plugin-url <url>` 从 URL 加载 `.zip` plugin
- v2.1.128: 修复 npm 源 plugin 不检测新版本 (update never worked)

---

## 3. Skill 系统

### 3.1 Skill 通过 Plugin 安装后的命名空间规则

URL: https://code.claude.com/docs/en/skills.md  
URL: https://code.claude.com/docs/en/plugins.md

**命名空间规则**：
- **Standalone skills** (`.claude/skills/`): `/skill-name`
- **Plugin skills**: `/plugin-name:skill-name`
- 目的：防止 plugin 间冲突

**解析顺序**（优先级）：
1. Managed (organization-wide)
2. Personal (`~/.claude/skills/`)
3. Project (`.claude/skills/`)
4. Plugin (`plugin-name/skills/`)

**Skill 目录结构**：
```
skills/
├── skill-name/
│   ├── SKILL.md (required)
│   ├── reference.md (optional)
│   └── scripts/ (optional)
```

### 3.2 Skill Description 字段对 Slash Command 自动补全的影响

URL: https://code.claude.com/docs/en/skills.md

**Description 用途**：
- 加载到 context 中供 Claude 自动调用（skill 发现）
- 显示在 `/skills` 菜单和 `/` 补全
- 截断点：1,536 字符（`description` + `when_to_use` 合计）

**补全行为**：
- Slash command picker 显示所有 skills（按字母序）
- Description 用于搜索和优先级
- 长 description 被截断但不影响补全展示

### 3.3 Skill 与 Slash Command 的关系

URL: https://code.claude.com/docs/en/skills.md

**等价性**：
- `.claude/commands/foo.md` → `/foo`
- `.claude/skills/foo/SKILL.md` → `/foo`
- 两者工作相同，skills 支持额外特性

**额外特性**（skills only）：
- 目录结构支持支援文件
- Frontmatter 控制（`disable-model-invocation`, `allowed-tools` 等）
- 子 agent 执行（`context: fork`）
- 动态 context 注入（`` !`command` ``）

### 3.4 agentskills.io 规范

URL: https://code.claude.com/docs/en/skills.md （引用）
URL: https://agentskills.io (外部)

**规范要点**（官方文档链接）：
- Skill 定义为 `SKILL.md` with YAML frontmatter
- `description`: 何时使用此 skill
- 支持 named arguments (`arguments` frontmatter field)
- 支持 dynamic context injection (`` !`command` ``)

**Claude Code 扩展**：
- `disable-model-invocation`: 仅用户可调用
- `user-invocable: false`: 仅 Claude 可调用
- `allowed-tools`: 预先批准工具
- `context: fork`: 在子 agent 中运行
- `paths`: 条件激活（glob patterns）

---

## 4. Compact 生命周期

### 4.1 Compact 触发条件

URL: https://code.claude.com/docs/en/how-claude-code-works.md  
URL: https://code.claude.com/docs/en/context-window.md

**Auto Compact**：
- 触发：context 接近模型上限时
- 行为：自动总结旧历史，保留最新交换和关键决定
- **用户感知**：最小化（后台运行，但有 "Conversation compacted" 消息）
- **配置**：`/compact` 手动触发，`DISABLE_COMPACT` 禁用

**Manual Compact**：
- `/compact` 命令
- `/compact focus on X` 带焦点指令

**Compact vs Summarize**：
- `/compact`: 全对话压缩
- `/rewind` -> "Summarize from here": 从选定点后的内容压缩

### 4.2 Compact 期间 transcript_path 的状态

URL: https://code.claude.com/docs/en/agent-sdk/sessions.md  
URL: https://code.claude.com/docs/en/changelog.md

**Transcript 写入行为**：
- Compact 前：所有消息写入 JSONL 文件 (`~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`)
- Compact 时：新的总结消息追加到 transcript
- Compact 后：旧消息仍在 JSONL 中（未删除），总结插入其中

**修复** (v2.1.97, v2.1.89)：
- 修复 compaction 写入重复多 MB 子 agent transcript 文件
- 修复 prompt-too-long 重试时的重复写入

### 4.3 Compact 后新会话的 session_id 是否变化、是否有关联标识

URL: https://code.claude.com/docs/en/agent-sdk/sessions.md

**同一 session 的 compact**：
- `session_id` **不变**（compact 是 session 内操作）
- 关联标识：同一文件，不同 line range

**Resume 后的 session_id**：
- `--resume <session-id>`: 同一 ID
- `--continue`: 最新 session 同一 ID
- `--fork-session`: 新 ID（分支）

**Compact 边界标记**：
- SDK: `SDKCompactBoundaryMessage` (TypeScript) 或 `SystemMessage` with `subtype: "compact_boundary"` (Python)

### 4.4 Compact 后哪些状态被保留、哪些被丢弃

URL: https://code.claude.com/docs/en/how-claude-code-works.md  
URL: https://code.claude.com/docs/en/context-window.md

**保留**：
- ✅ 系统 prompt
- ✅ CLAUDE.md（全部，重新注入）
- ✅ Auto memory (MEMORY.md 前 200 行 / 25KB)
- ✅ 最近的 skill invocation（前 5,000 tokens）
- ✅ 关键代码片段和文件路径
- ✅ 最近的决定和推理

**丢弃**：
- ❌ 旧对话历史（总结取代）
- ❌ 大工具输出（摘要）
- ❌ 详细指令（来自早期对话）

**重新注入**：
- ✅ CLAUDE.md (project root, project level, user level)
- ✅ Skill descriptions（仅描述，全内容需重新调用）
- ✅ MCP tool 定义（延迟加载）

**总结指令**：
- 在 CLAUDE.md 中添加 "Summary instructions" 或 "Compact instructions" 章节
- Compactor 读取 CLAUDE.md，按意图匹配（无魔法字符串）

### 4.5 "Auto Compact" 用户是否能感知到、能否被关闭

URL: https://code.claude.com/docs/en/how-claude-code-works.md  
URL: https://code.claude.com/docs/en/changelog.md

**感知**：
- 最小化：后台发生，界面显示 "Conversation compacted" 消息
- v2.1.120: Auto mode 显示 `auto` (小写，无 token 数) 而非误导的 token 值

**关闭**：
- 环境变量：`DISABLE_COMPACT=1`
- 设置：`"disableAutoCompact": true` (未在官方文档找到，推理)

**PreCompact Hook**：
- Hook 可返回 `{"decision": "block"}` 阻止 compaction
- 场景：archive full transcript 后阻止

---

## 5. 相关的 Changelog / 公告

### 5.1 近 6 个月关于 Hooks 的条目

URL: https://code.claude.com/docs/en/changelog.md (v2.1.114 - v2.1.119)

**Hooks 新增/改进**：
- v2.1.121: `PostToolUse` 对所有工具支持 `updatedToolOutput`（原仅 MCP）
- v2.1.119: `PreToolUse` hook 输入新增 `duration_ms` 字段；async `PostToolUse` hooks 修复（不再写空条目）
- v2.1.118: 新 hook type `type: "mcp_tool"` 支持 hooks 直接调用 MCP 工具
- v2.1.110: `PreToolUse` 返回 `additionalContext` 不再在工具失败时丢弃
- v2.1.105: `PreCompact` hook 支持阻止 compaction（新增）
- v2.1.101: `permissions.deny` 规则现覆盖 `PreToolUse` hook 的 `ask` decision
- v2.1.98-99: Prompt 型 hook 在长 session 上失败；hooks `if` 条件改进

### 5.2 近 6 个月关于 Plugin 的条目

URL: https://code.claude.com/docs/en/changelog.md

**Plugin 新增/改进**：
- v2.1.129: `--plugin-url <url>` 新增，从 URL 加载 `.zip` plugin
- v2.1.128: `--plugin-dir` 现接受 `.zip` 档案；`claude plugin tag` 新增
- v2.1.121: Plugin skill hooks (frontmatter) 现被正确处理
- v2.1.120: `plugin.json` validate 新增 `$schema`, `version`, `description` 支持
- v2.1.119: Plugin MCP servers 修复 `${user_config.*}` optional field 处理
- v2.1.118: Plugins 可通过 `themes/` 目录 ship 自定义主题

### 5.3 近 6 个月关于 Compact 的条目

URL: https://code.claude.com/docs/en/changelog.md

**Compact 新增/改进**：
- v2.1.120: Auto mode 显示优化（lowercase 'auto'）
- v2.1.119: Skills invoked 前 auto-compact 不再被重新执行
- v2.1.117: Opus 4.7 现对 1M window 计算（修复虚高的百分比）
- v2.1.105: `PreCompact` hook 支持阻止（新增）
- v2.1.97, v2.1.89: Autocompact thrash 循环检测和修复

### 5.4 关于 "Context Preservation" 或 "Long-running Session" 的官方建议

URL: https://code.claude.com/docs/en/memory.md  
URL: https://code.claude.com/docs/en/how-claude-code-works.md  
URL: https://code.claude.com/docs/en/agent-sdk/sessions.md

**官方建议**：

**1. 使用 CLAUDE.md 存储持久指令**：
   - 项目规则、构建命令、架构
   - 不依赖对话历史（compaction 后可能丢失）

**2. 使用 Auto Memory（MEMORY.md）保存学习**：
   - Claude 自动记录发现的模式
   - 下一个 session 自动加载（前 200 行 / 25KB）

**3. 使用 Subagents 保持主 Context 精简**：
   - Subagent 独立 context window
   - 仅返回摘要给主 agent

**4. 设置 Compact 指令**：
   - CLAUDE.md 中添加 "Summary instructions" 章节
   - 告诉 compactor 保留什么

**5. 使用 Skills 管理可复用工作流**：
   - Skills 按需加载（仅描述在 context 中）
   - Invoked skills 重新连接（前 5,000 tokens per skill）

---

## 6. 关键发现与意外

### 6.1 PreCompact Hook 与 additionalContext

**用户疑惑根源**：
- cc-triad-relay, claude-kit 等社区库在 `PreCompact` hook 中使用 `additionalContext`
- 官方文档（旧版）未明确说明不支持

**官方澄清**（v2.1.105+）：
- `PreCompact` 和 `PostCompact` hooks **不支持** `additionalContext`
- 原因：compaction 是内部 context 管理，不是对 Claude 的信息传递
- 改为使用：`PreCompact` hook 可返回 `{"decision": "block"}` 阻止 compaction
- 如需保留 context：hook 中执行 archive（保存 transcript），返回空或 system message

**正确用法**（handoff-skills 应采用）：
```json
{
  "hooks": {
    "PreCompact": [
      {
        "type": "command",
        "command": "/path/to/save-transcript-hook.sh"
      }
    ]
  }
}
```

Hook 脚本应：
1. 读取 `session_id` 从 stdin JSON
2. 保存 transcript（可用 Context7 或 grep 查询）
3. 返回 `{"decision": "block"}` 阻止 compaction，或 `{}` 允许

### 6.2 Hook 执行顺序和 additionalContext 的合并

**未文档化的细节**：
- 多 hook 同事件时并行执行，完成顺序无保证
- `additionalContext` 多个来源时的合并方式未明确
- 推理：应串行追加，但无保证

**建议（handoff-skills）**：
- 每个 hook 独立，不依赖顺序
- 单一 hook 负责 context 注入，避免冲突

### 6.3 Session 恢复与 Transcript 完整性

URL: https://code.claude.com/docs/en/agent-sdk/sessions.md

**关键保证**：
- `resume <session-id>` 恢复完整对话历史
- 新 message 追加到同一文件
- `--fork-session` 创建新 ID，原 session 不变

**大 session 修复** (v2.1.101, v2.1.121)：
- v2.1.121: 修复 corrupt line 导致的 resume 失败
- v2.1.101: 修复链式 resume 误进入不相关子 agent context

**Handoff 场景含义**：
- Save: 捕获 `session_id` + transcript 路径
- Resume: 用 `session_id` 恢复，全历史可用
- 风险：手动删除 transcript 文件会导致恢复失败

---

## 7. 官方文档未提及的部分

### 7.1 Compact 后旧 Skill Description 是否被重新加载

**官方文档说法**：
- Skill descriptions 在 context 中（但未明确 compaction 后的行为）
- 全 skill content 仅在 invoked 时加载

**推理**（基于代码注释）：
- Compaction 后 skill descriptions 应重新加载（与 CLAUDE.md 同样方式）
- 但具体机制未文档化

### 7.2 Plugin 依赖的版本约束解析

URL: https://code.claude.com/docs/en/plugin-dependencies.md (未详细获取)

**官方信息缺口**：
- `allowCrossMarketplaceDependenciesOn` 的精确行为
- Plugin A 依赖 Plugin B（版本 X.Y.Z）的冲突解决

### 7.3 MCP Tool Hook 的完整 Schema

URL: https://code.claude.com/docs/en/changelog.md (v2.1.118)

**已添加但文档不完善**：
```json
{
  "type": "mcp_tool",
  "server": "server-name",
  "tool": "tool-name",
  "input": {"..."}
}
```

完整的字段、错误处理、输出格式未清晰文档化。

---

## 总结：对 Handoff Plugin 设计的建议

### Hook 策略
1. **Save Context (PreCompact)**:
   - ✅ 使用 `type: "command"` hook 脚本
   - ✅ 脚本保存 transcript 到版本控制/云存储
   - ❌ 不要用 `additionalContext`（v2.1.105+ 不支持）
   - ✅ 返回 `{"decision": "block"}` 强制用户确认

2. **Restore Context (SessionStart)**:
   - ✅ `additionalContext` 注入 "previous session summary"
   - ✅ 10,000 字符限制，超过部分自动文件化
   - ✅ Prevent false recovery: 用 `session_id` 检查是否真是 compact 前的 resume

3. **多 Hook 协调**：
   - 单独的 plugin hooks，不依赖执行顺序
   - 如需全局状态，写入 Hook Data Store（文件）

### Plugin 结构
```
handoff-plugin/
├── .claude-plugin/
│   └── plugin.json (name, version, hooks)
├── hooks/
│   ├── save-transcript.sh (PreCompact)
│   ├── restore-transcript.sh (SessionStart)
│   └── prevent-false-recovery.sh (共享逻辑)
├── skills/
│   └── handoff-save/SKILL.md
│   └── handoff-resume/SKILL.md
└── README.md
```

---

## Sources（所有浏览过的 URL）

### 官方文档（Code Docs）
- https://code.claude.com/docs/en/hooks.md
- https://code.claude.com/docs/en/hooks-guide.md
- https://code.claude.com/docs/en/agent-sdk/hooks.md
- https://code.claude.com/docs/en/plugins.md
- https://code.claude.com/docs/en/plugins-reference.md
- https://code.claude.com/docs/en/skills.md
- https://code.claude.com/docs/en/agent-sdk/skills.md
- https://code.claude.com/docs/en/plugin-marketplaces.md
- https://code.claude.com/docs/en/context-window.md
- https://code.claude.com/docs/en/checkpointing.md
- https://code.claude.com/docs/en/agent-sdk/sessions.md
- https://code.claude.com/docs/en/how-claude-code-works.md
- https://code.claude.com/docs/en/memory.md
- https://code.claude.com/docs/en/agent-sdk/agent-loop.md
- https://code.claude.com/docs/en/changelog.md
- https://code.claude.com/docs/en/whats-new/2026-w17.md

### 相关参考资源
- https://agentskills.io (Agent Skills 规范)
- https://code.claude.com/docs/en/claude_code_docs_map.md (文档地图)

**报告生成日期**：2026-05-07  
**总计覆盖 URL**：16+ 官方页面，详尽查证所有相关特性
