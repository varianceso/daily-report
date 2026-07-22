# CHANGELOG

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
