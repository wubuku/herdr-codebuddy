# herdr-codebuddy

[![test](https://github.com/Ziy1-Tan/herdr-codebuddy/actions/workflows/test.yml/badge.svg)](https://github.com/Ziy1-Tan/herdr-codebuddy/actions/workflows/test.yml)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

CodeBuddy hooks report session state to Herdr. This adds no Herdr process or screen detection.

| CodeBuddy event | Herdr report |
| --- | --- |
| `SessionStart` (`startup`, `resume`, `clear`) | `idle`, plus a separate session-identity report |
| `SessionStart` (`compact`) | `working` |
| `UserPromptSubmit` | `working` |
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure` | `working` |
| `PermissionDenied` | `working` |
| `Stop`, `StopFailure` | `idle` |
| `Notification` (`permission_prompt`) | `blocked` |
| `SessionEnd` | release the agent; `/clear` reports nothing (the paired `SessionStart` reports the new session) |

Every state report carries `--agent-session-id` and the `codebuddy --resume <id>` recovery command. `SessionStart` also sends a `pane report-agent-session` report: Herdr only persists a native `agent_session` reference for its built-in `(source, agent)` pairs, so this second report uses one (`herdr:codex`/`codex` by default) and is what lets `herdr agent list` find the pane by session ID. State and recovery stay on `custom:codebuddy`. The default pair accepts `startup`, `clear`, `resume`, and `compact`, so a relaunch in the same pane replaces a stale session ID. Set `HERDR_CODEBUDDY_SESSION_SOURCE` and `HERDR_CODEBUDDY_SESSION_AGENT` if a Herdr build lists `codebuddy` natively.

## Install

Requirements: `curl`, Node.js, and CodeBuddy.

```sh
curl -fsSL https://raw.githubusercontent.com/Ziy1-Tan/herdr-codebuddy/main/install.sh | sh
```

To uninstall:

```sh
curl -fsSL https://raw.githubusercontent.com/Ziy1-Tan/herdr-codebuddy/main/install.sh | sh -s -- uninstall
```

For a local checkout, run `node install.js install` or `node install.js uninstall`.

## Test

```sh
npm test
```

Reference: [Add Herdr support to your agent](https://herdr.dev/docs/add-herdr-support/).
