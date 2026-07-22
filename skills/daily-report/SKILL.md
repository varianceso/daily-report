---
name: daily-report
description: 生成当天（或指定日期）的工作日报，从 Claude session 记录中提取工作内容，结合<CALENDAR_PLATFORM>日程，按项目汇总输出。支持 --d 指定日期、--fs 追加到<CALENDAR_PLATFORM>文档。
argument-hint: [--d YYYY-MM-DD] [--fs <<CALENDAR_PLATFORM>文档URL>]
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit, mcp__<CALENDAR_MCP>__update-doc, mcp__<CALENDAR_MCP>__fetch-doc]
---

# 日报生成指令

扫描指定日期的 Claude Code session + git commit，结合<CALENDAR_PLATFORM>日历，生成面向团队审阅的日报。

## 核心约束

1. **禁止流水账**：同项目多 session 合并为要点
2. **术语替换**：输出前按 `../../config.yaml` 的 `term_replacements` 替换内部术语
3. **按项目分组**：以项目为二级标题，无法归类入"其他"
4. **每条 ≤ 30 字**，突出产出而非过程
5. **进度与报告分离**：执行过程中的 `🔍` `📋` `🔄` 是叙述性进度，**不混入最终日报**；最终日报必须是**干净的 Markdown 文档**（见 `../../daily-report-rules.md` 第一节模板）
6. **链接格式**：`[文档标题](https://...)` — 标准 Markdown 超链接，不使用裸 URL

## 执行流程

### 1. 解析参数

提取 `--date`（或 `--d`，默认当天）、`--<CALENDAR>`（或 `--fs`，可选）。

### 2. 读取配置

检查 `../../config.yaml` 是否存在：

**如果 config.yaml 存在**：直接读取项目映射、术语表、过滤阈值、日程黑名单，继续执行。

**如果 config.yaml 不存在**：尝试从旧版本自动迁移。

1. 列出 `../../` 同级目录（即 `../` 下的所有版本号子目录，如 `1.0.0` `1.0.1` ...）
2. 按版本号降序查找，取第一个存在 `config.yaml` 的目录
3. 如果找到：复制到 `../../config.yaml`，提示 `📋 已从 v{X.X.X} 迁移配置到当前版本`
4. 如果未找到（真正首次安装）：进入交互式配置流程 —— 扫描项目 → 选择 → 命名 → 术语 → 数据源 → 写入 `../../config.yaml`

交互式配置流程（真正首次安装时）：

1. 提示用户：`⚙️ 检测到首次使用，需要配置以下内容：`

2. **项目配置**：扫描 `data_sources.claude_session_base`（默认 `C:/Users/<用户名>/.claude/projects`）下所有项目的 JSONL，提取 cwd 去重列表。逐项询问：
   ```
   📁 扫描到 N 个项目工作目录:
   1. E:\<REDACTED>\<PROJECT_A>-agent
   2. E:\<REDACTED>\<DASHBOARD>
   3. E:\<REDACTED>\daily-report
   ...
   
   请选择要纳入日报统计的项目（输入序号，多选用空格分隔，如 "1 2 4"）：
   ```

3. **项目显示名称**：对每个选中的项目，询问对外显示名称（默认用目录名）：
   ```
   E:\<REDACTED>\<PROJECT_A>-agent → 对外名称（默认"<PROJECT_A>-agent"）：
   ```

4. **术语替换**：询问是否有内部术语需要替换：
   ```
   是否需要配置内部术语替换？（y/n）
   如："AC" → "交叉验证"、"ARC" → "架构设计"
   输入格式：内部术语=对外表述，每行一个，空行结束
   ```

5. **数据源路径**：确认 session 存储路径：
   ```
   Claude session 存储路径（默认 C:/Users/<当前用户名>/.claude/projects）：
   ```

6. 将所有回答写入 `../../config.yaml`，输出 `✅ 配置已保存到 config.yaml`，继续执行日报生成。

### 3. 数据采集（两步并行）

**3a. Git commit（优先）：**
对每个项目，执行 `git -C <项目路径> log --oneline --since="<日期>" --until="<次日>" --format="%s"`，提取 commit message。

**3b. Session JSONL（补充细节）：**
扫描 `data_sources.claude_session_base` 下所有 `{project_prefix}*/**.jsonl`，筛选 `timestamp` 在目标日期的行。>500KB 文件先读尾部 500 行再前翻。

> `🔍 Claude: 扫描 N 个项目，找到 M 个当日 session → 过滤后保留 K 个`

**3c. Codex Session JSONL（当 data_sources.codex_session_base 已配置）：**

扫描 `data_sources.codex_session_base`（默认 `~/.codex/sessions`）下所有 `rollout-*.jsonl` 文件。
从文件名提取时间戳（格式 `rollout-YYYY-MM-DDTHH-MM-SS-{uuid}.jsonl`），筛选目标日期的文件。

**Codex JSONL 结构**（与 Claude Code 不同，需独立解析）：

| Codex ThreadItem `type` | 提取内容 | 用途 |
|--------------------------|----------|------|
| `userMessage` | `content` 数组中的 `text` 字段 | 用户任务描述 |
| `agentMessage` | `text` 字段 | AI 产出内容 |
| `commandExecution` | `command` + **`cwd`** 字段 | **项目归属** + 实际操作 |
| `fileChange` | `changes` 数组（文件名 + 状态） | 文件修改记录 |
| `mcpToolCall` | `server` + `tool` + `arguments` | MCP 工具调用（含日历文档创建/更新） |

**关键差异**：
- 无顶层 `timestamp` 字段 → 用文件名中时间戳判断日期
- 无顶层 `cwd` 字段 → 从 `commandExecution.cwd` 提取工作目录，归一化（`\`→`/`，小写，去末尾`/`）后匹配 config.yaml projects 映射
- 无 `type: "user"` / `type: "assistant"` 标记 → 用 `userMessage` / `agentMessage` 区分
- 行内无 `tool_calls` 计数 → 用 `commandExecution` + `fileChange` + `mcpToolCall` 行数之和作为等效 tool_calls

**过滤规则**：丢弃等效 tool_calls < 3 或时长 < 5min 的 session（逻辑同步骤 4）。

> `🔍 Codex: 扫描到 C 个当日 rollout → 过滤后保留 D 个`

### 4. 智能过滤

丢弃：tool_calls < 3 或 时长 < 5min 或 纯对话无产出。

### 5. 提取工作内容（⚠️ 关键步骤，防漏报）

对每个保留的 session，**必须读取全部 user 消息**（非 meta、非系统），不能只读前几条：

1. **识别项目**：cwd 归一化（`\`→`/`，小写，去末尾`/`）后匹配 config.yaml projects 映射
   - **Claude session**：cwd 来自顶层 `cwd` 字段
   - **Codex session**：cwd 来自 `commandExecution` 行的 `cwd` 字段（取出现频率最高的那个）
2. **提取 gitBranch**：从第一条消息的 `gitBranch` 字段获取分支名，**仅供内部判断任务是否合并**。branch 中的数字编号（如 `feature/701-job-management` 的 `701`）**不写入最终日报**——外部读者看不懂工单号，只保留可读的任务描述
   - **Codex session**：无 `gitBranch` 字段，用「项目归属 + 任务关键词」组合替代内部合并判断
3. **提取任务主题**：
   - **Claude session**：遍历所有 type: "user" 消息，找到 X 条核心任务描述（非链接、非命令、非 tool_result）
   - **Codex session**：遍历所有 `userMessage` 行的 `content[].text`，提取核心任务描述
   - **如果第一条消息是<CALENDAR_PLATFORM>链接**：用 `fetch-doc` 获取文档标题作为任务主题标注
   - **任务主题禁止包含分支号**：❌"岗位管理基建（701）" → ✅"岗位管理基建"
   - 示例：链接 `GVYzdsPYdoYO36xx8HNc8x75npH` → 文档标题 "【BRD】岗位管理基建" → 任务主题 "岗位管理基建"
4. **提取产出**：优先 git commit + tool calls 结果，一句话概括
   - **Codex session**：等效 tool_calls = `commandExecution` + `fileChange` + `mcpToolCall` 行数之和
5. **⚠️ 提取<CALENDAR_PLATFORM>文档产物**（关键交付物，易遗漏）：
   - **Claude session**：搜索 assistant 消息中 `mcp__<CALENDAR_MCP>__create-doc` 的 tool_use → 获取 `title` 参数
   - **Codex session**：搜索 `mcpToolCall` 行中 `server` 为 `<CALENDAR_MCP>` 且 `tool` 为 `create-doc` 的记录 → 从 `arguments` 提取 `title`
   - 搜索对应该 tool_use 的 tool_result → 提取 `doc_url`（<CALENDAR_PLATFORM>文档链接）
   - 搜索 `mcp__<CALENDAR_MCP>__update-doc` 的 tool_use → 提取目标 `doc_id`（更新了哪个文档）
   - 将文档标题 + 链接作为产出附在要点末尾
   - 示例：`- 5 技能权限配置扫描分析 → [<CALENDAR_PLATFORM>文档](https://www.<CALENDAR_DOMAIN>/docx/AeNNdhJhJo6qrlxYL7zcDCttnOf)`
6. **产出导向**：❌"执行了 3 次 Edit" → ✅"修复 vacation/export 日期格式兼容问题"

**逐个 session 输出发现**（防止漏报）：
> `📋 session 1/4: <DASHBOARD>/feature/701-job-management → "岗位管理基建 BRD 评审 + slug 搭建"`
> `📋 session 2/4: <DASHBOARD>/main → "5 技能权限配置扫描"`

### 6. 获取<CALENDAR_PLATFORM>日程

使用<CALENDAR_PLATFORM> MCP 获取目标日期日历事件。过滤：标题含 config.yaml `calendar_keyword_blacklist` 关键词忽略（大小写不敏感）。

**失败时**：读取 config.yaml 的 `calendar_keyword_blacklist`，动态生成提示：
> `⚠️ 今日日程: 请通过「<TOOL_B>」获取今日日程数据后补充（过滤规则：排除含"{key1}" "{key2}" ...的日程）`
其中 key1, key2... 为该用户配置的实际黑名单关键词。

### 7. 项目归并

**合并规则**（防止误合并）：
- **同项目 + 同分支名** → 大概率是同一个任务 → 合并
- **同项目 + 不同分支名** → 不同任务 → **不合并**，各自保留独立要点
- 分支名无法区分时（如都在 main 上，或 Codex session 无 gitBranch）：根据用户消息中的任务关键词判断是否同一任务
- 单项目 > 4 条 → 保留前 4，其余入"其他"
- **跨平台 session**（Claude + Codex 同项目同天）：按任务关键词判断是否合并，不同平台来源在归并说明中标注

**归并前先输出确认**：
> `🔄 归并: 同项目 2 个 session 属于不同分支 → 保留为 2 条独立要点`
> `🔄 跨平台: Claude session + Codex session 都属"岗位管理基建" → 合并为 1 条`

### 8. 术语替换 + 明日计划

执行术语替换（精确→正则，按 config.yaml term_replacements）。推测 2-3 条明日计划。

### 9. 输出

**终端输出**：输出**纯 Markdown**，不含进度前缀。严格按照 `../../daily-report-rules.md` 第一节模板：

```
# 日报 — YYYY-MM-DD（周X）

## 📅 今日日程
| 时间 | 事项 |

## ✅ 今日完成

### {项目名}
**[g]项目名，开发进度X%，计划X.X上线**
- 要点 — [文档标题](链接)
```

- 项目标题行：[g]/[y]/[r] 状态标记 + 开发进度 + 计划上线/提测日期
- 进度有变动写 `50%->70%`，无变动写当前值
- 链接格式：`[名称](https://<REDACTED>/docx/xxx)` — 标准 Markdown 超链接
- 单项目 > 4 条要点时保留前 4 条，其余入"其他"

**本地存档**：写入 `../../reports/{YYYY-MM}/{YYYY-MM-DD}.md`（目录不存在则创建），内容同上。

**<CALENDAR_PLATFORM>**：若 `--<CALENDAR>`，`update-doc` + `insert_before` 插入文档顶部。失败报错 + 本地存档保留。

### <REDACTED> 完成确认

```
✅ 日报已生成
📅 日期: YYYY-MM-DD（周X）
📁 本地存档: reports/YYYY-MM/YYYY-MM-DD.md
📋 涉及项目: N 个（项目A、项目B...）
（若 --<CALENDAR>：📤 已追加到<CALENDAR_PLATFORM>: <url>）
```
