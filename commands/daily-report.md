---
description: 生成当天（或指定日期）工作日报，从 Claude/Codex session + git commit 提取，按项目汇总
argument-hint: [--d YYYY-MM-DD] [--fs <<CALENDAR_PLATFORM>文档URL>]
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit]
---

# 日报生成

读取 `../skills/daily-report/SKILL.md` 并严格按其完整流程执行。

用户以参数形式调用本命令：`$ARGUMENTS`

- `--d YYYY-MM-DD`：指定日期（默认当天）
- `--fs <<CALENDAR_PLATFORM>文档URL>`：将日报追加到<CALENDAR_PLATFORM>文档顶部

执行要点：
1. 解析 `$ARGUMENTS` 中的参数
2. 读取 `../config.yaml`（不存在则走交互式首次配置，见 SKILL.md 步骤 2）
3. 双数据源并行采集：Claude session JSONL + Codex rollout JSONL（若已配置 `data_sources.codex_session_base`）+ git commit
4. 按 `../daily-report-rules.md` 第一节模板输出纯 Markdown，并写入 `../reports/{YYYY-MM}/{YYYY-MM-DD}.md`
