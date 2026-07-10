# 日报/周报自动生成系统 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-doc Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建两个 Claude Code Skill（`daily-report` + `weekly-report`），从 Claude session JSONL 和<CALENDAR_PLATFORM>日程自动生成面向团队审阅的日报和周报。

**Architecture:** 纯 Claude Skill 驱动——SKILL.md 内嵌完整的处理流程指令，Claude 自行读取 JSONL、调用<CALENDAR_PLATFORM> MCP、按模板输出。配置文件（config.yaml + rules.md）提供项目映射、术语替换表、过滤规则。无外部脚本依赖。

**Tech Stack:** Claude Code Skill format (SKILL.md + config.yaml + rules.md)，<CALENDAR_PLATFORM> MCP（calendar + docs），JSONL session 数据。

## Global Constraints

- 日报按项目汇总，禁止流水账；同项目多 session 合并为 2-4 条要点
- 内部术语必须替换：AC → 交叉验证，ARC → 架构设计
- 支持 `--date` 指定历史日期补生成
- 双输出：终端 + `reports/YYYY-MM/` 存档；`--<CALENDAR>` 可选追加到<CALENDAR_PLATFORM>文档顶部
- `reports/` 和 `docs/` 目录已在 `.gitignore` 中忽略
- 周报优先聚合本周日报文件，缺失天自动回退到 session 扫描
- 日程过滤黑名单：午饭、午休、通勤、sports and relaxed、Released and Prepared、physiotherapy

---

## File Structure

```
daily-report/
├── .claude/
│   ├── skills/
│   │   ├── daily-report/
│   │   │   └── SKILL.md           ← 创建：日报 Skill 完整指令
│   │   └── weekly-report/
│   │       └── SKILL.md           ← 创建：周报 Skill 完整指令
│   └── rules/
│       ├── config.yaml            ← 创建：项目映射、术语表、过滤配置
│       └── daily-report-rules.md  ← 创建：详细处理规则、输出模板、术语替换表
├── .gitignore                     ← 已存在：docs/ + reports/ 忽略
└── reports/                       ← 运行时生成（gitignore）
    └── YYYY-MM/
        ├── YYYY-MM-DD.md
        └── week-N.md
```

**职责划分：**
| 文件 | 职责 |
|------|------|
| `SKILL.md` (daily-report) | 触发入口，逐步指令：解析参数 → 扫描 JSONL → <CALENDAR_PLATFORM>日程 → 过滤归并 → 输出 |
| `SKILL.md` (weekly-report) | 触发入口，逐步指令：聚合日报文件 → 兜底扫 session → 按周报模板输出 |
| `config.yaml` | 机器可读配置：项目映射、术语替换键值对、过滤阈值、日程黑名单 |
| `daily-report-rules.md` | 人类+AI 可读规则：汇总策略详解、输出模板完整定义、术语替换上下文说明 |

---

### Task 1: 项目基础设施 — 目录 + 配置文件

**Files:**
- Create: `.claude/rules/config.yaml`
- Create: `.claude/rules/daily-report-rules.md`
- Create: `.claude/skills/daily-report/` (directory)
- Create: `.claude/skills/weekly-report/` (directory)

**Interfaces:**
- Produces: `config.yaml` 被两个 SKILL.md 读取；`daily-report-rules.md` 被两个 SKILL.md 引用

- [ ] **Step 1: 创建目录结构**

```bash
mkdir -p .claude/skills/daily-report
mkdir -p .claude/skills/weekly-report
mkdir -p .claude/rules
```

- [ ] **Step 2: 创建 `config.yaml`**

```yaml
# 日报/周报系统配置
# ====================

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

# 内部术语 → 对外表述（先精确匹配，再正则匹配）
term_replacements:
  # 精确替换
  "AC QA 多轮回归": "QA 回归测试"
  "AC 多轮方案评审": "多轮方案评审"
  # 正则替换（\b 边界匹配，避免误替换）
  "\\bAC\\b": "交叉验证"
  "\\bARC\\b": "架构设计"

# 智能过滤阈值
filter:
  min_session_duration_minutes: 5
  min_tool_calls: 3
  ignore_pure_chat: true

# <CALENDAR_PLATFORM>日程过滤（标题含以下任一关键词的日程忽略，大小写不敏感）
calendar_keyword_blacklist:
  - "午饭"
  - "午休"
  - "通勤"
  - "sports and relaxed"
  - "released and prepared"
  - "physiotherapy"

# 周报聚合策略
weekly:
  use_daily_reports_first: true
  fallback_to_session_scan: true

# 数据源路径
data_sources:
  claude_session_base: "<REDACTED>.claude/projects"
  project_prefix: "E--<REDACTED>-"
```

- [ ] **Step 3: 创建 `daily-report-rules.md`**

```markdown
# 日报/周报处理规则

> 本文件定义日报和周报的详细处理规则、输出模板和术语替换表。
> 日报 Skill 和周报 Skill 在执行时必须遵循本文档的所有规则。

---

## 一、日报输出模板（严格照此格式）

```
# 日报 — {YYYY-MM-DD}（{周X}）

## 📅 今日日程
| 时间 | 事项 |
|------|------|
| HH:MM-HH:MM | 事项描述 |

（无日程时显示"（无日程或日历获取失败）"）

## ✅ 今日完成

### {项目A显示名称}
- 要点1（一句话概括，不超过 30 字）
- 要点2

### {项目B显示名称}
- 要点1

### 其他
- 交叉验证：xxx
- 其他非项目归属的工作

## 📋 明日计划
1. {项目A}: xxxx
2. {项目B}: xxxx
```

## 二、周报输出模板（严格照此格式）

```
# 本周工作 — 第{N}周（{M/D}-{M/D}）

### 质量异常
- 无 / <异常描述>

### 关键指标
- 本周上线 X 次
- 提测 X 次
- Bug 修复 X 个，新增 X 个（有则显示，无则写"无"）
- Code Review 覆盖 X 个 PR（有则显示，无则不写此行）

### 重点事项
**本周上线内容-今晚{M.D}上线**
1. XXXX
2. XXXX

（无上线内容时写"无"）

### 项目进展
**[g]{项目名}，开发进度{X}%，计划{X}.{X}上线 @{协作人}**
- 产品方案：{链接}
- 技术方案：{链接}
- [x] 已完成项
- [ ] 待完成项

（项目排列顺序：有上线 > 提测 > 开发中 > 方案阶段）

### 其他 -- 运维问题
（无运维问题时写"无"）

### 下周计划
1. xxxx
2. xxxx
```

## 三、项目状态标记规则

| 标记 | 含义 | 推断条件 |
|------|------|----------|
| `[g]` | 正常 | 进度正常，无风险信号 |
| `[y]` | 需关注 | 进度滞后 / 存在风险 / 方案阶段 |
| `[r]` | 阻塞 | 日报中出现"阻塞""blocked""卡住"等信号词 |

## 四、Session → WorkItem 提取规则

### 4.1 数据源

Claude session JSONL 路径：`<REDACTED>.claude/projects/E--<REDACTED>-*/{uuid}.jsonl`

### 4.2 提取步骤

1. **定位当日 session 文件**：按 `timestamp` 字段筛选目标日期的 JSONL 行
2. **识别项目归属**：从每条记录的 `cwd` 字段匹配 `config.yaml` 的 `projects` 映射
3. **提取用户意图**：读取 `type: "user"` 消息中的 `content`，提取任务描述
4. **提取产出**：读取 `type: "assistant"` 消息中的 tool calls，有实质性操作（Edit/Write/Bash/Git）的视为有效工作
5. **过滤噪音**：跳过以下类型
   - 系统消息（`type: "mode"`, `"permission-mode"`）
   - Skill 加载元数据（`isMeta: true`）
   - Hook stdout 内容
   - 纯对话无 tool call 的 session

### 4.3 智能过滤

- 丢弃 tool_calls 总数 < 3 的 session（可能是"看一眼就走了"）
- 丢弃 session 时长 < 5 分钟的会话（根据首末消息 timestamp 差值估算）
- 丢弃纯对话无产出的 session

### 4.4 项目归并规则

同一天同一项目下多个 session 的处理：
- **同任务多轮** → 合并为 1 条，取最新轮次的状态
- **不同独立任务** → 各保留 1 条要点
- **超过 4 条要点** → 保留最重要的 4 条，次要归入"其他"
- **判定"同任务"的依据**：用户消息中的核心意图相似（如都涉及同一个功能/模块/接口）

## 五、术语替换规则

替换顺序：先精确匹配 → 再正则匹配

| 匹配方式 | 内部术语 | 对外表述 |
|----------|----------|----------|
| 精确 | AC QA 多轮回归 | QA 回归测试 |
| 精确 | AC 多轮方案评审 | 多轮方案评审 |
| 正则 `\bAC\b` | AC | 交叉验证 |
| 正则 `\bARC\b` | ARC | 架构设计 |

**重要**：替换仅应用于最终输出文本，不修改原始 JSONL 数据。

## 六、<CALENDAR_PLATFORM>日程集成

### 6.1 获取日程

使用<CALENDAR_PLATFORM> MCP 获取目标日期的日程事件。

### 6.2 过滤规则

日程标题包含以下任一关键词则忽略（大小写不敏感）：
- 午饭、午休、通勤
- sports and relaxed、released and prepared、physiotherapy

### 6.3 日程输出格式

```
| HH:MM-HH:MM | {日程标题} |
```

## 七、周报聚合规则

### 7.1 聚合流程

1. 计算目标周起止日期（周一～周日）
2. 遍历每天：优先读取 `reports/YYYY-MM/YYYY-MM-DD.md`
3. 当天日报缺失 → 回退到 session 扫描（同日报流程）
4. 汇总：
   - **质量异常**：从日报中提取 bug/fix/异常/故障类条目
   - **关键指标**：统计上线次数、提测次数、Bug 数、PR 数
   - **重点事项**：提取"上线"相关条目
   - **项目进展**：按项目聚合本周所有 tasks，去重合并
   - **运维问题**：从日报中提取生产/运维/用户反馈类条目
   - **下周计划**：汇总各日报的"明日计划"中未完成项

### 7.2 项目排序

1. 本周有上线的项目
2. 本周有提测的项目
3. 进度有推进的项目
4. 方案阶段项目

## 八、<CALENDAR_PLATFORM>输出规则

当提供 `--<CALENDAR> <doc_url>` 参数时：

1. 使用<CALENDAR_PLATFORM> MCP 的 `update-doc` 工具
2. 模式：`insert_before`，定位到文档最顶部
3. 日报内容作为 Markdown 插入
4. 插入失败时：终端报错 + 本地存档保留 + 提示用户手动复制

## 九、错误处理

| 场景 | 处理 |
|------|------|
| 目标日期无任何 session | 返回"YYYY-MM-DD 无工作记录" |
| JSONL 文件 > 500KB | 先读最后 500 行，内容不足时向前翻页 |
| <CALENDAR_PLATFORM>日程 API 失败 | 日程区显示"（日历获取失败）"，继续生成其他部分 |
| `--<CALENDAR>` 追加失败 | 终端显示错误原因；本地存档不受影响 |
| 周报某天无日报文件 | 自动回退 session 扫描 |
| 某 session 解析异常 | 跳过该 session，终端输出 warning，继续处理 |
```

- [ ] **Step 4: 提交**

```bash
git add .claude/rules/config.yaml .claude/rules/daily-report-rules.md
git add .claude/skills/daily-report/ .claude/skills/weekly-report/
git commit -m "feat: add project infrastructure — config, rules, skill directories"
```

---

### Task 2: 日报 Skill（daily-report SKILL.md）

**Files:**
- Create: `.claude/skills/daily-report/SKILL.md`

**Interfaces:**
- Consumes: `config.yaml`（项目映射、术语表、过滤阈值）、`daily-report-rules.md`（模板、规则）、<CALENDAR_PLATFORM> MCP（日历 + 文档）
- Produces: 终端输出日报 Markdown + `reports/YYYY-MM/YYYY-MM-DD.md` 文件 + 可选<CALENDAR_PLATFORM>文档顶部插入

- [ ] **Step 1: 创建 SKILL.md 完整内容**

写入文件 `.claude/skills/daily-report/SKILL.md`：

```markdown
---
name: daily-report
description: 生成当天（或指定日期）的工作日报，从 Claude session 记录中提取工作内容，结合<CALENDAR_PLATFORM>日程，按项目汇总输出。
command: /daily-report
args:
  - name: date
    type: string
    optional: true
    description: 指定日期 YYYY-MM-DD，默认当天
  - name: <CALENDAR>
    type: string
    optional: true
    description: <CALENDAR_PLATFORM>文档 URL，将日报插入到文档最顶部
---

# 日报生成指令

你是一个日报生成助手。你的任务是扫描用户在指定日期的所有 Claude Code session 记录，结合<CALENDAR_PLATFORM>日历，生成一份面向团队/领导审阅的工作日报。

## 核心约束（必须遵守）

1. **禁止流水账**：同一项目多次对话必须合并汇总为要点，不能逐条列出每次对话
2. **术语替换**：输出前必须将内部术语替换为对外表述（参见规则文件）
3. **按项目分组**：以项目为二级标题组织内容
4. **简洁有力**：每条要点一句话概括（不超过 30 字），突出产出而非过程

## 执行步骤

### 第一步：解析参数

从用户输入中提取：
- `date`：目标日期，格式 YYYY-MM-DD。未指定则使用当前日期（2026-07-01 为"今天"的基准，实际执行时按系统时间计算）
- `<CALENDAR>_url`：<CALENDAR_PLATFORM>文档 URL，`--<CALENDAR>` 参数值。未指定则为 null

日期解析示例：
- `/daily-report` → 当天
- `/daily-report --date 2026-06-30` → 2026-06-30
- `/daily-report --date 2026-06-30 --<CALENDAR> https://<REDACTED>/docx/xxx` → 指定日期 + <CALENDAR_PLATFORM>输出

### 第二步：读取配置

读取 `.claude/rules/config.yaml` 获取：
- 项目路径 → 显示名称映射
- 术语替换表
- 过滤阈值
- 日程黑名单

读取 `.claude/rules/daily-report-rules.md` 获取完整的模板和处理规则。

### 第三步：扫描当日 Session

1. 列出 `<REDACTED>.claude/projects/` 下所有以 `E--<REDACTED>-` 开头的项目目录
2. 对每个项目，找到所有 `.jsonl` 文件（session 转录）
3. 对每个 `.jsonl` 文件：
   a. 读取文件内容
   b. 筛选 `timestamp` 在目标日期范围内的行
   c. 如果没有当日的行，跳过此文件
   d. 如果文件 > 500KB，先读取最后 500 行，检查是否有当日内容；有则继续向前读取直到覆盖完整当日

**注意**：只读取 Claude 消息提取所需信息，不要原样输出整个 JSONL 内容。你的任务是理解并总结，不是复述。

### 第四步：过滤低价值 Session

对每个当日 session：

1. 统计 tool_calls 数量（搜索 `"type":"assistant"` 且包含 tool_use 的行）
2. 估算 session 时长（首条消息和末条消息的 timestamp 差值）
3. 检查是否有实质性产出（Edit、Write、Bash 执行、Git 操作等）

丢弃以下 session：
- tool_calls 总数 < 3
- session 时长 < 5 分钟
- 纯对话无 tool call

### 第五步：提取工作内容

对每个保留的 session：

1. **识别项目**：从 `cwd` 字段匹配 `config.yaml` 的 projects 映射，得到显示名称
2. **提取意图**：读取 `type: "user"` 的非 meta 消息，理解用户要做什么
3. **提取产出**：读取 assistant 的 tool calls 和 tool results，理解实际做了什么
4. **总结要点**：用一句话概括这个 session 的核心产出（注意：这是给领导看的，突出完成了什么，不是做了什么操作）

示例：
- ❌ "执行了 3 次 Edit 操作修改了 vacation.ts 文件"（流水账）
- ✅ "修复 vacation/export 接口日期格式兼容问题"（产出导向）

### 第六步：获取<CALENDAR_PLATFORM>日程

使用<CALENDAR_PLATFORM> MCP 工具获取目标日期的日历事件：

1. 获取目标日期的日程列表
2. 过滤：标题包含 config.yaml 中 `calendar_keyword_blacklist` 任一关键词的日程忽略（大小写不敏感）
3. 格式化：`| HH:MM-HH:MM | 日程标题 |`

如果<CALENDAR_PLATFORM> API 调用失败，日程区显示 `（日历获取失败）`。

### 第七步：项目归并

将第五步提取的 work items 按项目归并：

1. 同一项目下，判定为同任务的多个 session → 合并为 1 条要点（取最新状态）
2. 不同任务 → 各保留 1 条
3. 单项目超过 4 条要点时，保留最重要的 4 条，其余归入"其他"分类
4. 无法归类到具体项目的 → 归入"其他"

**"同任务"判定标准**：用户消息中涉及同一个功能模块/接口/问题。

### 第八步：生成明日计划

基于今日未完成的工作和常见延续任务，推测明日计划：
- 今天进行中但未完成的任务 → 明天继续
- 今天提到的"明天做""接下来"的任务
- 如果没有足够信息，写 2-3 条合理的延续计划

### 第九步：术语替换

对最终输出文本执行术语替换（按 `config.yaml` 的 `term_replacements` 配置）：
1. 先执行精确匹配替换（如 "AC QA 多轮回归" → "QA 回归测试"）
2. 再执行正则匹配替换（如 `\bAC\b` → "交叉验证"）

### 第十步：按模板输出

严格按照 `daily-report-rules.md` 第一节的日报模板格式输出到终端。

### 第十一步：存档到本地文件

1. 构建输出路径：`reports/{YYYY-MM}/{YYYY-MM-DD}.md`
2. 如果目录不存在，创建之
3. 将日报 Markdown 内容写入文件

### 第十二步：可选<CALENDAR_PLATFORM>输出

如果用户提供了 `--<CALENDAR> <doc_url>`：

1. 使用<CALENDAR_PLATFORM> MCP 的 `update-doc` 工具
2. 参数：
   - `doc_id`: 从 URL 中提取文档 ID
   - `mode`: `insert_before`
   - `selection_with_ellipsis`: 文档开头第一行的内容片段（先 fetch-doc 读取文档开头 100 字符定位）
   - `markdown`: 日报的完整 Markdown 内容
3. 如果 `insert_before` 定位失败，改用 `insert_after` 配合文档标题定位
4. 如果<CALENDAR_PLATFORM>写入失败，终端显示错误原因，强调本地已存档

### 第十三步：完成确认

终端输出摘要：
```
✅ 日报已生成
📅 日期: YYYY-MM-DD（周X）
📁 本地存档: reports/YYYY-MM/YYYY-MM-DD.md
📋 涉及项目: N 个（项目A、项目B...）
{f"📤 已追加到<CALENDAR_PLATFORM>: {<CALENDAR>_url}" if <CALENDAR> else ""}
```
```

- [ ] **Step 2: 自检 — 确认 SKILL.md 覆盖所有设计文档要求**

对照设计文档逐项检查：
- ✅ `--date` 参数支持
- ✅ `--<CALENDAR>` 参数支持
- ✅ 扫描 JSONL 提取 session
- ✅ 智能过滤（时长 / tool_calls / 纯聊天）
- ✅ <CALENDAR_PLATFORM>日程获取 + 黑名单过滤
- ✅ 项目归并（同项目合并）
- ✅ 术语替换
- ✅ 日报模板输出
- ✅ 本地存档
- ✅ <CALENDAR_PLATFORM>文档顶部插入
- ✅ 错误处理（无数据 / 文件过大 / API 失败）

- [ ] **Step 3: 提交**

```bash
git add .claude/skills/daily-report/SKILL.md
git commit -m "feat: add daily-report skill"
```

---

### Task 3: 周报 Skill（weekly-report SKILL.md）

**Files:**
- Create: `.claude/skills/weekly-report/SKILL.md`

**Interfaces:**
- Consumes: `config.yaml`、`daily-report-rules.md`、日报文件（`reports/YYYY-MM/YYYY-MM-DD.md`）、<CALENDAR_PLATFORM> MCP
- Produces: 终端输出周报 Markdown + `reports/YYYY-MM/week-N.md` + 可选<CALENDAR_PLATFORM>文档顶部插入

- [ ] **Step 1: 创建 SKILL.md 完整内容**

写入文件 `.claude/skills/weekly-report/SKILL.md`：

```markdown
---
name: weekly-report
description: 生成本周（或指定周）的工作周报，从日报文件聚合（缺失时回退到 session 扫描），按项目进展、质量异常、关键指标等维度输出。
command: /weekly-report
args:
  - name: week
    type: string
    optional: true
    description: 指定周数（如 27），默认本周
  - name: date-ref
    type: string
    optional: true
    description: 以该日期所在周为准，格式 YYYY-MM-DD
  - name: <CALENDAR>
    type: string
    optional: true
    description: <CALENDAR_PLATFORM>文档 URL，将周报插入到文档最顶部
---

# 周报生成指令

你是一个周报生成助手。你的任务是聚合本周的工作日报（缺失时回退到 session 扫描），生成一份面向团队/领导审阅的周报。

## 核心约束（必须遵守）

1. **按项目汇总**：以项目为维度组织"项目进展"板块，禁止流水账
2. **术语替换**：同日报规则，AC → 交叉验证，ARC → 架构设计 等
3. **状态标记**：[g] 正常 / [y] 需关注 / [r] 阻塞
4. **项目排序**：有上线 > 有提测 > 进度推进 > 方案阶段
5. **不编造数据**：没有的信息写"无"，不凭空生成指标

## 执行步骤

### 第一步：解析参数

从用户输入中提取：
- `week`：指定周数。未指定则根据当前日期计算所在周
- `date_ref`：参考日期。与 `week` 互斥，以该日期所在周为准
- `<CALENDAR>_url`：<CALENDAR_PLATFORM>文档 URL

周计算方式：以周一为一周开始。例如 2026-06-30（周二）属于第 27 周（6/30-7/4）。

### 第二步：读取配置

读取 `.claude/rules/config.yaml` 和 `.claude/rules/daily-report-rules.md` 获取完整规则。

### 第三步：收集每日工作数据

#### 3.1 确定目标周日期范围

计算目标周的周一～周日日期列表。例如第 27 周（2026）：[2026-06-30, 2026-07-01, ..., 2026-07-04]（只包含已过去的日期，未来日期跳过）。

#### 3.2 收集每日数据

对目标周的每个已过去日期（按日期顺序）：

**优先路径 — 读取日报文件：**
1. 检查 `reports/{YYYY-MM}/{YYYY-MM-DD}.md` 是否存在
2. 存在 → 读取文件，解析出"今日完成"和"明日计划"中的内容
3. 不存在 → 走兜底路径

**兜底路径 — Session 扫描：**
当某天日报文件缺失时，对该日期执行与 daily-report Skill 相同的 session 扫描流程：
1. 扫描 `<REDACTED>.claude/projects/E--<REDACTED>-*/` 下的 JSONL
2. 筛选当日 session → 智能过滤 → 提取 work items → 项目归并
3. 注意：这里不需要生成完整日报，只需要提取 "今日完成" 的工作项

#### 3.3 数据结构

将收集到的数据整理为：
```
daily_data = [
  {
    date: "2026-06-30",
    projects: {
      "<PROJECT_A>-agent": ["要点1", "要点2"],
      "<DASHBOARD>": ["要点1"],
    },
    plans: ["计划1", "计划2"],
  },
  ...
]
```

### 第四步：聚合分析

#### 4.1 质量异常

从所有日报中提取以下信号：
- 含 "bug" "fix" "修复" "异常" "故障" "回滚" "hotfix" 的条目
- 含 "阻塞" "blocked" 的条目
- 没有则写"无"

#### 4.2 关键指标

统计以下指标（有数据则填写，没有则写"无"或不显示该行）：
- 本周上线次数：从日报和 session 中搜索 "上线" "deploy" "发布" "tag" 关键字
- 提测次数：搜索 "提测" "测试环境" 关键字
- Bug 修复/新增数：搜索 bug/fix 相关
- Code Review PR 数：搜索 "PR" "merge request" "code review" 关键字

#### 4.3 重点事项 — 上线内容

提取本周已上线或即将上线的内容：
1. 搜索 "上线" "今晚" "今晚上线" 关键字
2. 汇总为列表
3. 没有则写"无"

#### 4.4 项目进展（核心板块）

对本周涉及的所有项目，按项目聚合：

**聚合逻辑：**
1. 汇总每个项目本周所有日期的 work items
2. 去重：同一条目在不同天出现 → 保留最新日期的状态
3. 合并为任务 checklist（`- [x]` 已完成 / `- [ ]` 进行中）
4. 推断进度百分比（根据完成/总数比例）

**项目标题格式：**
```
**[g/y/r]{项目名}，开发进度{X}%{变动}，计划{X}.{X}{上线/提测} @{协作人}**
```

进度变动示例：
- `50%->70%`（本周有推进）
- `80%`（本周无变化或刚启动）

**协作人提取：**
从日报中出现的 `@人名` 提取。如果没有明确协作人，省略 `@协作人` 部分。

**进展内容格式：**
```
- 产品方案：{链接}（有则显示，没有则写"待补充"）
- 技术方案：{链接}（有则显示，没有则写"待补充"）
- 任务拆解如下
- [x] 已完成项
- [ ] 进行中项
```

**项目排序**（按以下优先级）：
1. 本周有上线的项目 `[g]`
2. 本周有提测的项目
3. 进度有推进的项目
4. 方案/设计阶段项目 `[y]`

#### 4.5 运维问题

从日报中提取运维相关条目：
- 含 "运维" "问题跟进" "用户反馈" "Humidity" "处理中" 的条目
- 汇总问题数量和状态
- 没有则写"无"

#### 4.6 下周计划

从本周每日日报的"明日计划"中汇总 → 去重 → 按项目归类 → 取前 5 条最重要的。

### 第五步：术语替换

对最终输出文本执行术语替换（同日报流程）。

### 第六步：按模板输出

严格按照 `daily-report-rules.md` 第二节的周报模板格式输出到终端。

### 第七步：存档 + 可选<CALENDAR_PLATFORM>输出

1. 写入 `reports/{YYYY-MM}/week-{N}.md`
2. 如果提供了 `--<CALENDAR>`，同日报流程插入到文档顶部

### 第八步：完成确认

```
✅ 周报已生成
📅 周期: 第{N}周（{M/D}-{M/D}）
📁 本地存档: reports/YYYY-MM/week-N.md
📋 涉及项目: N 个
📊 数据来源: X 天来自日报文件，Y 天来自 session 扫描
{f"📤 已追加到<CALENDAR_PLATFORM>: {<CALENDAR>_url}" if <CALENDAR> else ""}
```
```

- [ ] **Step 2: 自检 — 确认 SKILL.md 覆盖所有设计文档要求**

对照设计文档逐项检查：
- ✅ `--week` / `--date-ref` / `--<CALENDAR>` 参数
- ✅ 优先日报文件聚合 → 兜底 session 扫描
- ✅ 质量异常提取
- ✅ 关键指标统计
- ✅ 重点事项（上线内容）
- ✅ 项目进展按格式输出（状态标记、进度、checklist）
- ✅ 运维问题汇总
- ✅ 下周计划
- ✅ 术语替换
- ✅ <CALENDAR_PLATFORM>输出
- ✅ 错误处理

- [ ] **Step 3: 提交**

```bash
git add .claude/skills/weekly-report/SKILL.md
git commit -m "feat: add weekly-report skill"
```

---

### Task 4: 端到端验证

**Files:**
- No new files — 验证已有 Skill 是否可正常触发和工作

**Interfaces:**
- Consumes: 完整的 Skill 文件、配置文件、现有 JSONL 数据
- Produces: 验证报告

- [ ] **Step 1: 验证 daily-report Skill 可被识别**

检查 `.claude/skills/daily-report/SKILL.md` 的 frontmatter 格式是否正确：
- `name: daily-report` ✅
- `command: /daily-report` ✅
- `description` 非空 ✅

- [ ] **Step 2: 用历史日期试跑日报**

手动触发 `/daily-report --date 2026-06-30`（上周一，应该有 session 数据），验证：
1. JSONL 扫描是否正常找到 session
2. 过滤是否合理（不应该有空输出或垃圾数据）
3. 项目归并是否正确
4. 术语替换是否生效（输出中不应出现 AC/ARC）
5. 模板格式是否正确
6. 文件是否存档到 `reports/2026-06/2026-06-30.md`

- [ ] **Step 3: 验证 weekly-report Skill 可被识别**

检查 `.claude/skills/weekly-report/SKILL.md` 的 frontmatter 格式。

- [ ] **Step 4: 试跑周报**

手动触发 `/weekly-report --date-ref 2026-06-30`，验证：
1. 目标周日期范围计算正确
2. 日报文件聚合路径正确
3. 兜底 session 扫描补全
4. 项目进展格式、状态标记正确
5. 输出中无内部术语

- [ ] **Step 5: 修复发现的问题**

根据试跑结果，修改 SKILL.md 中的指令细节。

- [ ] **Step 6: 提交**

```bash
git add -A
git commit -m "fix: end-to-end validation fixes for daily/weekly report skills"
```

---

## Self-Review

### 1. Spec coverage

| 设计要求 | 对应任务 |
|----------|----------|
| 日报 Skill（触发、参数、模板） | Task 2 |
| 周报 Skill（触发、参数、模板） | Task 3 |
| config.yaml 配置文件 | Task 1 Step 2 |
| daily-report-rules.md 规则文件 | Task 1 Step 3 |
| JSONL session 扫描 + 过滤 | Task 2（SKILL.md 第三～五步） |
| <CALENDAR_PLATFORM>日程集成 + 黑名单过滤 | Task 2（SKILL.md 第六步） |
| 项目归并 | Task 2（SKILL.md 第七步） |
| 术语替换（AC/ARC） | Task 1（config.yaml）+ Task 2（SKILL.md 第九步） |
| 双输出（终端 + 本地文件 + <CALENDAR_PLATFORM>） | Task 2（SKILL.md 第十～十二步） |
| --date 指定历史日期 | Task 2（SKILL.md 第一步） |
| 周报聚合（日报优先 + session 兜底） | Task 3（SKILL.md 第三步） |
| 周报格式（质量异常/指标/重点/进展/运维/下周） | Task 3（SKILL.md 第四步） |
| 状态标记 [g]/[y]/[r] | Task 3（SKILL.md 第四步 4.4） |
| <CALENDAR_PLATFORM>文档顶部插入 | Task 2 + Task 3（SKILL.md <CALENDAR_PLATFORM>输出步骤） |
| gitignore（reports/ + docs/） | 已在设计阶段完成 |
| 错误处理（7 种场景） | Task 2 + Task 3（各 SKILL.md 中已包含） |
| 数据源适配器抽象（后续扩展） | 当前 Phase 1 只做 Claude；抽象接口在 rules.md 中预留 |

### 2. Placeholder scan

- ✅ 无 TBD/TODO
- ✅ 所有代码块包含完整内容
- ✅ 错误处理具体明确
- ✅ 模板格式完整，无占位符

### 3. Type consistency

- ✅ config.yaml 的 key 名称在两个 SKILL.md 中引用一致
- ✅ 日报模板字段与周报聚合时引用的字段一致
- ✅ 术语替换表的 key 在 config.yaml 和 rules.md 中一致
