# CHANGELOG

## v1.2.0 (2026-07-23)

### 修复 — Codex 斜杠命令与 Session 解析

- **新增 Codex 斜杠命令**：新建 `commands/daily-report.md`、`commands/weekly-report.md`、`commands/drpt.md`。此前 `.codex-plugin/plugin.json` 仅有 manifest、无 `commands/` 目录，导致 Codex 输入 `/` 无法唤起 daily-report 命令。命令调用形式 `/daily-report:daily-report` 等，逻辑仍由对应 SKILL.md 承载（Claude/Codex 共用单一事实源）。
- **修正 Codex session 解析规则**：经采样 `~/.codex/sessions/**/*.jsonl` 验证，原 `daily-report-rules.md` 第九节与 `skills/daily-report/SKILL.md` 步骤 3c 假设的 ThreadItem 类型（`userMessage`/`agentMessage`/`commandExecution`/`fileChange`/`mcpToolCall`）与真实结构不符，会导致提取失败。改为真实结构：
  - cwd 与日期取自 `session_meta.payload.cwd` / `session_meta.payload.timestamp`（原误以为无顶层 cwd/timestamp）
  - 用户/AI 消息取自 `response_item` payload.type=`message`（按 role 区分）
  - 命令执行取自 `response_item` payload.type=`function_call`（name=`exec_command`，`arguments` 含 `cmd`+`workdir`）
  - 文件修改取自 `patch_apply_end` / `custom_tool_call`
  - 等效 tool_calls = `function_call` + `custom_tool_call` + `patch_apply_end` 行数之和
- **修正 README Codex 安装说明**：原误写为 `~/.codex/settings.json` 与 `codex plugin add` 命令，改为真实的 `~/.codex/config.toml` 注册（`[marketplaces.*]` + `[plugins."*@*"]`），并补充 commands 目录与 `/daily-report:*` 调用形式。

### 影响

- **不破坏 Claude 侧**：Claude 走 `.claude-plugin` + skills，新增 `commands/` 不影响；3c 修正仅作用于 Codex 分支。
- **Codex 侧**：注册插件后 `/` 可唤起命令，session 提取字段准确。

---

## v1.1.0 (2026-07-22)

### 新增 — Codex 适配

- **Codex 插件清单**：新增 `.codex-plugin/plugin.json`，支持在 Codex CLI 中通过 `codex plugin add` 安装
- **Codex 数据源**：新增 `data_sources.codex_session_base` 配置项（默认 `~/.codex/sessions`），支持自动读取 Codex rollout JSONL 文件
- **双解析器并行**：Claude 和 Codex session 作为独立数据源并行采集，互不干扰
- **Codex JSONL 解析规则**：新增 `daily-report-rules.md` 第九节，详细记录 ThreadItem 类型映射、cwd 提取策略、等效 tool_calls 计算

### 改进

- **跨平台合并**：日报/周报的项目归并逻辑支持 Claude + Codex 同项目 session 合并
- **配置示例更新**：`config.example.yaml` 增加 `codex_session_base` 字段
- **README**：新增 Codex 安装指南，Phase 2 标记完成

---

## v1.0.0 (2026-07-01)

### 首次发布

- 日报生成（`/daily-report`、`/drpt`）
- 周报生成（`/weekly-report`）
- Claude Code session JSONL 数据源
- 日历日程集成
- 术语脱敏替换
- 交互式首次配置引导
- 版本升级自动迁移配置
