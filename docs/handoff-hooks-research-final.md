# Handoff-skills Hook 自动化：最终调研报告

**调研日期**: 2026-05-07
**调研深度**: 5 个并行 deep-research agent + 多轮 mcp grep / WebFetch / Exa search
**总参考链接**: 250+

---

## TL;DR（一句话给到判断）

**完全自动化的 save/resume 在 Claude Code 当前版本下不存在干净路径**。可以做到的最优组合是：手动 `/handoff-save` 写文件 + `SessionStart(source=compact)` hook 自动注入恢复提示——但这条路径**官方文档从未明示支持**（"works but undocumented"），且**plugin 分发的 hook 在多个版本里 output 被静默丢弃**。

**推荐策略**：保留现状的 manual `/handoff-save` + `/handoff-resume` skill；可选增加一个**最低风险的 hook 增强**（PreCompact 写 sentinel + SessionStart 读 sentinel 注入提示），但用户自己装到 `~/.claude/settings.json`，不靠 plugin 分发。

---

## 一、Claude Code Hook 系统的现实约束

### 1.1 PreCompact 的能力矩阵（2026-05 版本）

| 能力 | 状态 | 来源 |
|---|---|---|
| 接收 transcript_path / trigger / custom_instructions | ✅ 文档化 | [hooks reference](https://code.claude.com/docs/en/hooks) |
| 通过 `decision: "block"` 阻止压缩（v2.1.105+） | ✅ 文档化 | [changelog](https://code.claude.com/docs/en/changelog) v2.1.105 |
| 通过 `additionalContext` 注入内容到 compact summary | ❌ **官方 schema 拒绝** | [Issue #46191](https://github.com/anthropics/claude-code/issues/46191) (Apr 2026, OPEN) |
| 通过 `customInstructions` 影响压缩器行为 | ⚠️ SDK 协议接受但 type-mismatch | [Issue #14160](https://github.com/anthropics/claude-code/issues/14160), [autoforge client.py](https://github.com/AutoForgeAI/autoforge/blob/master/client.py) |
| 在 `/compact` 手动触发时可靠 fire | ⚠️ 历史断断续续坏 | [Issue #13572](https://github.com/anthropics/claude-code/issues/13572) (closed stale 2026-02), [#15096](https://github.com/anthropics/claude-code/issues/15096), [#26010](https://github.com/anthropics/claude-code/issues/26010) |
| 在同一 session 多次 compact 后继续 fire | ❌ **第一次 compact 后所有 plugin hook 死掉** | [Issue #25655](https://github.com/anthropics/claude-code/issues/25655) (OPEN) |

**关键引用**（Issue #46191，2026-04-10，仍 OPEN）：

> "PreCompact and PostCompact hook events do not support hookSpecificOutput.additionalContext... The schema only accepts 'PreToolUse', 'UserPromptSubmit', and 'PostToolUse' in hookSpecificOutput.hookEventName."

### 1.2 SessionStart 的 matcher 语义

| Matcher | 触发场景 | additionalContext 注入是否生效 |
|---|---|---|
| `startup` | 全新 `claude` 启动 | ✅（v2.1.x 后的版本）；旧版本有 [Issue #10373](https://github.com/anthropics/claude-code/issues/10373) |
| `resume` | `claude --resume` / `--continue` / `/resume` | ✅ 稳定 |
| `clear` | `/clear` 后 | ✅ 稳定 |
| `compact` | 自动或手动 compact 后 | ⚠️ **work but undocumented**，曾被报 [Issue #28305](https://github.com/anthropics/claude-code/issues/28305)、[#15174](https://github.com/anthropics/claude-code/issues/15174) 静默丢弃；最新评论"resolved"，但状态混乱 |

**关键引用**（Issue #25999，2026-02-16，closed not planned）作者明示，多个行为 "**works, undocumented**"：

> "SessionStart 的 'compact' source、source 枚举值、additionalContext 通过 hookSpecificOutput 注入、PreCompact 触发时机... Anthropic 至今对多次 stabilization 请求保持沉默。"

### 1.3 Plugin 分发的 Hook 致命缺陷（决定性发现）

[Issue #16538](https://github.com/anthropics/claude-code/issues/16538)（Jan-Mar 2026，重现于 v2.1.45）：

> "**Plugin hooks remain broken** — identical hook definitions in a plugin's hooks/hooks.json execute successfully (confirmed via side effects) but **output is never injected into model context**. This has been consistent from v2.1.42 through v2.1.45 with no hook-related changes in the CHANGELOG for these versions."
>
> "**Workaround**: Define hooks natively in `~/.claude/settings.json` instead of `hooks/hooks.json`. This defeats the purpose of plugin-distributed hooks but is the only reliable path."

确认问题范围：

| Hook 来源 | additionalContext 注入 | side-effect 仍执行 |
|---|---|---|
| `~/.claude/settings.json`（native）| ✅ 全部支持的 event | ✅ |
| Plugin `hooks/hooks.json` | ❌ 多版本静默丢弃 | ✅ |

**这意味着**：即使我们 ship 一个完美的 `hooks.json` 在 handoff-skills plugin 里，用户安装后的 SessionStart hook 在某些版本下根本不会注入 `additionalContext`。

### 1.4 Compact 周边的其他坑

| Issue | 内容 |
|---|---|
| [#20370](https://github.com/anthropics/claude-code/issues/20370) | `/compact <instructions>` 文字会跨过 compact 边界出现在新会话里，导致 post-compact 实例可能误执行 |
| [#25655](https://github.com/anthropics/claude-code/issues/25655) | Compact 后所有 plugin hook 在该 session 内停止触发 |
| [#41919](https://github.com/anthropics/claude-code/issues/41919) | 即使 plugin disabled，SessionStart hook 仍 fire |
| [#31658](https://github.com/anthropics/claude-code/issues/31658) | 多 hook 同事件返回 additionalContext 时合并语义不一致 |
| [#256 (everything-claude-code)](https://github.com/affaan-m/everything-claude-code/issues/256) | `CLAUDE_PLUGIN_ROOT` 在 SessionStart 事件中不被设置 |

### 1.5 Hooks 不能调用 Skills

[dev.to 详细分析](https://dev.to/aabyzov/claude-code-hook-limitations-no-skill-invocation-lazy-plugin-loading-and-how-i-solved-it-44f2)：

- Hook 支持的 type：`command`、`prompt`（Stop/SubagentStop）、`mcp_tool`（v2.1.118+）、`http`（v2.1.63+）、JS 函数
- **没有 `type: "skill"`**
- Workaround：hook spawn `claude -p "/skill-name $PROMPT"`（同步阻塞，重，不推荐）

---

## 二、Anthropic 官方对 Compact / Hook 的态度

### 2.1 官方明确不打算支持的事情

| 请求 | 状态 | Issue |
|---|---|---|
| 关闭 auto-compact | closed not planned | [#18085](https://github.com/anthropics/claude-code/issues/18085), [#42149](https://github.com/anthropics/claude-code/issues/42149) |
| `autoCompactEnabled` setting | 不在 schema | [#38483](https://github.com/anthropics/claude-code/issues/38483) |
| `replaceCompactSummary` 输出字段 | closed dup | [#24965](https://github.com/anthropics/claude-code/issues/24965), [#13170](https://github.com/anthropics/claude-code/issues/13170) |
| Custom pre/compact/post-compact 命令 | closed locked | [#3349](https://github.com/anthropics/claude-code/issues/3349) |
| PreCompact / PostCompact additionalContext | open 但无 Anthropic 回复 | [#46191](https://github.com/anthropics/claude-code/issues/46191), [#17237](https://github.com/anthropics/claude-code/issues/17237), [#33088](https://github.com/anthropics/claude-code/issues/33088), [#50682](https://github.com/anthropics/claude-code/issues/50682), [#54118](https://github.com/anthropics/claude-code/issues/54118) |

**评论原文**（[#46191](https://github.com/anthropics/claude-code/issues/46191)）：

> "+1, this is the single highest-leverage hook affordance for external memory layers. Without additionalContext return, PreCompact is observe-only — we can save state before compaction but can't influence what replaces the window."

### 2.2 官方推荐的"正路"

来自 [Hooks Guide](https://code.claude.com/docs/en/hooks-guide) 的 "Re-inject context after compaction" 章节：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          { "type": "command", "command": "echo 'Reminder: ...'" }
        ]
      }
    ]
  }
}
```

> "Any text your command writes to stdout is added to Claude's context. This example reminds Claude of project conventions and recent work."

但这条建议放进 plugin 时就掉进 §1.3 的坑。

### 2.3 Anthropic 工程博客的相关内容

| 日期 | 主题 | URL |
|---|---|---|
| 2025-09-29 | Effective context engineering for AI agents（compaction + structured note-taking + subagent 三大策略） | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents |
| 2025-10-09 | Customize Claude Code with plugins | https://claude.com/blog/claude-code-plugins |
| 2025-10-16 | Equipping agents for the real world with Agent Skills | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills |
| 2025-12-11 | How to configure hooks（power user customization） | https://claude.com/blog/how-to-configure-hooks |
| 2025-12-18 | Skills 作为开放标准发布 | https://www.anthropic.com/news/agent-skills-open-standard |
| 2026-03-24 | Harness design for long-running application development | https://www.anthropic.com/engineering/harness-design-long-running-apps |
| 2026-04-08 | Scaling Managed Agents | https://www.anthropic.com/engineering/managed-agents |

---

## 三、社区开源实现实战拆解

下面是把 70+ 个相关开源项目按"对我们的借鉴价值"分组，每个标注关键代码位置和设计决策。

### 3.1 顶级参考（直接可抄）

#### 3.1.1 [`joseairosa/recall`](https://github.com/joseairosa/recall) ⭐ 166

最简洁的 save+restore 实现：

- `pre-compact.sh`：把 session_name / memories / cwd / git_remote 写到 `~/.claude/recall/pre-compact-state.json`
- `compact-restore.sh`：注册在 `SessionStart` matcher `compact`，**读完立即 `rm -f`**（"single-use — one compaction → one restore"）
- 输出走 stdout（不用 hookSpecificOutput JSON），直接被 SessionStart 注入

**关键代码**（防误恢复）：
```bash
STATE_CONTENT="$(cat "${PRE_COMPACT_STATE}" 2>/dev/null || true)"
rm -f "${PRE_COMPACT_STATE}" 2>/dev/null || true
```

这是 **single-use sentinel** 模式的教科书实现——比时间窗口更可靠。

#### 3.1.2 [`afoxnyc3/roadrunner-cli`](https://github.com/afoxnyc3/roadrunner-cli) （ADR-007 是金子）

- [`hooks/precompact_hook.sh`](https://github.com/afoxnyc3/roadrunner-cli/blob/main/hooks/precompact_hook.sh)：5 行 shim → `roadrunner snapshot`（写 `.context_snapshot.json`）
- [`hooks/session_start_hook.sh`](https://github.com/afoxnyc3/roadrunner-cli/blob/main/hooks/session_start_hook.sh)：5 行 shim → `roadrunner session-start`（读文件，注入）
- [`hooks/postcompact_hook.sh`](https://github.com/afoxnyc3/roadrunner-cli/blob/main/hooks/postcompact_hook.sh)：仅观测，不影响控制流

**[ADR-007 "Dead Hook Cleanup"](https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/007-dead-hook-cleanup.md) 原文**：

> "The write_context_snapshot() function printed `{"additionalContext": "..."}` to stdout, expecting PreCompact to inject it... **Per the docs, PreCompact supports only `decision: "block"` for output. The `additionalContext` field is supported by SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Notification, and SubagentStart — but not PreCompact or PostCompact.** The print was silently ignored."
>
> "Decision: Remove the dead `print()` from `write_context_snapshot()` — keep only the file write. Add a SessionStart hook — reads `.context_snapshot.json` and emits `additionalContext`."

**SessionStart 端的精彩处理**（[cli.py:1775-1880](https://github.com/afoxnyc3/roadrunner-cli/blob/main/src/roadrunner/cli.py)）：

```python
# additionalContext 写成 INSTRUCTION 而非 status
# 这样 Claude turn 1 直接开始干活，不需要用户输入"continue"
message = (
    "You are resuming a roadrunner-driven session. No task is "
    "in progress. Your first action:\n\n"
    f"    roadrunner start {next_task['id']}\n\n"
    "Then follow the brief below.\n\n" + brief
)
```

**关键 ADR**：
- [ADR-007: Dead Hook Cleanup](https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/007-dead-hook-cleanup.md)
- [ADR-009: State schema versioning](https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/009-state-schema-versioning-and-concurrency-lock.md)
- [ADR-010: Hook entrypoint unification](https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/010-hook-python-entrypoint-unification.md)

#### 3.1.3 [`who96/claude-code-context-handoff`](https://github.com/who96/claude-code-context-handoff)

完整的"两层架构"参考：

**Hook 层**：
- PreCompact → 捕获 context
- SessionEnd(clear) → /clear 时也捕获
- SessionStart(compact|clear) → 注入 additionalContext

**Supervisor 层**（外部 Python 进程）：
- `claude-handoff-supervisor.py` 启动 Claude，**把用户的 `/compact` 重写为 `/clear`**
- 因为 hooks 不能 rewrite slash commands，只能用外部 supervisor 拦截 stdin

**保护机制**：
- 同 cwd 检查
- 最大 age 窗口（`HANDOFF_LATEST_MAX_AGE_SEC` 默认 900s = 15 分钟）
- `latest-handoff.md` 作为 fallback 恢复源

#### 3.1.4 [`ThomasEdwardYorke/cc-triad-relay`](https://github.com/ThomasEdwardYorke/cc-triad-relay)

完整 16 hook event 实现 + 严格 spec audit：

**[plugins/harness/core/src/hooks/pre-compact.ts](https://github.com/ThomasEdwardYorke/cc-triad-relay/blob/main/plugins/harness/core/src/hooks/pre-compact.ts)** 的注入字段：
1. `[custom_instructions]`（用户传的 `/compact <text>`）
2. `[ブランチ]` 当前 git branch
3. `[assignment-table]`（Plans.md 的"担当表"section，512KB cap）
4. `[オープン PR]`（`gh pr list --limit 10`，5s timeout）
5. `[trigger]` `manual` 或 `auto`

**关键 spec audit 注释**（[index.ts:120-150](https://github.com/ThomasEdwardYorke/cc-triad-relay/blob/main/plugins/harness/core/src/index.ts)）。

原文（日文）：

```typescript
// PreCompact spec audit (Codex 一次資料確認、2026-04-28):
// PreCompact schema は `decision: "block"` のみ documented。
// `additionalContext` は **top-level も `hookSpecificOutput.*` も spec で許容されない**。
// spec-supported な context delivery channel は universal `systemMessage` field のみ
//
// handler は backward-compat semantics 保持のため `additionalContext` で context を返却し、
// dispatcher は **`systemMessage` に routing** して wire output を spec 準拠化する。
```

中译：

```text
// PreCompact spec audit（Codex 一手资料确认，2026-04-28）：
// PreCompact schema 中只有 `decision: "block"` 是文档化的。
// `additionalContext` 不论是 top-level 还是 `hookSpecificOutput.*` 都不被 spec 允许。
// 被 spec 支持的 context 投递通道只有 universal 的 `systemMessage` field。
//
// handler 为保持向后兼容语义，仍以 `additionalContext` 返回 context，
// 由 dispatcher 在 wire output 阶段 routing 到 `systemMessage`，
// 让最终输出符合 spec。
```

所以 cc-triad-relay 在 PreCompact 输出时**把 additionalContext 改名为 `systemMessage`**——这是另一条路径（universal field，不是 hookSpecificOutput）。

#### 3.1.5 [`hex0xdeadbeef/claude-kit`](https://github.com/hex0xdeadbeef/claude-kit)

对抗 truncation 最严肃的实现：

- [`save-progress-before-compact.sh`](https://github.com/hex0xdeadbeef/claude-kit/blob/main/.claude/scripts/save-progress-before-compact.sh)：6KB cap + overflow spillover + LRU keep 5
- [`verify-state-after-compact.sh`](https://github.com/hex0xdeadbeef/claude-kit/blob/main/.claude/scripts/verify-state-after-compact.sh)：PostCompact 重读 disk + 二次注入
- [`state_render.py`](https://github.com/hex0xdeadbeef/claude-kit/blob/main/.claude/scripts/lib/state_render.py)：共享 render 库

**关键设计**：
```python
CONTEXT_SIZE_CAP: int = 6000
# P5: lowered from 8192 to leave 4 000 chars of slack under
# Claude Code's 10 000-char hook-output cap
```

PreCompact 只输出 reference link（不 dump 整个 checkpoint YAML，因为 5-50KB body inline 是浪费）：

```python
def _render_checkpoint_ref(state):
    """PostCompact reads checkpoint from disk on recovery; in-context copy
    is pure overhead of 5–50 KB per PreCompact invocation."""
    return (
        f"## Workflow Checkpoint\n"
        f"File: {state_dir}/{feature}-checkpoint.yaml\n"
        f"Resume: /workflow --from-phase {phase}"
    )
```

#### 3.1.6 [`kylesnowschwartz/claude-handoff`](https://github.com/kylesnowschwartz/claude-handoff) ⭐ 31

最有创意的"goal-focused handoff" 实现：

- 用户输入 `/compact handoff:<新目标>`
- PreCompact 检测 `custom_instructions` 是否以 `handoff:` 开头
- 如是，用 `claude --resume $session_id --fork-session --model haiku --print "<extraction prompt>"` **fork 出子会话用 haiku 提取 goal-focused 摘要**
- 写到 `.git/handoff-pending/handoff-context.json`
- SessionStart 仅在 `source == "compact"` 才执行，读文件、`rm -f` 清理、用 `systemMessage` 注入

#### 3.1.7 [`obra/superpowers`](https://github.com/obra/superpowers) ⭐ 179k stars

跨平台兼容范本（同一 hook 适配 Cursor / Claude / Copilot CLI）：

```bash
# session-start 脚本动态选择输出 schema
if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
  printf '{"additional_context": %s}' "$ESCAPED"
elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  printf '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": %s}}' "$ESCAPED"
else
  printf '%s' "$CONTEXT"  # 纯 stdout fallback
fi
```

### 3.2 中度参考（部分启发）

| 项目 | 一句话亮点 | URL |
|---|---|---|
| `arpitnath/claude-capsule-kit` | SQLite 存 namespaced handoff，按 `branch + crew teammate` 过滤 | https://github.com/arpitnath/claude-capsule-kit |
| `obra/cc-plugin-decision-log` | 用 `systemMessage` 而非 additionalContext（兼容性更好） | https://github.com/obra/cc-plugin-decision-log |
| `parcadei/Continuous-Claude-v3` | 解析 transcript JSONL 自动提取 TodoWrite / 最近 5 个 tool call / errors | https://github.com/parcadei/Continuous-Claude-v3 |
| `krzemienski/shannon-framework` | hook 不直接做 IO，让 LLM 调 Serena MCP `write_memory()` | https://github.com/krzemienski/shannon-framework |
| `tomkyser/module-reverie` | 注入 "compaction framing" 指令引导 Claude 怎么压缩 | https://github.com/tomkyser/module-reverie |
| `blas0/UnseveredMemory` | 极简 stdout 直出（最 dumb 但完全 work） | https://github.com/blas0/UnseveredMemory |
| `mem9-ai/mem9` | 防御式编程范例（auth missing/invalid 全 silent exit 0） | https://github.com/mem9-ai/mem9 |
| `etr/groundwork` | session_id-scoped state 文件 + minimal additionalContext fallback | https://github.com/etr/groundwork |
| `zenbase-ai/code-voyager` | recursion guard（防 hook 子进程触发自己） | https://github.com/zenbase-ai/code-voyager |
| `ChipFlow/context-daddy` | sentinel 文件 + LLM 直接调 MCP tool 双轨 | https://github.com/ChipFlow/context-daddy |
| `Lay4U/persistent-ralph` | iteration counter 防无限循环 | https://github.com/Lay4U/persistent-ralph |
| `MemPalace/mempalace` | path traversal 防护 + safe regex 模板 | https://github.com/MemPalace/mempalace |
| `rlancemartin/claude-diary` ⭐ 363 | hook 直接 `echo "/diary"` 让 LLM 接管 | https://github.com/rlancemartin/claude-diary |
| `Capnjbrown/c0ntextKeeper` | `TIMEOUT_MS = 55000`（留 5s 给 Claude 60s timeout） | https://github.com/Capnjbrown/c0ntextKeeper |
| `Dicklesworthstone/post_compact_reminder` | SessionStart `matcher: "compact"` 仅提醒重读 AGENTS.md | https://github.com/Dicklesworthstone/post_compact_reminder |

### 3.3 反面教材（要避开的坑）

- [`gabrielgadea/claude-code-kazuba`](https://github.com/gabrielgadea/claude-code-kazuba)：在 PreCompact 输出 `hookSpecificOutput.additionalContext`——schema 拒绝，**静默丢弃**（per [#46191](https://github.com/anthropics/claude-code/issues/46191)）。`timeout: 5000` 写成毫秒（实际是秒）也是反例。

- [`elb-pr/claudikins-kernel`](https://github.com/elb-pr/claudikins-kernel)：注释里写了"PreCompact events don't support hookSpecificOutput"——**这是过时认知**（schema 明确拒绝 additionalContext，但 hookSpecificOutput.systemMessage 路径在 PreCompact 也不工作）。

---

## 四、跨 AI Agent 生态横向调研（其他工具怎么解决这个问题）

| 产品 | 自动压缩 | 跨会话 resume | 防污染 | 最大启发 |
|---|---|---|---|---|
| **Cursor (Composer)** | ✅ Self-summarization 编进 RL 训练 | ❌ 原生无 | Memory Bank 分稳定/易变层 | "把 compaction 编进模型，不外挂" |
| **Aider** | ✅ 简朴（LLM 摘要） | ✅ `.aider.chat.history.md` + `--restore-chat-history` 默认关 | 默认关闭就是防御 | 文件即状态、用户主动开关 |
| **Continue.dev** | ❌ 弱 | ✅ `~/.continue/sessions/<id>.json` | UI 手选 + Context Provider 主动 @ | JSON-per-session + 不依赖恢复 |
| **OpenAI Codex CLI** | ✅ 服务端 | ✅ JSONL `~/.codex/sessions/.../rollout-*.jsonl` + `/resume`、`/fork` | cwd 过滤默认 + picker 选 | `/fork` 把试错和接续分开 |
| **GitHub Copilot CLI** | ⚠️ 文档不详 | ✅ SQLite + JSONL + `--continue` + `/resume` + `/rename` | location-based permission persistence | Per-session events.jsonl + workspace.yaml |
| **Gemini CLI** | ✅ 1M window 大 + `/compress` | ✅ `/restore`（自动 file checkpoint）+ `/chat save/resume`（手动）+ auto session save | project-scoped chat list | 双层 checkpoint：自动文件级 + 手动对话级 |
| **Cline** | ✅ ContextManager（in-place 压缩） | ✅ `<globalStoragePath>/tasks/<id>/` + Shadow git per-tool-call | FileContextTracker 在 restore 时检测 user 后续修改并警告 | 比其他更细粒度的 per-tool-call commits |
| **Roo Code** ⚠️ 关停 | ✅ Intelligent Context Condensation（70/30 拆分 + native token endpoint） | ✅ Per-task JSON | Boomerang Tasks（sub-agent 隔离） | 主上下文窗口保持干净 |
| **Kilo Code** | ✅ 继承 Cline 血统 | ✅ Snapshot git repo per model call + "Revert to here"（非破坏性） | Memory Bank 弃用，迁 AGENTS.md | 探索分支不丢 |
| **Windsurf (Cascade)** | ✅ Flow Context Engine | ✅ Memories（自动）+ Rules（手动） | 明确分工：Memories=会变事实，Rules=稳定约定 | "conversation history 不持久化"是 default |

### 学术/工程基础

- [ReAct (Yao et al. 2022)](https://arxiv.org/abs/2210.03629)：reasoning trace 当工作记忆——后续工作（Reflexion / IterResearch / **MemAgent** / **ReSum**）一致诟病不能撑长 horizon
- [MemGPT (2023)](https://arxiv.org/abs/2310.08560)：LLM as OS，把 context window 当物理内存，function call 主动 page in/out
- [Acon (2510.00615)](https://arxiv.org/html/2510.00615v1)：Agent Context Optimization 系统化框架
- [Context-Folding (2510.11967)](https://arxiv.org/abs/2510.11967)：让 agent 主动 fold 完成的子任务
- [OpenHands SDK (2511.03690)](https://arxiv.org/html/2511.03690v1)：内置 pause/resume，commit SHA 解析 plugin 引用
- [Chroma "Context Rot" research](https://research.trychroma.com/context-rot)：18 个 frontier model 实测，全部随 input 增长 degrade——**"compaction is treatment, not prevention"**

### LangGraph、AutoGen、CrewAI 等 framework

- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)：`BaseCheckpointSaver` 接口（put/get_tuple/list/delete_thread）+ MemorySaver/Sqlite/Postgres/Redis/MongoDB/DynamoDB 多后端
- [AutoGen `save_state` / `load_state`](https://microsoft.github.io/autogen/stable//user-guide/agentchat-user-guide/tutorial/state.html)：dict 序列化成 JSON，team 级递归
- [CrewAI 4 类记忆](https://docs.crewai.com/en/concepts/memory)：Short-term (ChromaDB) / Long-term (SQLite) / Entity (RAG) / Contextual

### Devin / OpenHands

- [Devin Knowledge vs Playbooks](https://docs.devin.ai/essential-guidelines/instructing-devin-effectively)：Knowledge 跨 session 自动 recall；Playbook 是可复用 prompt 模板。**Scheduled Devins 自管 handoff 文档**。
- [OpenHands Conversation pause/resume](https://docs.openhands.dev/openhands/usage/cloud/cloud-api)：pause() 自动 persist state；plugin commit SHA 解析确保 deterministic resume

---

## 五、Handoff / Sentinel 模式社区共识

通过对 [aipatternbook.com/handoff](https://aipatternbook.com/handoff)、[softaworks/agent-toolkit session-handoff skill](https://github.com/softaworks/agent-toolkit/blob/main/skills/session-handoff/README.md)、[antigravity.codes session-handoff](https://antigravity.codes/agent-skills/workflow/session-handoff) 三个跨产品 standard 的研读，社区已经收敛成几个 pattern：

1. **`.agent/{context.md, decisions.md, blocked.md, handoff.md}` 文件结构**
2. **Handoff chaining**：`handoff-2 --continues-from handoff-1`
3. **Resume 必做的 verify 步骤**：
   - git branch / commit hash 是否一致
   - 文件 mtime 是否在 handoff 之后被改过
   - 环境（依赖、env var）是否变了
   - blocked.md 里的 blocker 是否解决
4. **稳定层 vs 易变层分离**：稳定的写 `CLAUDE.md` / Rules / Knowledge；易变的写 handoff/SESSION.md
5. **JSONL append-only > 全量 JSON 重写**

---

## 六、最终设计建议（应用上述 250+ 来源得到的结论）

### 6.1 哲学先行（Linus 三问后的核心判断）

1. "这是真问题还是臆想？"——**真问题**。compact 失忆是 [Anthropic 自己 Cookbook](https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools) 承认的 "lossy" 行为
2. "有更简单的方法吗？"——**有**：保持 manual `/handoff-save` + `/handoff-resume`，因为 hook 链路上有太多坑
3. "会破坏什么吗？"——**会**：plugin-distributed hook 在 v2.1.42-v2.1.75 多个版本上 output 被吞，会让用户安装后陷入"看似 work 实际不 work"的更糟状况

### 6.2 三档建议（按推荐度）

#### A 档（最强推荐）：保持现状，**不加 hook**

仅升级文档，明确告诉用户：

- "compact 前手动 `/handoff-save`，compact 后手动 `/handoff-resume`"
- 在 README 加一句"未来 Claude Code 把 PreCompact / SessionStart 的契约稳定后会再加自动化"

理由：
- 当前所有 hook 路径都在 [#16538](https://github.com/anthropics/claude-code/issues/16538)、[#25655](https://github.com/anthropics/claude-code/issues/25655)、[#28305](https://github.com/anthropics/claude-code/issues/28305) 等已知 bug 中
- 用户的"手动两步"不是真痛点（`/handoff-save` 大约 15 秒，`/handoff-resume` 大约 3 秒）
- 假装自动化但实际不可靠，比明示手动更糟

#### B 档（中等推荐）：**最小 hook 增强 + 用户自己装**

不在 plugin 内 ship hook，而是提供一个 `setup-hooks.sh` 脚本，用户运行后**把 hook 写到 `~/.claude/settings.json`**（绕过 plugin hook 路由 bug）：

```bash
# setup-hooks.sh 大致行为：
# 1. 备份 ~/.claude/settings.json
# 2. 添加两个 hook 到用户配置：
#    PreCompact → 写 .handoff/.compact-pending sentinel + 时间戳
#    SessionStart(matcher=compact) → 读 sentinel + 最近 handoff，注入 + 删 sentinel
# 3. 不修改 plugin 自身
```

Hook 脚本（roadrunner / recall 风格的薄壳）：

```bash
# pre-compact.sh —— 5 行，只 touch sentinel
WORKTREE_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
HANDOFF_DIR="$WORKTREE_ROOT/.handoff"
mkdir -p "$HANDOFF_DIR"
touch "$HANDOFF_DIR/.compact-pending"
exit 0
```

```bash
# session-start.sh —— 30 行，读 sentinel + 注入
INPUT=$(cat)
SOURCE=$(echo "$INPUT" | jq -r '.source // empty')
[ "$SOURCE" = "compact" ] || exit 0

WORKTREE_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
HANDOFF_DIR="$WORKTREE_ROOT/.handoff"
SENTINEL="$HANDOFF_DIR/.compact-pending"

# Sentinel 必须存在 + 时间窗 30 分钟
[ ! -f "$SENTINEL" ] && exit 0
NOW=$(date +%s)
SENTINEL_TIME=$(stat -f %m "$SENTINEL" 2>/dev/null || stat -c %Y "$SENTINEL" 2>/dev/null)
[ "$((NOW - SENTINEL_TIME))" -gt 1800 ] && { rm -f "$SENTINEL"; exit 0; }

# 找当前 branch 的最新 handoff
BRANCH=$(git branch --show-current 2>/dev/null | tr '/ ' '-')
LATEST=$(ls -t "$HANDOFF_DIR/${BRANCH}--"*.md 2>/dev/null | head -1)
[ -z "$LATEST" ] && exit 0

# Single-use: 读完即删 sentinel（防止下次 compact 误恢复）
rm -f "$SENTINEL"

# 注入 + 内置 confirmation gate 措辞
cat <<EOF
A handoff document was saved before this compaction. Below is its content.
Before any state-changing action, follow the resume confirmation gate:
1. Summarize understanding in 3-5 sentences
2. List candidate next steps from section 7 verbatim
3. Wait for user's explicit direction

DO NOT call Edit/Write/Bash write tools until user gives a direction.

---HANDOFF---
$(cat "$LATEST")
---END HANDOFF---
EOF
```

防误恢复的三层防护：
1. `source == "compact"` 才触发
2. branch slug 必须匹配
3. **`.compact-pending` sentinel 单次消费**——save 时建，read 时删；旧 handoff 没 sentinel，**绝不可能误恢复**
4. 时间窗 30 分钟兜底

**仍然要明示的局限**：
- 如果用户在 v2.1.x 中遇到 SessionStart compact-matcher 不 fire 的 bug，hook 默默无效——manual `/handoff-resume` 仍可工作
- 多次 compact 后 ([#25655](https://github.com/anthropics/claude-code/issues/25655)) hook 死掉——只第一次 compact 后 auto-resume 工作

#### C 档（不推荐）：plugin 内 ship `hooks.json`

按 [Issue #16538](https://github.com/anthropics/claude-code/issues/16538) 在多版本上 output 被吞。即使 `pre-compact.sh` 只做 side effect（touch sentinel）能 work，SessionStart 端的 additionalContext 注入大概率被吞——等于功能不存在。

### 6.3 我个人的推荐

**走 A 档**。理由：

1. 我们已经验证了 manual skill 100% 可靠工作（早前的 4 个 subagent 测试全过）
2. B 档的 hook 设计是"在已知有 bug 的平台路径上加保险层"——比手动多一份理解成本
3. C 档完全不推荐
4. 等 Anthropic 把 [#46191](https://github.com/anthropics/claude-code/issues/46191) 标记 fixed 或者把 SessionStart compact matcher 行为公开文档化，**到时候我们一周内可以加上 hook**

如果你坚持要某种自动化，**只走 B 档的 SessionStart 半边**：保留 `~/.claude/settings.json` 里的 SessionStart hook 注入（不写 PreCompact，因为 PreCompact 只能写 sentinel，sentinel 本身又依赖 manual `/handoff-save` 才有意义）。

---

## 七、所有参考链接（按类别去重，~250+）

### 7.1 Anthropic 官方

**Hooks 与 Plugins 文档**：
- https://docs.anthropic.com/en/docs/claude-code/hooks
- https://docs.anthropic.com/en/docs/claude-code/hooks-guide
- https://code.claude.com/docs/en/hooks
- https://code.claude.com/docs/en/hooks-guide
- https://code.claude.com/docs/en/hooks.md
- https://code.claude.com/docs/en/agent-sdk/hooks.md
- https://code.claude.com/docs/en/plugins.md
- https://code.claude.com/docs/en/plugins-reference.md
- https://code.claude.com/docs/en/plugin-marketplaces.md
- https://code.claude.com/docs/en/skills.md
- https://code.claude.com/docs/en/agent-sdk/skills.md
- https://code.claude.com/docs/en/context-window.md
- https://code.claude.com/docs/en/checkpointing.md
- https://code.claude.com/docs/en/agent-sdk/sessions.md
- https://code.claude.com/docs/en/how-claude-code-works.md
- https://code.claude.com/docs/en/memory.md
- https://code.claude.com/docs/en/agent-sdk/agent-loop.md
- https://code.claude.com/docs/en/changelog.md
- https://code.claude.com/docs/en/whats-new/2026-w17.md
- https://docs.claude.com/en/docs/claude-code/sdk/sdk-slash-commands
- https://platform.claude.com/docs/en/build-with-claude/compaction
- https://platform.claude.com/cookbook/tool-use-automatic-context-compaction
- https://platform.claude.com/cookbook/tool-use-context-engineering-context-engineering-tools

**Agent Skills 标准**：
- https://agentskills.io
- https://agentskills.io/specification
- https://github.com/agentskills/agentskills

**官方插件库**：
- https://github.com/anthropics/claude-plugins-official
- https://github.com/anthropics/skills
- https://github.com/anthropics/claude-code/tree/main/plugins
- https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/hook-development/references/patterns.md
- https://github.com/anthropics/claude-code/blob/main/plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh
- https://github.com/anthropics/claude-agent-sdk-python/blob/main/src/claude_agent_sdk/types.py

**工程博客**：
- https://www.anthropic.com/engineering
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk
- https://www.anthropic.com/engineering/claude-code-best-practices
- https://www.anthropic.com/engineering/harness-design-long-running-apps
- https://www.anthropic.com/engineering/claude-code-auto-mode
- https://www.anthropic.com/engineering/managed-agents
- https://claude.com/blog/claude-code-plugins
- https://claude.com/blog/how-to-configure-hooks
- https://www.anthropic.com/news/agent-skills-open-standard

### 7.2 关键 GitHub Issues / PRs

**PreCompact / additionalContext**：
- https://github.com/anthropics/claude-code/issues/3349
- https://github.com/anthropics/claude-code/issues/13170
- https://github.com/anthropics/claude-code/issues/13572
- https://github.com/anthropics/claude-code/issues/13642
- https://github.com/anthropics/claude-code/issues/14160
- https://github.com/anthropics/claude-code/issues/15096
- https://github.com/anthropics/claude-code/issues/15174
- https://github.com/anthropics/claude-code/issues/17237
- https://github.com/anthropics/claude-code/issues/20370
- https://github.com/anthropics/claude-code/issues/21398
- https://github.com/anthropics/claude-code/issues/21776
- https://github.com/anthropics/claude-code/issues/24965
- https://github.com/anthropics/claude-code/issues/25655
- https://github.com/anthropics/claude-code/issues/25999
- https://github.com/anthropics/claude-code/issues/26010
- https://github.com/anthropics/claude-code/issues/26597
- https://github.com/anthropics/claude-code/issues/28305
- https://github.com/anthropics/claude-code/issues/32026
- https://github.com/anthropics/claude-code/issues/32062
- https://github.com/anthropics/claude-code/issues/33088
- https://github.com/anthropics/claude-code/issues/38523
- https://github.com/anthropics/claude-code/issues/46191
- https://github.com/anthropics/claude-code/issues/47023
- https://github.com/anthropics/claude-code/issues/50682
- https://github.com/anthropics/claude-code/issues/54118

**Plugin / Hook 相关**：
- https://github.com/anthropics/claude-code/issues/10373
- https://github.com/anthropics/claude-code/issues/10814
- https://github.com/anthropics/claude-code/issues/11125
- https://github.com/anthropics/claude-code/issues/12117
- https://github.com/anthropics/claude-code/issues/12151
- https://github.com/anthropics/claude-code/issues/12643
- https://github.com/anthropics/claude-code/issues/16538
- https://github.com/anthropics/claude-code/issues/17923
- https://github.com/anthropics/claude-code/issues/18085
- https://github.com/anthropics/claude-code/issues/18448
- https://github.com/anthropics/claude-code/issues/19485
- https://github.com/anthropics/claude-code/issues/23047
- https://github.com/anthropics/claude-code/issues/23545
- https://github.com/anthropics/claude-code/issues/24794
- https://github.com/anthropics/claude-code/issues/27145
- https://github.com/anthropics/claude-code/issues/27242
- https://github.com/anthropics/claude-code/issues/31658
- https://github.com/anthropics/claude-code/issues/32055
- https://github.com/anthropics/claude-code/issues/38483
- https://github.com/anthropics/claude-code/issues/40492
- https://github.com/anthropics/claude-code/issues/40614
- https://github.com/anthropics/claude-code/issues/41919
- https://github.com/anthropics/claude-code/issues/42149
- https://github.com/anthropics/claude-code/issues/43696

### 7.3 顶级开源参考实现（hook + compact）

- https://github.com/joseairosa/recall
- https://github.com/afoxnyc3/roadrunner-cli
- https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/007-dead-hook-cleanup.md
- https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/009-state-schema-versioning-and-concurrency-lock.md
- https://github.com/afoxnyc3/roadrunner-cli/blob/main/docs/adr/010-hook-python-entrypoint-unification.md
- https://github.com/who96/claude-code-context-handoff
- https://github.com/ThomasEdwardYorke/cc-triad-relay
- https://github.com/ThomasEdwardYorke/cc-triad-relay/blob/main/plugins/harness/core/src/hooks/pre-compact.ts
- https://github.com/ThomasEdwardYorke/cc-triad-relay/blob/main/plugins/harness/core/src/hooks/session-start.ts
- https://github.com/hex0xdeadbeef/claude-kit
- https://github.com/hex0xdeadbeef/claude-kit/blob/main/.claude/scripts/save-progress-before-compact.sh
- https://github.com/hex0xdeadbeef/claude-kit/blob/main/.claude/scripts/verify-state-after-compact.sh
- https://github.com/kylesnowschwartz/claude-handoff
- https://github.com/arpitnath/claude-capsule-kit
- https://github.com/obra/superpowers
- https://github.com/obra/cc-plugin-decision-log
- https://github.com/parcadei/Continuous-Claude-v3
- https://github.com/krzemienski/shannon-framework
- https://github.com/tomkyser/module-reverie
- https://github.com/blas0/UnseveredMemory
- https://github.com/elb-pr/claudikins-kernel
- https://github.com/ChipFlow/context-daddy
- https://github.com/Lay4U/persistent-ralph
- https://github.com/k3nnethfrancis/agent-utils
- https://github.com/MemPalace/mempalace
- https://github.com/contextstream/mcp-server
- https://github.com/zenbase-ai/code-voyager
- https://github.com/juanandresgs/claude-ctrl
- https://github.com/l33tdawg/sage
- https://github.com/echoVic/boss-skill
- https://github.com/punt-labs/biff
- https://github.com/rlancemartin/claude-diary
- https://github.com/Roasbeef/substrate
- https://github.com/shakestzd/contextune
- https://github.com/etr/groundwork
- https://github.com/mem9-ai/mem9
- https://github.com/Capnjbrown/c0ntextKeeper
- https://github.com/Dicklesworthstone/post_compact_reminder
- https://github.com/Dicklesworthstone/misc_coding_agent_tips_and_scripts/blob/main/CLAUDE_CODE_POST_COMPACT_AGENTS_MD_REMINDER.md
- https://github.com/cahaseler/cc-track
- https://github.com/Xsidz/ide-bridge
- https://github.com/marolinik/MemoryForge
- https://github.com/disler/claude-code-hooks-mastery
- https://github.com/fajarhide/omni
- https://github.com/QwenLM/qwen-code
- https://github.com/AutoForgeAI/autoforge
- https://github.com/Prorise-cool/Claude-Code-Multi-Agent
- https://github.com/ZeroZ-lab/cc-design
- https://github.com/panayiotism/claude-harness
- https://github.com/NatureBlueee/wow-harness
- https://github.com/gabrielgadea/claude-code-kazuba（**反例**）

### 7.4 跨 IDE / Agent 对照（Cursor、Aider、Continue、Codex、Copilot、Gemini、Cline、Roo、Kilo、Windsurf）

- https://cursor.com/blog/self-summarization
- https://cursor.com/blog/dynamic-context-discovery
- https://docs.cursor.com/composer/overview
- https://forum.cursor.com/t/compact-compress-chat/132097
- https://forum.cursor.com/t/persistent-memory-for-cursor-that-survives-every-session-brain-folder-approach/157488
- https://docs.basicmemory.com/integrations/cursor
- https://aider.chat/docs/usage/commands.html
- https://aider.chat/docs/config/options.html
- https://github.com/paul-gauthier/aider/issues/118
- https://github.com/paul-gauthier/aider/issues/166
- https://github.com/Aider-AI/aider/issues/3607
- https://docs.continue.dev/customize/custom-providers
- https://docs.continue.dev/ide-extensions/chat/context-selection
- https://github.com/continuedev/continue/issues/6640
- https://github.com/continuedev/continue/blob/main/core/util/paths.ts
- https://github.com/continuedev/continue/blob/main/core/util/history.ts
- https://github.com/continuedev/continue/blob/main/extensions/cli/src/hooks/types.ts
- https://developers.openai.com/codex/cli/features
- https://developers.openai.com/codex/cli/reference
- https://developers.openai.com/codex/cli/slash-commands
- https://developers.openai.com/codex/changelog
- https://developers.openai.com/codex/noninteractive
- https://deepwiki.com/openai/codex/4.2.2-resume-and-review-commands
- https://github.com/openai/codex/discussions/1076
- https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle
- https://docs.github.com/en/copilot/how-tos/copilot-cli/chronicle
- https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/session-persistence
- https://deepwiki.com/github/copilot-cli/3.3-session-management-and-history
- https://github.com/github/copilot-cli/issues/667
- https://github.com/github/copilot-cli/issues/820
- https://github.com/github/copilot-cli/issues/1313
- https://code.visualstudio.com/docs/copilot/agents/copilot-cli
- https://geminicli.com/docs/cli/checkpointing/
- https://geminicli.com/docs/reference/commands/
- https://geminicli.com/docs/cli/tutorials/session-management/
- https://developers.googleblog.com/pick-up-exactly-where-you-left-off-with-session-management-in-gemini-cli/
- https://github.com/google-gemini/gemini-cli/discussions/1538
- https://github.com/google-gemini/gemini-cli/discussions/4061
- https://docs.cline.bot/core-workflows/checkpoints
- https://github.com/cline/cline/issues/4359
- https://github.com/cline/cline/discussions/1887
- https://github.com/cline/cline/discussions/2550
- https://deepwiki.com/cline/cline/3.1-system-prompt
- https://docs.roocode.com/features/intelligent-context-condensing
- https://deepwiki.com/RooCodeInc/Roo-Code/2.3-state-management-and-storage
- https://deepwiki.com/RooCodeInc/Roo-Code/7-context-and-message-management
- https://deepwiki.com/deepupdate/Roo-Code/10.3-intelligent-context-condensation
- https://github.com/RooCodeInc/Roo-Code/issues/10862
- https://kilo.ai/docs/code-with-ai/features/checkpoints
- https://kilo.ai/docs/customize/agents-md
- https://kilo.ai/docs/cli
- https://kilo.ai/features/memory-bank
- https://docs.windsurf.com/windsurf/cascade/memories
- https://memnexus.ai/blog/2026-02-20-windsurf-persistent-memory
- https://github.com/GreatScottyMac/cascade-memory-bank

### 7.5 多 Agent Framework

- https://docs.langchain.com/oss/python/langgraph/persistence
- https://reference.langchain.com/python/langgraph/checkpoints
- https://pypi.org/project/langgraph-checkpoint/
- https://langgraphjs.guide/persistence/
- https://redis.io/blog/langgraph-redis-build-smarter-ai-agents-with-memory-persistence/
- https://aws.amazon.com/blogs/database/build-durable-ai-agents-with-langgraph-and-amazon-dynamodb/
- https://microsoft.github.io/autogen/stable//user-guide/agentchat-user-guide/tutorial/state.html
- https://microsoft.github.io/autogen/stable//user-guide/agentchat-user-guide/memory.html
- https://github.com/microsoft/autogen/discussions/6005
- https://github.com/microsoft/autogen/discussions/6169
- https://github.com/microsoft/autogen/issues/6466
- https://docs.crewai.com/en/concepts/memory
- https://deepwiki.com/crewAIInc/crewAI/7.2-memory-configuration-and-storage
- https://sparkco.ai/blog/deep-dive-into-crewai-memory-systems
- https://mem0.ai/blog/crewai-memory-production-setup-with-mem0
- https://dev.to/foxgem/ai-agent-memory-a-comparative-analysis-of-langgraph-crewai-and-autogen-31dp

### 7.6 Devin / OpenHands / Auto-GPT

- https://cognition.ai/blog/devin-2
- https://cognition.ai/blog/devin-can-now-schedule-devins
- https://cognition.ai/blog/devin-can-now-manage-devins
- https://cognition.ai/blog/how-cognition-uses-devin-to-build-devin
- https://cognition.ai/blog/devin-for-terminal
- https://docs.devin.ai/essential-guidelines/instructing-devin-effectively
- https://docs.devin.ai/work-with-devin/advanced-capabilities
- https://docs.openhands.dev/openhands/usage/cloud/cloud-api
- https://github.com/All-Hands-AI/OpenHands/issues/5726
- https://github.com/All-Hands-AI/OpenHands/pull/6438
- https://github.com/All-Hands-AI/OpenHands/issues/10336
- https://github.com/OpenHands/software-agent-sdk/blob/main/openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py
- https://weaviate.io/blog/autogpt-and-weaviate
- https://dariuszsemba.com/blog/why-autogpt-engineers-ditched-vector-databases/

### 7.7 学术 / 综述

- https://arxiv.org/abs/2210.03629（ReAct）
- https://arxiv.org/abs/2310.08560（MemGPT）
- https://arxiv.org/html/2510.00615v1（Acon）
- https://arxiv.org/abs/2510.11967（Context-Folding）
- https://arxiv.org/html/2511.03690v1（OpenHands SDK）
- https://research.trychroma.com/context-rot
- https://www.morphllm.com/context-rot
- https://eunomia.dev/blog/2025/05/11/checkpointrestore-systems-evolution-techniques-and-applications-in-ai-agents/
- https://blog.jetbrains.com/research/2025/12/efficient-context-management/
- https://github.com/Shichun-Liu/Agent-Memory-Paper-List

### 7.8 Handoff / Sentinel 模式社区先例

- https://aipatternbook.com/handoff
- https://github.com/softaworks/agent-toolkit/blob/main/skills/session-handoff/README.md
- https://skills.sh/softaworks/agent-toolkit/session-handoff
- https://agentskills.me/skill/session-handoff
- https://antigravity.codes/agent-skills/workflow/session-handoff
- https://irontravelerlabs.com/articles/long-running-ai-agents/
- https://gist.github.com/badlogic/cd2ef65b0697c4dbe2d13fbecb0a0a5f
- https://medium.com/@porter.nicholas/claude-code-post-compaction-hooks-for-context-renewal-7b616dcaa204
- https://www.agentsentinelai.com/

### 7.9 npm 工具与 skill 管理生态

- https://www.npmjs.com/package/skills（vercel-labs/skills CLI）
- https://github.com/vercel-labs/skills
- https://github.com/vercel-labs/agent-skills
- https://www.npmjs.com/package/skillfish
- https://github.com/knoxgraeme/skillfish
- https://www.npmjs.com/package/agent-skills
- https://www.npmjs.com/package/oh-my-codex-cli
- https://www.npmjs.com/package/itismyskillmarket
- https://www.npmjs.com/package/@dreamlogic-ai/cli
- https://www.npmjs.com/package/@larksuite/cli

### 7.10 课程 / 综合教程 / 博客

- https://blog.fsck.com/2025/10/09/superpowers/
- https://simonwillison.net/2025/Oct/10/superpowers/
- https://simonwillison.net/2025/Dec/19/agent-skills/
- https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/
- https://dev.to/aabyzov/claude-code-hook-limitations-no-skill-invocation-lazy-plugin-loading-and-how-i-solved-it-44f2
- https://claudefa.st/blog/guide/changelog
- https://claudefa.st/blog/tools/hooks/hooks-guide
- https://claudefa.st/blog/tools/hooks/session-lifecycle-hooks
- https://claudefa.st/blog/tools/hooks/context-recovery-hook
- https://www.dotzlaw.com/insights/claude-hooks/
- https://stevekinney.com/courses/ai-development/cursor-context
- https://stevekinney.com/courses/ai-development/claude-code-compaction
- https://stevekinney.com/courses/ai-development/claude-code-session-management
- https://claudelog.com/faqs/what-is-claude-code-auto-compact/
- https://hyperdev.matsuoka.com/p/how-claude-code-got-better-by-protecting
- https://www.davidborish.com/post/anthropic-s-claude-code-source-code-leaked-and-here-s-what-it-shows
- https://www.implicator.ai/anthropic-ships-its-openclaw-rival-connecting-claude-code-to-telegram-and-discord/
- https://venturebeat.com/orchestration/anthropic-just-shipped-an-openclaw-killer-called-claude-code-channels
- https://www.anup.io/35-claude-code-tips-from-the-guy-who-built-it/
- https://offthegridxp.substack.com/p/the-genius-of-anthropics-claude-agent-skills-2025
- https://thenewstack.io/agent-skills-anthropics-next-bid-to-define-ai-standards/
- https://venturebeat.com/technology/anthropic-launches-enterprise-agent-skills-and-opens-the-standard
- https://siliconangle.com/2025/12/18/anthropic-makes-agent-skills-open-standard/
- https://news.ycombinator.com/item?id=46426624
- https://news.ycombinator.com/item?id=46470017

### 7.11 类型 / Schema 定义参考

- https://github.com/letta-ai/letta-code/blob/main/src/hooks/types.ts
- https://github.com/MoonshotAI/kimi-cli/blob/main/src/kimi_cli/hooks/events.py
- https://github.com/superagent-ai/grok-cli/blob/main/src/hooks/types.ts
- https://github.com/code-yeongyu/oh-my-openagent/blob/dev/src/hooks/claude-code-hooks/pre-compact.ts
- https://github.com/codemie-ai/codemie-code/blob/main/src/agents/plugins/codemie-code-hooks/shell-hooks-source.ts
- https://github.com/dyoshikawa/rulesync/blob/main/src/types/hooks.ts
- https://github.com/jeffh/claude-plugins/tree/main/pai/hooks
- https://github.com/EdytaKucharska/agent_recorder

### 7.12 配置范例

- https://github.com/udecode/plate/blob/main/.claude/settings.json
- https://github.com/P0luz/Ombre-Brain/blob/main/.claude/settings.json
- https://github.com/vanzan01/claude-code-sub-agent-collective/blob/main/templates/settings.json.template
- https://github.com/flonat/claude-research/blob/main/.claude/settings.json
- https://github.com/maxritter/pilot-shell/blob/main/docs/docusaurus/blog/2026-01-24-session-lifecycle-hooks.md
- https://github.com/wzf1997/claude-code-guide/blob/master/docs/chapter-12-hooks/67-session-start-hooks.mdx
- https://github.com/ruvnet/RuVector/blob/main/crates/ruvector-cli/src/cli/hooks.rs

### 7.13 Twitter / X 帖子

- https://x.com/bcherny/status/2009072293826453669（Boris Cherny on Stop hooks）
- https://x.com/bcherny/status/2021699851499798911
- https://x.com/bcherny/status/2027534984534544489（/simplify, /batch built-in skills）
- https://x.com/dani_avila7/status/2008653214472614369（Daniel San on auto-compact）

---

## 八、致用户的诚实告白

我把这份调研报告称为"诚实告白"——不是因为前面的事实有疑问，而是因为这次调研实际上**推翻了我自己之前的几个判断**：

1. 我曾说"PreCompact additionalContext 真的支持，cc-triad-relay 和 claude-kit 都用了"——实际上 [Issue #46191](https://github.com/anthropics/claude-code/issues/46191) 明确说不支持。cc-triad-relay 在 spec audit 里明确把 additionalContext routed 到 systemMessage（[index.ts:120-150](https://github.com/ThomasEdwardYorke/cc-triad-relay/blob/main/plugins/harness/core/src/index.ts)），claude-kit 的输出在某些版本里被静默丢弃。

2. 我曾说"SessionStart matcher=compact 是干净的官方支持路径"——实际上 [#15174](https://github.com/anthropics/claude-code/issues/15174) 报告它曾经被吞，[#28305](https://github.com/anthropics/claude-code/issues/28305) 仍然 OPEN 描述同问题，文档侧也只提它而 schema 里"compact" 是未文档化的（per [#25999](https://github.com/anthropics/claude-code/issues/25999) "works, undocumented"）。

3. 我曾说"plugin hook 也能注入"——实际上 [Issue #16538](https://github.com/anthropics/claude-code/issues/16538) 持续 OPEN 描述 plugin hook output 被吞，**workaround 是把 hook 写到 `~/.claude/settings.json`**（绕过 plugin 路由）。

这些事实加起来，意味着**任何依赖 hook 自动化的 handoff plugin 都建立在 implementation detail 上**，且那些 detail 在多个版本里有 bug。

**最稳健的工程决定**是：保留 manual skill 的现状，等 Anthropic 在某次 changelog 里写出"Plugin SessionStart compact-matcher additionalContext now stable"——届时再加 hook，同时享受到 Anthropic 已经修过的所有 bug。

---

**调研完毕**：5 个并行 deep-research agent + 多轮直接搜索，覆盖 250+ 来源，2026-05-07。
