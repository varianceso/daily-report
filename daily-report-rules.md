# 日报/周报处理规则

> 本文件定义日报和周报的详细处理规则、输出模板和术语替换表。

---

## 一、日报输出模板（严格照此格式）

```
# 日报 — {YYYY-MM-DD}（{周X}）

## 📅 今日日程
| 时间 | 事项 |
|------|------|
| HH:MM-HH:MM | 事项描述 |

（无日程数据时显示以下提示，过滤规则从 config.yaml 动态读取）：
> ⚠️ 今日日程: 请通过「<TOOL_B>」获取今日日程数据后补充（过滤规则：排除含 {从 config.yaml calendar_keyword_blacklist 动态拼接} 的日程）

## ✅ 今日完成

### {项目A显示名称}
**[g]项目A，开发进度70%，计划X.X上线 @协作人**
- 要点1 — [<CALENDAR_PLATFORM>文档](链接)
- 要点2

### {项目B显示名称}
**[y]项目B，技术方案中，计划X.X上线**
- 要点1 — [<CALENDAR_PLATFORM>文档](链接)

### 其他
- 交叉验证：xxx
- 其他非项目归属的工作

## 📋 明日计划
1. {项目A}: xxxx
2. {项目B}: xxxx
```

> **项目标题格式**：`**[g/y/r]{项目名}，开发进度{X}%{->变动}，计划{X}.{X}{上线/提测} @{协作人}**`
> - `[g]` 正常 / `[y]` 需关注 / `[r]` 阻塞（同周报状态标记规则）
> - 有进度变动的写 `50%->70%`，无变动只写当前值
> - @协作人 从 session 中提取，无可省略
> - 无上线/提测计划时可写 `方案阶段` 或 `开发中`

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
**本周上线内容**
1. XXXX，已于X.X上线
2. XXXX，计划X.X今晚上线

（无上线内容时写"无"）

### 项目进展
**[g]{项目名}，{状态描述}，{计划}**
- 产品方案/业务需求：{链接}
- 技术方案：{链接} / 待PRD产出后输出
- 上线方案：{链接}（有则附）
- 任务拆解如下
- [x] 已完成项
- [ ] 进行中项

（已完成上线的项目同样列出，标注"已于X.X上线"）
（项目排列顺序：已完成上线 > 提测中 > 开发中 > 方案阶段）

### 其他
1. {跨项目工作} — [文档](链接)
（无其他内容时写"无"）

### 下周计划
1. {项目名}，{具体计划}
2. xxxx
```

## 三、Session → WorkItem 提取规则

1. 定位当日 session：按 `timestamp` 筛选目标日期
2. 识别项目归属：从 `cwd` 字段匹配 config.yaml 的 projects 映射（先归一化：`\`→`/`，小写，去末尾`/`）
3. 提取用户意图：读 `type: "user"` 非 meta 消息
4. 提取产出：读 assistant tool calls + git commit message + **<CALENDAR_PLATFORM>文档创建/更新记录**（`create-doc` → 提取 title+doc_url; `update-doc` → 提取 doc_id）
5. 过滤噪音：跳过 mode/permission-mode 系统消息、isMeta=true 元数据、Hook stdout

### 智能过滤

- 丢弃 tool_calls < 3 的 session
- 丢弃时长 < 5min 的 session
- 丢弃纯对话无产出的 session

### 项目归并

- **分支编号不写入日报**：`feature/701-job-management` 的 `701` 仅用于内部判断合并，最终输出只用可读任务描述（如"岗位管理基建"，禁止出现"701"）
- **同项目 + 同分支名** → 合并为 1 条
- **同项目 + 不同分支名** → **不合并**
- 分支名无法区分时：根据任务关键词判断
- 单项目 > 4 条 → 保留前 4，其余入"其他"

## 四、术语替换规则

替换顺序：先精确匹配 → 再正则匹配

| 匹配方式 | 内部术语 | 对外表述 |
|----------|----------|----------|
| 精确 | AC QA 多轮回归 | QA 回归测试 |
| 精确 | AC 多轮方案评审 | 多轮方案评审 |
| 精确 | 搭建 slug | 建立项目文档目录结构 |
| 正则 `\bAC\b` | AC | 交叉验证 |
| 正则 `\bARC\b` | ARC | 架构设计 |
| 正则 `\bslug\b` | slug | 项目文档目录 |

**重要**：替换仅应用于最终输出文本，不修改原始 JSONL 数据。

## 五、<CALENDAR_PLATFORM>日程集成

使用<CALENDAR_PLATFORM> MCP 获取目标日期的日历事件。标题含 config.yaml 中 `calendar_keyword_blacklist` 任一关键词的忽略（大小写不敏感）。

**注意**：当前 <CALENDAR_MCP> 不包含日历 API。如有独立的<CALENDAR_PLATFORM>日历 MCP 则调用之；否则日程区显示 `（日历获取失败）`。

## 六、周报聚合规则

1. 从 config.yaml `weekly.week_start_day` 读取起始日（默认 `monday`），以此向后 7 天为目标周。如 `thursday` → 上周四~本周三
2. 遍历每天：优先读 `reports/YYYY-MM/YYYY-MM-DD.md` → 缺失回退 session 扫描
3. 汇总六大板块：质量异常 / 关键指标 / 重点事项 / 项目进展 / 运维问题 / 下周计划
4. 项目排序：上线 > 提测 > 进度推进 > 方案阶段
5. 下周计划：汇总各日报"明日计划" → 剔除已完成 → 去重 → 取前 5

## 七、<CALENDAR_PLATFORM>输出规则

当提供 `--<CALENDAR> <doc_url>` 时：
1. 使用<CALENDAR_PLATFORM> MCP 的 `update-doc` 工具，`insert_before` 模式插入文档顶部
2. 失败回退 `insert_after` + 文档标题定位
3. 全部失败：终端报错 + 本地存档保留

## 八、错误处理

| 场景 | 处理 |
|------|------|
| 目标日期无 session | 返回"YYYY-MM-DD 无工作记录" |
| JSONL > 500KB | 先读尾部 500 行，不足时前翻 |
| <CALENDAR_PLATFORM>日历失败 | 日程区显示"（日历获取失败）" |
| <CALENDAR_PLATFORM>写入失败 | 终端报错，本地存档不受影响 |
| 周报某天缺日报 | 回退 session 扫描 |
| session 解析异常 | 跳过并输出 warning |

## 九、Codex Session 解析规则

### 9.1 文件定位

Codex session 以 JSONL 文件存储在 `data_sources.codex_session_base`（默认 `~/.codex/sessions`）目录下，文件名格式为：

```
rollout-{YYYY-MM-DDTHH-MM-SS}-{uuid}.jsonl
```

例：`rollout-2025-05-07T17-24-21-5973b6c0-94b8-487b-a530-2aeb6098ae0e.jsonl`

从文件名提取的时间戳（UTC）用于判断日期归属。

### 9.2 JSONL 行结构

每行是一个 JSON 事件，`type` 字段标识类型。以下是提取日报信息所需的类型映射：

| Codex `type` | 提取字段 | 用途 |
|-------------|----------|------|
| `userMessage` | `content[].text` | 用户任务描述 |
| `agentMessage` | `text` | AI 产出摘要 |
| `commandExecution` | `command`, `cwd`, `status`, `exitCode` | 命令操作 + **cwd 项目归属** |
| `fileChange` | `changes[].path`, `changes[].status` | 文件修改记录 |
| `mcpToolCall` | `server`, `tool`, `arguments`, `result` | MCP 工具调用（含日历） |
| `reasoning` | `summary[]`, `content[]` | AI 推理过程（可忽略） |
| `webSearch` | `query` | 网络搜索（可忽略） |

### 9.3 cwd 提取策略

Codex session 无顶层 `cwd` 字段。项目归属通过以下策略确定：

1. 收集所有 `commandExecution` 行中非空的 `cwd` 字段
2. 取出现频率最高的 cwd 作为该 session 的主工作目录
3. 归一化（`\`→`/`，小写，去末尾`/`）后匹配 config.yaml projects 映射
4. 若全无 `commandExecution` 行（纯对话 session），标记为"无法识别项目"，归入"其他"

### 9.4 等效 tool_calls 计算

Codex 无 `tool_calls` 计数。过滤时计算等效值：

```
等效 tool_calls = count(commandExecution) + count(fileChange) + count(mcpToolCall)
```

丢弃等效值 < `filter.min_tool_calls` 的 session。

### 9.5 gitBranch 缺失处理

Codex session 无 `gitBranch` 字段。合并判断改为：

- **同项目 + 任务关键词高度重叠** → 合并
- **同项目 + 任务关键词明显不同** → 不合并

### 9.6 与 Claude 的协同

当同一天同时存在 Claude session 和 Codex session 时：
- 两个数据源并⾏采集，合并后统一进入步骤 4（智能过滤）
- 项目归并时按任务关键词判断，不按数据源区分
- 归并说明中标注跨平台来源
