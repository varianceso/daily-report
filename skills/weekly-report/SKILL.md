---
name: weekly-report
description: 生成本周（或指定周）的工作周报，从日报文件聚合（缺失时回退到 session 扫描），按项目进展、质量异常、关键指标等维度输出。支持 --w 指定周数、--dr 指定日期、--fs 追加到<CALENDAR_PLATFORM>文档。
argument-hint: [--w N] [--dr YYYY-MM-DD] [--fs <<CALENDAR_PLATFORM>文档URL>]
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit, mcp__<CALENDAR_MCP>__update-doc, mcp__<CALENDAR_MCP>__fetch-doc]
---

# 周报生成指令

聚合本周日报（缺失时回退 session 扫描），生成面向团队审阅的周报。

## 核心约束

1. **按项目汇总**，禁止流水账
2. **术语替换**：按 `../../config.yaml` term_replacements 替换
3. **状态标记**：[g] 正常 / [y] 需关注 / [r] 阻塞
4. **项目排序**：有上线 > 有提测 > 进度推进 > 方案阶段
5. **不编造数据**：没有写"无"
6. **进度与报告分离**：执行中的进度输出不混入最终周报；最终周报必须是**干净的 Markdown 文档**（见 `../../daily-report-rules.md` 第二节模板）
7. **链接格式**：`[文档标题](https://...)` 标准 Markdown 超链接

## 执行流程

### 1. 解析参数

提取 `--week`（或 `--w`）、`--date-ref`（或 `--dr`，与 week 互斥）、`--<CALENDAR>`（或 `--fs`）。同时提供 `--week` + `--date-ref` 时以 `--date-ref` 为准。

**周计算方式**：从 config.yaml 的 `weekly.week_start_day` 读取起始日（默认 `monday`），以此为周期第一天向后7天。例如 `week_start_day: thursday` → 上周四~本周三（适合周四开周会的团队）。

### 2. 读取配置

读取 `../../config.yaml` 和 `../../daily-report-rules.md`。

**如果 config.yaml 不存在**：先从同级旧版本目录自动迁移（同 daily-report 流程）；仍未找到则进入交互式配置流程（扫描项目 → 选择 → 命名 → 术语 → 数据源 → 保存），写入 config.yaml 后继续。

### 3. 收集每日数据

根据 `week_start_day` 确定目标周范围（如 `thursday` → 上周四~本周三），只处理已过去的日期。

每天：**优先**读 `../../reports/{YYYY-MM}/{YYYY-MM-DD}.md` 解析"今日完成" → **缺失则兜底**执行 session 扫描（同 daily-report 流程，Claude + Codex 双数据源并行采集）。

> `📊 数据来源: X 天来自日报文件，Y 天来自 session 扫描（含 Claude/Codex）`

### 4. 聚合分析（六大板块）

**4.1 质量异常：** 收集含 "bug""fix""修复""异常""故障""回滚""阻塞" 的信号项，没有写"无"。

**4.2 关键指标：** 统计上线次数 / 提测次数 / Bug 修复&新增数 / Code Review PR 数。没有写"无"。

**4.3 重点事项：**

**不仅搜索关键词，更要提取整周实际里程碑**：
- 本周已上线的项目/功能（标注上线日期，如"6.30已上线"）
- 计划今晚/本周上线的内容（标注"计划X.X今晚上线"）
- 已完成的里程碑节点（如"提测""上线方案准备"）
- 没有则写"无"

**4.4 项目进展（核心）：**

聚合逻辑：
1. **按分支拆分为独立项目**：不同 gitBranch 的 session 必须分成独立项目标题，不合并（Codex session 无 gitBranch → 按任务关键词 + 项目归属区分）
   - `feature/625-org-standard-management` → "组织架构规范化管理"
   - `feature/org-architecture-diagram` → "组织架构调整闭环-新增组织架构图能力"
   - `feature/701-job-management` → "岗位管理基建"
2. 每项目汇总本周所有 work items，去重后合并为 checklist
3. **已完成上线的项目也要列出**（不是删掉），标注状态和上线日期
4. 推断进度和状态描述（如"自测中""方案阶段""联调中"）

标题格式：`**[g/y/r]{项目名}，{状态}，{计划} @{协作人}**`
- 有变动写 `50%->70%`，无变动写当前值
- @协作人 从日报中提取，无则省略

排序：已完成上线 > 提测 > 开发中 > 方案阶段。

**项目内容格式**（参考模板）：
```
- 产品方案/业务需求：{链接}
- 技术方案：{链接}
- 上线方案：{链接}（有则附）
- 任务拆解如下
- [x] 已完成项
- [ ] 进行中项
```
如项目有延后/风险，在标题后加一行 **延后说明**（如"**延后一天，因XX需求紧急介入**"）。

**4.5 "其他"区：** 汇总不归属单个项目但值得记录的工作（如跨项目分析扫描、学习备考、文档规范类工作），以编号列表附文档链接。没有写"无"。

**4.6 下周计划：** 汇总各日报"明日计划" → 剔除本周已完成 → 去重 → 按项目归类 → 取前 5。

> `📋 聚合 N 个项目，M 项质量信号，K 项上线内容`

### 5. 术语替换

### 6. 输出

**终端输出**：输出**纯 Markdown**，不含进度前缀。严格按照 `../../daily-report-rules.md` 第二节模板。

**本地存档**：写入 `../../reports/{YYYY-MM}/week-{N}.md`（目录不存在则创建）。

**<CALENDAR_PLATFORM>**：若 `--<CALENDAR>`，`update-doc` + `insert_before` 插入文档顶部。失败报错 + 本地存档保留。

### 7. 完成确认

```
✅ 周报已生成
📅 周期: 第{N}周（{M/D}-{M/D}）
📁 本地存档: reports/YYYY-MM/week-N.md
📋 涉及项目: N 个
📊 数据来源: X 天来自日报，Y 天来自 session 扫描
（若 --<CALENDAR>：📤 已追加到<CALENDAR_PLATFORM>: <url>）
```
