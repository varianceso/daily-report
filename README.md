# 日报/周报自动生成插件

从 Claude/Codex/OpenCode 等 AI 编码助手的 session 记录中自动提取工作内容，结合<CALENDAR_PLATFORM>日程，生成面向团队和领导审阅的标准日报和周报。

## 功能

- **日报**：扫描当天 session + git commit + <CALENDAR_PLATFORM>文档产物，按项目汇总输出"今日日程 / 今日完成 / 明日计划"，项目标题含状态标记与进度
- **周报**：聚合本周日报（缺失天回退 session 扫描），按分支拆分为独立项目，输出"质量异常 / 关键指标 / 重点事项 / 项目进展 / 其他 / 下周计划"
- **智能归并**：同项目同分支自动合并为 checklist，不同分支独立保留为不同项目标题
- **已完成项目不遗漏**：即使本周已上线，也会列在项目进展中标注"已于X.X上线"
- **<CALENDAR_PLATFORM>产物提取**：自动识别 `create-doc` / `update-doc` 调用，将文档链接作为交付物附在要点末尾
- **术语脱敏**：自动替换内部术语（AC→交叉验证、搭建 slug→项目文档目录 等），输出适合外部审阅
- **可配置周报周期**：`week_start_day` 支持按团队周会节奏自定义起始日（如周四→上周四~本周三）
- **交互式首次配置**：首次运行自动扫描项目目录，引导选择项目、命名、术语替换
- **版本升级自动迁移配置**：`plugin update` 后自动从旧版本目录迁移 `config.yaml`

## 安装

### Claude Code

```bash
# 第一步：添加插件市场
/plugin marketplace add https://github.com/varianceso/daily-report.git

# 第二步：安装插件
/plugin install daily-report@daily-report-marketplace
```

或手动配置 `~/.claude/settings.json`：

```json
{
  "pluginMarketplaces": [
    {
      "source": "git",
      "url": "https://github.com/varianceso/daily-report.git"
    }
  ],
  "enabledPlugins": {
    "daily-report@daily-report-marketplace": true
  }
}
```

### Codex / OpenCode / ZCode

同样支持 `/plugin install` 命令（后续适配阶段将实现各自的数据源适配器）。

### Codex

```bash
# 添加插件市场
codex plugin marketplace add https://github.com/varianceso/daily-report.git

# 安装插件
codex plugin add daily-report@daily-report-marketplace
```

或手动配置 `~/.codex/settings.json`：

```json
{
  "pluginMarketplaces": [
    {
      "source": "git",
      "url": "https://github.com/varianceso/daily-report.git"
    }
  ],
  "enabledPlugins": {
    "daily-report@daily-report-marketplace": true
  }
}
```

> **注意**：Codex 安装后需在 config.yaml 中配置 `data_sources.codex_session_base`（默认 `~/.codex/sessions`）才能自动读取 Codex session 数据。

## 初次配置

首次运行 `/daily-report` 或 `/weekly-report` 时，会自动进入交互式配置引导：

```
⚙️ 检测到首次使用，需要配置以下内容：
📁 扫描到 8 个项目工作目录:
1. E:\<REDACTED>\<PROJECT_A>-agent
2. E:\<REDACTED>\<DASHBOARD>
...

请选择要纳入日报统计的项目（如 "1 2"）：
```

按提示选择项目、设置显示名称、配置术语替换后，自动生成 `config.yaml`。

### 手动配置

也可直接创建 `config.yaml`：

```yaml
# 项目工作目录 → 对外显示名称
projects:
  "E:\\<REDACTED>\\<PROJECT_A>-agent": "<PROJECT_A>-agent"
  "E:\\<REDACTED>\\<DASHBOARD>": "<DASHBOARD>"

# 内部术语 → 对外表述（支持正则）
term_replacements:
  "\\bAC\\b": "交叉验证"
  "\\bARC\\b": "架构设计"
  "\\bslug\\b": "项目文档目录"
  "搭建 slug": "建立项目文档目录结构"
  "<TOOL_A>": "岗位机器人"

# session 过滤阈值
filter:
  min_session_duration_minutes: 5
  min_tool_calls: 3
  ignore_pure_chat: true

# 周报聚合策略
weekly:
  week_start_day: "thursday"     # 周四开周会 → 上周四~本周三
                                 # 默认 "monday"（周一~周日）

# <CALENDAR_PLATFORM>日程黑名单（大小写不敏感）
calendar_keyword_blacklist:
  - "午饭"
  - "午休"
  - "physiotherapy"

# 数据源路径
data_sources:
  claude_session_base: "<REDACTED>.claude/projects"
  project_prefix: "E--<REDACTED>-"
```

**配置项说明：**

| 配置 | 必填 | 说明 |
|------|------|------|
| `projects` | ✅ | JSONL 中 cwd 字段值 → 对外显示名称 |
| `term_replacements` | 否 | 内部术语替换，支持精确匹配和正则 |
| `filter` | 否 | session 过滤阈值，默认忽略 <3 tool_calls 或 <5min 的会话 |
| `weekly.week_start_day` | 否 | 周报起始日（monday~sunday），默认 monday |
| `calendar_keyword_blacklist` | 否 | 日程标题含这些词则忽略 |
| `data_sources.claude_session_base` | ✅ | Claude Code session JSONL 存储根目录 |
| `data_sources.codex_session_base` | 否 | Codex rollout JSONL 存储目录（默认 `~/.codex/sessions`），配置后才启用 Codex 数据源 |
| `data_sources.project_prefix` | ✅ | session 目录名前缀，用于扫描时过滤 |

## 使用

```bash
# 日报
/daily-report                          # 今天
/daily-report --d 2026-06-30          # 补生成
/daily-report --d 2026-06-30 --fs https://<REDACTED>/docx/xxx  # 推送<CALENDAR_PLATFORM>

# 周报
/weekly-report                         # 本周
/weekly-report --dr 2026-06-30        # 指定日期所在周
/weekly-report --w 27                 # 指定周数
```

### 参数缩写

| 参数 | 缩写 | 用途 |
|------|------|------|
| `--date` | `--d` | 日报日期 |
| `--week` | `--w` | 周报周数 |
| `--date-ref` | `--dr` | 周报参考日期 |
| `--<CALENDAR>` | `--fs` | <CALENDAR_PLATFORM>文档 URL |

## 配置

`config.yaml` 可配置项：

| 配置区 | 说明 |
|--------|------|
| `projects` | 项目路径 → 对外显示名称映射 |
| `term_replacements` | 内部术语 → 对外表述（先精确再正则） |
| `filter` | session 过滤阈值 |
| `weekly` | 周报聚合策略（含 `week_start_day`） |
| `calendar_keyword_blacklist` | 日程标题黑名单 |
| `data_sources` | Claude session 存储路径 |

完整规则见 `daily-report-rules.md`。

## 文件结构

```
daily-report/
├── .claude-plugin/
│   ├── plugin.json              # 插件清单
│   └── marketplace.json         # 插件市场清单
├── skills/
│   ├── daily-report/SKILL.md    # 日报 Skill
│   └── weekly-report/SKILL.md   # 周报 Skill
├── config.example.yaml           # 配置模板
├── daily-report-rules.md         # 处理规则
└── reports/                      # 输出存档（gitignore）
```

## 扩展计划

| 阶段 | 内容 |
|------|------|
| Phase 1 ✅ | Claude 数据源 + 日报/周报 |
| Phase 2 ✅ | Codex 数据源适配器 |
| Phase 3 🔲 | OpenCode 数据源适配器 |
| Phase 4 🔲 | ZCode 数据源适配器 |
| Phase 5 🔲 | 多人配置模板 |
