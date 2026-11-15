# 备忘录：Herdr `agent_session` 为空 /「按 ID 寻址失败」

> 状态：**等上游（Herdr）修复后回来跟进**。本仓库已回滚到 upstream 基线，未改动 Herdr；本地那份 Herdr 检出只用来读源码排查，不做任何修改。
> 记录日期：2026-11-15

## 结论先行

- CodeBuddy 作为 agent 在 Herdr 里**可见、状态正确**，但 `agent_session` 一直为空，导致无法按 session ID 找到它。
- 根因不在插件：插件在 `SessionStart` 一直有上报 `--agent-session-id` 和 `codebuddy --resume <id>`，是 **Herdr 0.9.3 把非官方 `(source, agent)` 对的这些字段直接丢掉了**。
- Herdr 0.9.3 的「官方 source」白名单写死在 Rust 里，没有配置开关；因此**插件侧没有干净的解法**，干净修复落在 Herdr 侧。
- 曾尝试用插件侧 hack 绕过，造成严重回归，已回滚；详见下文，避免重蹈。

## 本次收尾时的快照

| 项 | 状态 |
| --- | --- |
| 本仓库 `main` / `origin/main` | `b68fc18`（revert `3516366`），内容 == upstream 基线 `9c54fab` |
| 插件 | 已 `node install.js install`；10 个 hook 事件齐全；`npm test` 16/16 |
| 已安装 hook | == 仓库内基线脚本 |
| 行为 | agent 可见，`idle/working/blocked` 正确；`agent_session` 仍为空（已知限制） |
| Herdr | 未改动；本仓库不依赖任何 Herdr 本地修改 |
| 排查用的测试 tab | 已清理 |

`README.md` 的 `SessionStart` 一行写「with the session ID ... command when available」，描述的是**插件发出的内容**，不是 Herdr 实际存下来的结果；别被它误导以为问题已解。

## 问题到底是什么

「按 ID 寻址」在实践中是用 `herdr agent list` / `pane get` 按 `agent_session.value` 过滤（`agent get` 接的是 pane id / 唯一 live agent 名，不吃原生 session id）。`value` 为空 → 过滤不到 → 「找不到」。
补充：这个字段来自 Herdr 的 *persisted agent session*，不是任意元数据，只有被认可的 source 才能写入。

## 根因（Herdr 0.9.3；路径均为 Herdr 仓库内相对路径）

- `src/agent_resume.rs` — `is_official_agent_source()` 是**硬编码 `matches!` 白名单**（`herdr:claude`/`claude`、`herdr:codex`/`codex` 等），`custom:codebuddy`/`codebuddy` 不在其中。
- 同文件 `session_ref_from_report()` 第一句即 `if !is_official_agent_source(...) { return None; }` → `pane report-agent` 与 `pane report-agent-session`（`src/app/api/panes.rs` 的两个 handler 都走它）里的 `--agent-session-id` 被丢弃。
- `plan()`（恢复命令）同样先过白名单，非官方 pair 直接 `None`。
- `src/config/` 无相关配置；**没有**配置驱动的白名单。
- 未合并的 `origin/architecture/agent-registry` 分支把这块改成了 TOML 数据驱动（`vendor/agent-registry/agents/<id>/*.toml`，`plan()` → `registry.session_profile_for_exact_report_pair()`，`is_official_agent_source()` → `agents::bundled_report_pair()`）。排查时该分支**尚无 codebuddy 条目**。这正是「能否用配置做白名单」的方向，但当前运行的 0.9.3 用不上。

## 试过并回滚的方案（勿重蹈）

`3516366`「两报告」设计：state 仍用 `custom:codebuddy`/`codebuddy`，另发一条 `pane report-agent-session` 借内置 pair `herdr:codex`/`codex` 来填 `agent_session`。新 pane 里确实能填上，但：

- `src/terminal/state.rs` — `current_session_owner_conflicts()`（在 `set_hook_authority_with_session_ref()` 里）：一旦 persisted session owner 是 `herdr:codex`，所有 `custom:codebuddy` 的 state 上报都被判冲突而丢弃。
- `src/app/actions.rs` — `HookAgentReleased` 对 official source 的 `release-agent` 是 no-op；`SessionEnd` 清不掉 foreign 的 `herdr:codex` persisted session。
- 后果：`/exit` 后**同 pane 重启，agent 永远不可见**（用户实测发现的严重回归）。
- 另：`effective_agent_label()` 会把借用的内置 pair 显示成 `codex` 而不是 `codebuddy`。
- 结论：state 与 session 必须**同源**才不冲突；而插件自身上报的 source 又过不了白名单 → 死结。已 `git revert`（`b68fc18`）。

一句话：**「不改 Herdr 也能修」的那条路只有这个 hack，且已证伪**；所以收尾回到基线等上游，不是放弃，是避免在插件里堆一个冒充内置 agent 的脆弱实现。

## 回来跟进时：怎么判断「Herdr 解决了」

按优先级核对（以当时的 Herdr 版本为准，重新看源码 / changelog / agent registry 是否合入主线）：

1. **白名单是否数据/配置驱动、是否新增了 CodeBuddy**：`is_official_agent_source` 是否还是硬编码 `matches!`；agent registry 目录里有没有 `codebuddy`。
2. **认哪一对**：是放行 `custom:codebuddy`/`codebuddy`，还是新增 `herdr:codebuddy`/`codebuddy`（或同等）。这决定插件要不要改 `--source`。
3. **release 语义**：确认 `HookAgentReleased` 对新的官方 pair 不会把 `release-agent` 变成 no-op，否则 `/exit` 会留悬空 agent（这是 Herdr 侧最小改动里最容易漏的一点）。
4. **resume**：`plan()` 能否生成 `codebuddy --resume <id>`。

### 端到端验收（建议隔离 tab/pane，别在当前会话 pane 里试）

命令以当时 Herdr CLI / SKILL 为准，覆盖：

- 新 pane 起 `codebuddy` → agent 可见、`agent_session.value == 该 Session ID`、状态 `idle`。
- 发一句提问 → 状态变 `working`，结束回 `idle`。
- `/exit` → 不留悬空 agent。
- **同 pane 再起一次 `codebuddy`**（这次踩坑的场景）→ agent 仍可见，`agent_session` 更新为新 ID，`codebuddy --resume` 可作为恢复目标。
- 重启/恢复后按 ID 能在 `agent list` / `pane get` 里过滤到。

### 若上游已解决，本仓库可能要做的

- 视上游认哪一对，把 `herdr-codebuddy-hook.js` 里 `--source custom:codebuddy` 改成 Herdr 认可的官方 pair（若上游直接放行 `custom:codebuddy` 则**零改动**）。
- 相应更新 `README.md` 的 `SessionStart` 行与 `test.js` 里的 argv 断言，补 `buildArgs` 的单测。
- 保持「审核友好」：只改相关行，不顺手重写注释/格式。

## 关键符号索引（便于日后重新核对）

| 符号 | 位置（Herdr 仓库内） | 作用 |
| --- | --- | --- |
| `is_official_agent_source` | `src/agent_resume.rs` | 官方 `(source, agent)` 白名单（根因） |
| `session_ref_from_report` | `src/agent_resume.rs` | 丢弃非官方 pair 的 `--agent-session-id` |
| `plan` | `src/agent_resume.rs` | 生成恢复 argv，同样过白名单 |
| `handle_pane_report_agent[_session]` | `src/app/api/panes.rs` | 两个 socket handler 都调上面的函数 |
| `HookStateReported` / `HookAgentReleased` | `src/app/actions.rs` | 路由到 state / release；official release 是 no-op |
| `current_session_owner_conflicts` | `src/terminal/state.rs` | 两来源冲突 → 丢弃 state（回归根因） |
| `effective_agent_label` | `src/terminal/state.rs` | 决定 agent 显示名（借用内置 pair 会显示成 `codex`） |
| `full_lifecycle_hook_authority` / `session_identity_only_integration` | `src/detect/mod.rs` | 官方集成的分类 |
| agent-registry TOML | `vendor/agent-registry/agents/<id>/*.toml`（未合并分支） | 配置驱动白名单的方向 |

参考：Herdr 集成文档入口见本仓库 `README.md` 底部的链接。
