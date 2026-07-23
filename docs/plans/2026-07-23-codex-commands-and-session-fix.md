# Codex 斜杠命令修复 + Session 解析规则修正

> 2026-07-23 | 修复 Codex 中 `/` 唤不起 daily-report 命令，并修正 Codex session JSONL 解析规则

## 背景

用户反馈：在 Codex 输入 `/` 不会唤起 daily-report 相关命令。同时需确认 Claude 能否扫描 Codex session 做总结。

经采样 `~/.codex/sessions/**/*.jsonl` 与已安装的 `cloudflare` 插件，定位到两类问题：

### 问题一：Codex 斜杠命令缺失（根因两层）

1. **缺少 `commands/` 目录**：Codex 斜杠命令来自插件根目录 `commands/*.md`（frontmatter 格式，命令名 `/<插件名>:<命令名>`）。当前 `.codex-plugin/plugin.json` 只有 manifest，`interface.defaultPrompt` 仅是安装示例，不注册命令。
2. **插件未在 Codex 注册**：`~/.codex/config.toml` 无 daily-report 的 marketplace/plugin 条目，Codex 未加载本插件。

### 问题二：Codex session 解析规则错误

`skills/daily-report/SKILL.md` 步骤 3c 与 `daily-report-rules.md` 第八节假设的 ThreadItem 类型与真实结构不符：

| SKILL.md 假设（错误） | Codex 真实结构（已验证） |
|---|---|
| 无顶层 timestamp / cwd | `session_meta.payload` 含 `timestamp` + `cwd` |
| `userMessage` / `agentMessage` | `response_item` payload.type=`message`，按 `role`(user/assistant) 区分；另有 `event_msg` 的 `user_message`/`agent_message` |
| `commandExecution`（含 cwd） | `response_item` payload.type=`function_call`，`name=exec_command`，`arguments` 含 `cmd` + `workdir` |
| `fileChange` | `response_item` payload.type=`patch_apply_end` / `custom_tool_call` |
| `mcpToolCall` | `function_call`（mcp 工具名）或 `custom_tool_call` |
| 文件名判断日期 | `session_meta.payload.timestamp` 有 ISO 时间，文件名亦含日期 |

> 结论：Claude **可以**扫描 Codex session 做总结（纯 JSONL 可读），但需先修正解析规则，否则提取失败。

## 改动清单

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 1 | `commands/daily-report.md` | **新建** | Codex 斜杠命令，调用 `skills/daily-report/SKILL.md` |
| 2 | `commands/weekly-report.md` | **新建** | Codex 斜杠命令，调用 `skills/weekly-report/SKILL.md` |
| 3 | `commands/drpt.md` | **新建** | daily-report 短别名命令 |
| 4 | `skills/daily-report/SKILL.md` | 修改 | 步骤 3c 改用真实 Codex 结构（session_meta 取 cwd/timestamp，response_item 解析 message/function_call/patch_apply_end） |
| 5 | `skills/weekly-report/SKILL.md` | 修改 | Codex 回退路径同步修正 |
| 6 | `daily-report-rules.md` | 修改 | 第八节 ThreadItem 类型映射表改为真实结构 |
| 7 | `README.md` | 修改 | Codex 安装说明：commands 目录 + config.toml 注册步骤 |
| 8 | `docs/CHANGELOG.md` | 修改 | 记录 v1.2.0 变更 |

## 各文件改动详情

### 1-3. `commands/*.md`（新建）

参照 cloudflare 插件 `commands/build-agent.md` 的 frontmatter 格式：

```markdown
---
description: 生成当天（或指定日期）工作日报
argument-hint: [--d YYYY-MM-DD] [--fs <文档URL>]
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit]
---

# 日报生成

读取 `../skills/daily-report/SKILL.md` 并严格按其流程执行。
用户参数：$ARGUMENTS
```

- 命令调用形式：`/daily-report:daily-report`、`/daily-report:weekly-report`、`/daily-report:drpt`
- 命令体保持精简，真正逻辑仍由 SKILL.md 承载（单一事实源，Claude/Codex 共用）

### 4. `skills/daily-report/SKILL.md` 步骤 3c 重写

修正为真实结构：

```
### 3c. Codex Session JSONL（当 data_sources.codex_session_base 已配置）

扫描 codex_session_base 下 YYYY/MM/DD/rollout-*.jsonl，按 session_meta.payload.timestamp
日期（或文件名日期）筛选目标日期。

Codex JSONL 真实结构（每行一个顶层事件，含 timestamp + type）：

| 顶层 type | payload.type | 提取内容 | 用途 |
|-----------|--------------|----------|------|
| session_meta | — | payload.cwd, payload.timestamp | 项目归属 + 日期 |
| response_item | message (role=user) | content[].text | 用户任务描述 |
| response_item | message (role=assistant) | content[].text | AI 产出内容 |
| response_item | function_call (name=exec_command) | arguments.cmd + arguments.workdir | 命令执行 + 项目归属 |
| response_item | function_call (其他 name) | name + arguments | MCP/自定义工具调用 |
| response_item | patch_apply_end | changes/patch | 文件修改记录 |
| response_item | custom_tool_call | name + arguments | 自定义工具（含 apply_patch） |
| event_msg | user_message / agent_message | message/text | 备用消息来源 |

关键修正：
- cwd 优先取 session_meta.payload.cwd；function_call.exec_command 的 workdir 更细粒度
- timestamp 直接取 session_meta.payload.timestamp（ISO），无需依赖文件名
- 等效 tool_calls = function_call + custom_tool_call + patch_apply_end 行数之和
- reasoning / token_count / context_compacted 行忽略
```

同步修正步骤 5（提取工作内容）中 Codex 分支的字段引用。

### 5. `skills/weekly-report/SKILL.md`

Codex 回退路径说明同步指向上述真实结构。

### 6. `daily-report-rules.md` 第八节

ThreadItem 类型映射表整体替换为上表，更新 cwd 提取策略与项目归属逻辑说明。

### 7. `README.md`

Codex 安装说明补充：
- 插件需含 `commands/` 目录（已补齐）
- 在 `~/.codex/config.toml` 注册 marketplace + 启用 plugin 的示例（路径占位，不含敏感信息）
- 命令调用形式 `/daily-report:daily-report` 等

### 8. `docs/CHANGELOG.md`

新增 v1.2.0：Codex 斜杠命令支持 + Session 解析规则修正。

## 影响范围

- **不破坏 Claude 侧**：Claude 走 `.claude-plugin` + skills，新增 `commands/` 不影响；SKILL.md 3c 修正仅影响 Codex 分支
- **Codex 侧**：补齐命令 + 修正解析后，`/daily-report:*` 可用，session 提取准确
- **总改动量**：8 文件，~180 行

## 不做的事

- 不自动修改用户 `~/.codex/config.toml`（部署/注册由用户执行，符合 §0.4 agent 不部署红线）
- 不改 Claude 侧解析逻辑
