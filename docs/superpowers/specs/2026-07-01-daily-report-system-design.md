# 日报/周报自动生成系统 — 设计文档

> Generated: 2026-07-01 | Status: 待用户 review

---

## 1. 目标与范围

### 1.1 目标

构建一套 Claude Code Skill 插件，支持从多个 AI 编码助手（Claude、Codex、OpenCode、ZCode 等）的会话记录和缓存中自动提取当天工作内容，结合<CALENDAR_PLATFORM>日程，生成面向领导和同事审阅的日报和周报。

### 1.2 核心原则

- **按项目汇总**，禁止流水账（同一项目多轮 session 合并为要点）
- **内部术语过滤**（AC → 交叉验证、ARC → 架构设计等）
- **可指定历史日期**（支持补生成某天的日报）
- **双输出通道**：终端直接输出 + 可选追加到<CALENDAR_PLATFORM>文档

---

## 2. 架构设计

### 2.1 整体结构

```
daily-report/                              # 本仓库（Claude Code 插件格式）
├── .claude/
│   ├── skills/
│   │   ├── daily-report/
│   │   │   └── SKILL.md                # 日报 Skill 定义
│   │   └── weekly-report/
│   │       └── SKILL.md                # 周报 Skill 定义
│   └── rules/
│       ├── daily-report-rules.md        # 汇总规则、术语表、过滤策略
│       └── config.yaml                  # 项目映射、术语替换表、日程过滤黑名单
├── .gitignore                           # 忽略 reports/ 目录
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-07-01-daily-report-system-design.md  # 本设计文档
└── reports/                             # 生成的日报/周报存档（gitignore）
    └── 2026-07/
        ├── 2026-07-01.md
        └── week-27.md
```

### 2.2 数据采集抽象层

为后续适配多 Agent（Codex、OpenCode、ZCode），数据采集设计为适配器模式：

```
DataSource 接口:
  collect(date) → [{project, summary, tasks, metrics, ...}]
```

| 适配器 | 数据源 | 初期状态 |
|--------|--------|----------|
| `ClaudeAdapter` | `~/.claude/projects/E--<REDACTED>-*/**.jsonl` | ✅ 首批实现 |
| `CodexAdapter` | `~/.codex/` 对应目录 | 🔲 后续扩展 |
| `OpenCodeAdapter` | `~/.opencode/` 对应目录 | 🔲 后续扩展 |
| `ZCodeAdapter` | 待调研 | 🔲 后续扩展 |

每个适配器输出统一的 `DailyWorkItem[]` 结构，上层汇总逻辑与具体 Agent 解耦。

### 2.3 核心流程

```
触发 /daily-report [--date YYYY-MM-DD] [--<CALENDAR> <doc_url>]
  │
  ├── 1. 数据采集
  │   ├── ClaudeAdapter.collect(date) → 扫描所有项目 session JSONL
  │   └── <CALENDAR_API>.get_events(date) → 获取当日日程
  │
  ├── 2. 智能过滤
  │   ├── 丢弃 session 时长 < 5min 或 tool_calls < 3
  │   ├── 日程黑名单过滤（午休 / sports / physiotherapy 等）
  │   └── 去重（同项目同任务只保留最新）
  │
  ├── 3. 归并 & 术语替换
  │   ├── 同项目多轮 session → 合并为 2-4 条要点
  │   └── AC→交叉验证, ARC→架构设计 等
  │
  ├── 4. 输出
  │   ├── 终端：按模板渲染 Markdown
  │   ├── 本地存档：写入 reports/YYYY-MM/YYYY-MM-DD.md
  │   └── --<CALENDAR>: 通过<CALENDAR_PLATFORM> MCP 插入到指定文档最顶部（最新在前）
  │
  └── 5. 完成
```

周报流程类似，额外包含"本周日报聚合 → 回退 session 扫描补全"的兜底逻辑。

---

## 3. Skill 定义

### 3.1 daily-report

- **触发命令**：`/daily-report [--date YYYY-MM-DD] [--<CALENDAR> <doc_url>]`
- **默认行为**：生成当天日报，输出到终端 + 存档到 `reports/`
- **参数**：
  - `--date`：指定日期（默认当天）
  - `--<CALENDAR>`：<CALENDAR_PLATFORM>文档 URL，生成后追加到该文档头部

### 3.2 weekly-report

- **触发命令**：`/weekly-report [--week N] [--date-ref YYYY-MM-DD] [--<CALENDAR> <doc_url>]`
- **默认行为**：生成本周周报
- **参数**：
  - `--week`：指定第几周
  - `--date-ref`：以该日期所在周为准
  - `--<CALENDAR>`：同上

---

## 4. 输出模板

### 4.1 日报模板

```markdown
# 日报 — YYYY-MM-DD（周X）

## 📅 今日日程
| 时间 | 事项 |
|------|------|
| 10:00-11:00 | XXX周会 |
| 14:00-15:00 | XXX技术评审 |

## ✅ 今日完成

### <项目A>
- 要点1
- 要点2

### <项目B>
- 要点1

### 其他
- 交叉验证：完成 xxx v1.3 QA 回归，N 个问题已修复

## 📋 明日计划
1. 项目A: xxxx
2. 项目B: xxxx
```

### 4.2 周报模板

基于实际工作周报习惯，完整模板如下：

```markdown
# 本周工作 — 第N周（M/D-M/D）

### 质量异常
- 无 / <具体异常描述>

### 关键指标
- 本周上线 X 次
- 提测 X 次
- Bug 修复 X 个，新增 X 个
- Code Review 覆盖 X 个 PR

### 重点事项
**本周上线内容-今晚X.X上线**
1. XXXX
2. XXXX

### 项目进展
**[g]项目A，开发进度80%，计划X.X上线 @协作人**
- 产品方案：<链接>
- 技术方案：<链接>
- [x] 已完成项
- [ ] 待完成项

**[g]项目B，开发进度50%->70%，计划X.X提测，X.X上线**
- 产品方案：<链接>
- 技术方案：<链接>
- 任务拆解如下
- [x] XXXX
- [ ] XXXX

**[y]项目C，技术方案中，计划X.X上线**
- 产品方案：<链接>
- 技术方案：待补充

### 其他 -- 运维问题
- [问题看板名称-Humi](链接)
- 用户反馈问题总数：X 个，处理中 X 个

### 下周计划
1. xxxx
2. xxxx
```

**状态标记规则**：
| 标记 | 含义 | 自动推断条件 |
|------|------|-------------|
| `[g]` | 绿色正常 | 进度 >= 计划进度 |
| `[y]` | 黄色需关注 | 进度 < 计划或存在风险 |
| `[r]` | 红色阻塞 | 存在明确阻塞项 |

---

## 5. 配置文件

### 5.1 `config.yaml`

```yaml
# 项目路径 → 对外显示名称
projects:
  "E--<REDACTED>-<PROJECT_A>-agent": "<PROJECT_A>-agent"
  "E--<REDACTED>-<DASHBOARD>": "<DASHBOARD>"
  "E--<REDACTED>-multi-reviewer": "三方协同（multi-reviewer）"
  "E--<REDACTED>-<PROJECT_C>": "<PROJECT_C>"
  "E--<REDACTED>-coder-delegation": "架构设计"
  "E--<REDACTED>-<PROJECT_B>": "<PROJECT_B>"
  "E--<REDACTED>-tri-agent-coding": "三方协同"
  "E--<REDACTED>-codingagent": "codingagent"
  "E--<REDACTED>-data-analysis": "数据分析"
  "E--<REDACTED>-<PROJECT_A>-<DASHBOARD>-web": "<PROJECT_A>-<DASHBOARD>-web"
  "E--<REDACTED>-2026H1": "2026H1"

# 内部术语 → 对外表述
term_replacements:
  "AC": "交叉验证"
  "ARC": "架构设计"
  "\\bAC\\b": "交叉验证"
  "\\bARC\\b": "架构设计"

# 智能过滤
filter:
  min_session_duration_minutes: 5
  min_tool_calls: 3
  ignore_pure_chat: true        # 纯对话无 tool call 的 session 忽略

# <CALENDAR_PLATFORM>日程过滤（标题含以下关键词的日程忽略）
calendar_keyword_blacklist:
  - "午饭"
  - "午休"
  - "通勤"
  - "sports and relaxed"
  - "Released and Prepared"
  - "physiotherapy"
  - "Sports and relaxed"
  - "Released and prepared"

# 周报聚合
weekly:
  use_daily_reports_first: true  # 优先使用日报文件聚合
  fallback_to_session_scan: true # 缺日报时回退扫 session
```

---

## 6. 数据处理规则

### 6.1 Session → WorkItem 提取规则

从 JSONL 中提取的关键字段：
- `type: "user"` → 用户输入的任务描述
- `type: "assistant"` → AI 的实际操作（tool calls 类型和参数）
- `cwd` → 所在项目目录
- `timestamp` → 时间
- `toolUseResult` → 操作结果

**提取策略**（由 Claude 在 Skill 执行时自行理解，而非脚本硬解析）：
1. 识别 session 所属项目（根据 `cwd` 匹配 `config.yaml` 项目映射）
2. 提取用户的核心任务意图（只看 user 消息的任务描述部分）
3. 提取实质性产出（有 tool call + 成功结果的操作为"有效工作"）
4. 过滤噪音（系统消息、skill 加载、hook 元数据等）

### 6.2 项目归并规则

同一天同一项目下多个 session → 合并规则：
- 同任务拆分成多轮 session → 合并为一条，用最新状态
- 不同独立任务 → 各保留一条要点
- 超过 4 条要点 → 按重要性截断，次要归入"其他"
- 不与当天其他 session 冲突的独立任务 → 优先保留

### 6.3 术语替换规则

- 先精确匹配（配置中的 key），再正则匹配
- 替换发生在终端输出前、<CALENDAR_PLATFORM>推送前
- 不修改源数据（JSONL 原样保留）

---

## 7. 错误处理

| 场景 | 处理 |
|------|------|
| 指定日期无数据 | 提示"YYYY-MM-DD 无工作记录"，不生成空日报 |
| JSONL 文件过大(>500KB) | 先读尾部 N 条消息，不够再向前翻 |
| <CALENDAR_PLATFORM>日程 API 失败 | 日程区显示"（日历获取失败）"，其他部分正常输出 |
| `--<CALENDAR>` 追加失败 | 终端显示错误原因，本地存档已正常保存 |
| 周报某天缺日报文件 | 自动回退到 session 扫描补全 |
| 某项目 session 解析异常 | 跳过该 session，记录 warning，继续处理其他 |

---

## 8. 扩展计划

| 阶段 | 内容 |
|------|------|
| Phase 1 | Claude 数据源 + 日报/周报 Skill + <CALENDAR_PLATFORM>输出 |
| Phase 2 | Codex 数据源适配器 |
| Phase 3 | OpenCode 数据源适配器 |
| Phase 4 | ZCode 数据源适配器 |
| Phase 5 | 支持多人使用（配置文件个人化，可 override 项目列表和术语表） |

---

## 9. gitignore

```gitignore
# 设计文档目录
docs/

# 生成的日报/周报存档
reports/

# 个人本地配置覆盖
.claude/rules/config.local.yaml
```

---

## <REDACTED> 待确认项

- [ ] 是否有其他内部术语需要加入替换表？
- [ ] Codex / OpenCode / ZCode 的 session 文件存储路径格式确认？
- [ ] <CALENDAR_PLATFORM>日程是否需要只取"工作相关"日历（排除个人日历）？
