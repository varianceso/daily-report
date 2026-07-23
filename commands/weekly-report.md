---
description: 生成本周（或指定周）工作周报，从日报聚合，缺失天回退 session 扫描
argument-hint: [--w N] [--dr YYYY-MM-DD] [--fs <<CALENDAR_PLATFORM>文档URL>]
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit]
---

# 周报生成

读取 `../skills/weekly-report/SKILL.md` 并严格按其完整流程执行。

用户以参数形式调用本命令：`$ARGUMENTS`

- `--w N`：指定周数
- `--dr YYYY-MM-DD`：指定参考日期所在周（与 `--w` 互斥，同时给以 `--dr` 为准）
- `--fs <<CALENDAR_PLATFORM>文档URL>`：将周报追加到<CALENDAR_PLATFORM>文档顶部

执行要点：
1. 解析 `$ARGUMENTS` 中的参数
2. 读取 `../config.yaml` 和 `../daily-report-rules.md`
3. 按周范围逐日采集：优先读 `../reports/{YYYY-MM}/{YYYY-MM-DD}.md`，缺失回退 Claude + Codex session 扫描
4. 按 `../daily-report-rules.md` 第二节模板输出六大板块纯 Markdown，写入 `../reports/{YYYY-MM}/week-{N}.md`
