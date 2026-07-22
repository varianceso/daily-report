# Codex 适配执行计划

> 2026-07-22 | 方案 A：双解析器并行

## 改动清单

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 1 | `.codex-plugin/plugin.json` | **新建** | Codex 插件清单，含 interface 定义 |
| 2 | `config.example.yaml` | 修改 | 新增 `data_sources.codex_session_base`（默认 `~/.codex/sessions`） |
| 3 | `skills/daily-report/SKILL.md` | 修改 | 步骤3 增加 Codex JSONL 解析流程（3c），允许 Claude/Codex 双数据源并行采集 |
| 4 | `skills/weekly-report/SKILL.md` | 修改 | 数据采集步骤同步增加 Codex 回退路径 |
| 5 | `daily-report-rules.md` | 修改 | 新增第八节「Codex Session 解析规则」 |
| 6 | `README.md` | 修改 | Phase 2 标记完成，增加 Codex 安装说明 |
| 7 | `marketplace.json` | 修改 | 更新 tags |
| 8 | `docs/CHANGELOG.md` | 新建 | 记录 v1.1.0 变更 |

## 各文件改动详情

### 1. `.codex-plugin/plugin.json`（新建）

参考 `coder-delegation/.codex-plugin/plugin.json` 结构，创建 Codex 插件清单：
- `name`: `daily-report`
- `skills`: `./skills/`
- `interface`: 含 displayName、capabilities（Read/Write/Execute）、defaultPrompt 等

### 2. `config.example.yaml` — 新增 Codex 数据源

```yaml
data_sources:
  claude_session_base: "C:/Users/<你的用户名>/.claude/projects"
  codex_session_base: "~/.codex/sessions"          # 新增
  project_prefix: "..."
```

### 3. `skills/daily-report/SKILL.md` — 新增步骤 3c

在步骤 3（数据采集）中新增 3c 子步骤：

```
### 3c. Codex Session JSONL（当 data_sources.codex_session_base 已配置）

扫描 codex_session_base 下 rollout-*.jsonl，从文件名提取日期，筛选目标日期的文件。

Codex JSONL 结构（区别于 Claude Code）：
- 每行是一个 ThreadItem 事件，type 字段标识类型
- userMessage → 用户输入，content 数组含消息文本
- agentMessage → AI 回复，text 字段
- commandExecution → 命令执行，cwd 字段用于项目归属
- fileChange → 文件修改记录
- mcpToolCall → MCP 工具调用
- 无顶层 timestamp → 用文件名中时间戳判断日期
- 无顶层 cwd → 从 commandExecution.cwd 提取

项目归属：cwd 归一化后匹配 config.yaml projects 映射
```

同时调整步骤 7（项目归并）支持跨平台 session 合并。

### 4. `skills/weekly-report/SKILL.md`

步骤 3（收集每日数据）中增加 Codex 回退路径说明，与 daily-report 保持一致。

### 5. `daily-report-rules.md` — 新增第八节

新增「Codex Session 解析规则」章节，记录：
- JSONL 文件路径和命名规范
- ThreadItem 类型映射表
- cwd 提取策略
- 项目归属识别逻辑

### 6. `README.md`

- 扩展计划：Phase 2 标记 ✅
- 新增 Codex 安装命令
- 新增 `data_sources.codex_session_base` 配置说明

### 7. `marketplace.json` / `.claude-plugin/marketplace.json`

tags 增加 `codex`。

## 影响范围

- **不破坏现有功能**：Claude 解析逻辑完全不变，Codex 为独立并行路径
- **向后兼容**：未配置 `codex_session_base` 时 Codex 解析路径不触发
- **总改动量**：~8 文件，~120 行新增/修改
