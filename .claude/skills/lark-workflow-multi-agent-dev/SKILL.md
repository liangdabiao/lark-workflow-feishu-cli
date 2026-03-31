---
name: lark-workflow-multi-agent-dev
version: 1.0.0
description: "多编程代理协同：支持竞赛模式、分工模式、流水线模式三种工作模式，任务管理和结果追踪通过飞书多维表格和消息完成。当用户需要'并行开发'、'多代理'、'竞赛模式'、'分工模式'、'流水线协作'、'代码对比'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 多编程代理协同工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我用多代理做这个任务" / "并行开发"
- "同一个任务让多个 AI 各写一份" / "竞赛模式"
- "这几件事分别交给不同的 AI" / "分工模式"
- "一个写代码、一个做审查、一个写测试" / "流水线模式"
- "对比一下哪个方案更好" / "代码对比"
- "看看当前的开发任务" / "任务列表"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain im,base,task
```

## 工作流总览

```
任务输入：
  用户描述任务 ──► AI 解析任务复杂度和可拆分性

模式选择：
  ├── 竞赛模式 ──► 同一任务 × N 代理 → 对比选优
  ├── 分工模式 ──► N 个子任务 × 各代理 → 并行完成
  └── 流水线模式 ──► 代码 → 审查 → 测试 → 顺序协作

执行：
  Agent Tool ──► 启动子代理并行执行

追踪：
  base +record-upsert ──► 记录任务和结果

通知：
  im +messages-send ──► 完成通知 + 结果摘要
```

---

## 数据初始化

### Base 表结构

**表 1：开发任务 (dev_tasks)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 任务描述 | 文本 | 任务内容的简要描述 |
| 工作模式 | 单选 | 竞赛/分工/流水线 |
| 分配代理 | 文本 | 分配给的代理名称 |
| 代理数量 | 数字 | 参与的代理数量 |
| 状态 | 单选 | 待分配/执行中/已完成/已取消 |
| 结果摘要 | 文本 | 执行结果的简要描述 |
| 结果详情 | 文本 | 详细结果（如代码diff链接） |
| 代码仓库 | URL | 关联的代码仓库 |
| 分支名 | 文本 | 创建的分支名 |
| 开始时间 | 日期 | 任务开始时间 |
| 完成时间 | 日期 | 任务完成时间 |
| 耗时(秒) | 数字 | 任务执行耗时 |

**表 2：竞赛记录 (competition_log)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 任务描述 | 文本 | 竞赛任务描述 |
| 代理A名称 | 文本 | 如 "Claude Opus 4.6" |
| 代理A结果 | 文本 | 代理 A 的执行结果摘要 |
| 代理B名称 | 文本 | 如 "Claude Sonnet 4.6" |
| 代理B结果 | 文本 | 代理 B 的执行结果摘要 |
| 代理C名称 | 文本 | 可选第三个代理 |
| 代理C结果 | 文本 | |
| 优胜者 | 文本 | AI 评估后的推荐方案 |
| 评估理由 | 文本 | 为什么选择该方案 |
| 评估日期 | 日期 | 评估时间 |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "开发任务" --fields '[
  {"name":"任务描述","type":"text"},
  {"name":"工作模式","type":"select","multiple":false,"options":[{"name":"竞赛"},{"name":"分工"},{"name":"流水线"}]},
  {"name":"分配代理","type":"text"},
  {"name":"代理数量","type":"number"},
  {"name":"状态","type":"select","multiple":false,"options":[{"name":"待分配"},{"name":"执行中"},{"name":"已完成"},{"name":"已取消"}]},
  {"name":"结果摘要","type":"text"},
  {"name":"结果详情","type":"text"},
  {"name":"代码仓库","type":"link"},
  {"name":"分支名","type":"text"},
  {"name":"开始时间","type":"datetime"},
  {"name":"完成时间","type":"datetime"},
  {"name":"耗时(秒)","type":"number"}
]'
lark-cli base +table-create --name "竞赛记录" --fields '[
  {"name":"任务描述","type":"text"},
  {"name":"代理A名称","type":"text"},
  {"name":"代理A结果","type":"text"},
  {"name":"代理B名称","type":"text"},
  {"name":"代理B结果","type":"text"},
  {"name":"代理C名称","type":"text"},
  {"name":"代理C结果","type":"text"},
  {"name":"优胜者","type":"text"},
  {"name":"评估理由","type":"text"},
  {"name":"评估日期","type":"datetime"}
]'
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 模式一：竞赛模式

### 适用场景
- 需要找到最优实现方案
- 同一个需求有多种实现思路
- 对代码质量要求较高的核心功能

### 工作流

```
用户描述任务
      │
      ▼
AI 解析任务，确保描述清晰无歧义
      │
      ├──► Agent 1（如 Claude Opus 4.6）──► 结果 A
      ├──► Agent 2（如 Claude Sonnet 4.6）──► 结果 B
      └──► Agent 3（如 Claude Haiku 4.5）──► 结果 C
                                                      │
                                                      ▼
                                              AI 对比评估
                                                      │
                                                      ▼
                                              选出最优方案
                                                      │
                                                      ▼
                                              飞书消息通知
```

### 执行步骤

1. **解析任务**：AI 确保任务描述完整、无歧义，必要时追问用户补充细节
2. **启动代理**：使用 Claude Code 的 Agent Tool 并行启动多个子代理（建议 2-3 个）

**Agent Tool 调用示例（竞赛模式）：**

在一条消息中同时发起多个 Agent Tool 调用即可实现并行：

```
Agent(subagent_type="general-purpose", model="opus",   description="竞赛方案A", prompt="在仓库 {repo_path} 中完成任务：{任务描述}。要求：写出完整的实现代码。")
Agent(subagent_type="general-purpose", model="sonnet",  description="竞赛方案B", prompt="在仓库 {repo_path} 中完成任务：{任务描述}。要求：写出完整的实现代码。")
Agent(subagent_type="general-purpose", model="haiku",   description="竞赛方案C", prompt="在仓库 {repo_path} 中完成任务：{任务描述}。要求：写出完整的实现代码。")
```

> **说明**：Agent Tool 是 Claude Code 的内置能力（非 lark-cli 命令）。
> - `subagent_type`：使用 `general-purpose` 处理代码任务
> - `model`：可选 `opus`（复杂逻辑）、`sonnet`（代码审查/文档）、`haiku`（测试/简单修改）
> - `isolation: "worktree"`：如需各代理独立工作互不干扰，可启用 worktree 隔离
> - 多个 Agent Tool 在同一消息中并行调用即可同时执行

3. **收集结果**：等待所有代理完成
4. **对比评估**：AI 从以下维度对比：
   - 功能完整性
   - 代码质量（可读性、可维护性）
   - 性能表现
   - 边界情况处理
   - 安全性
5. **记录结果**：保存到多维表格
6. **通知用户**：发送飞书消息

### 评估输出格式

```
## 竞赛结果：{任务描述}

### 方案 A（{代理名称}）
- 优点：{列表}
- 缺点：{列表}
- 评分：{分}/10

### 方案 B（{代理名称}）
- 优点：{列表}
- 缺点：{列表}
- 评分：{分}/10

### 推荐：方案 {X}
理由：{详细理由}
```

---

## 模式二：分工模式

### 适用场景
- 有多个独立的任务需要并行处理
- 任务之间没有依赖关系
- 需要快速完成多个小任务

### 工作流

```
用户描述多个任务
      │
      ▼
AI 解析并拆分任务列表
      │
      ├──► Agent 1 ──► 任务 A（修 bug）
      ├──► Agent 2 ──► 任务 B（写新页面）
      └──► Agent 3 ──► 任务 C（优化查询）
                                        │
                                        ▼
                                  全部完成后汇总
                                        │
                                        ▼
                                  飞书消息通知
```

### 执行步骤

1. **拆分任务**：AI 分析用户描述，拆分为独立的子任务
2. **分配代理**：每个子任务分配一个代理
3. **并行执行**：使用 Agent Tool 同时启动所有子代理

**Agent Tool 调用示例（分工模式）：**

```
Agent(subagent_type="general-purpose", description="修bug", prompt="修复 {repo_path} 中 src/api/login.ts 的登录超时问题")
Agent(subagent_type="general-purpose", description="写新页面", prompt="在 {repo_path} 中创建 src/pages/dashboard.tsx 仪表盘页面")
Agent(subagent_type="general-purpose", description="优化查询", prompt="优化 {repo_path} 中 src/db/queries.ts 的数据库查询性能")
```

4. **收集汇总**：等待所有代理完成
5. **记录和通知**

### 任务拆分原则

- 任务之间无数据依赖
- 每个任务可独立验证
- 任务粒度适中（预计 5-30 分钟完成）

---

## 模式三：流水线模式

### 适用场景
- 复杂功能需要多阶段处理
- 需要代码审查和质量保障
- 标准化的开发流程

### 标准流水线

```
阶段 1: 编写代码（Claude Opus 4.6）
  │  - 编写功能实现
  │  - 编写基础测试
  │
  ▼
阶段 2: 代码审查（Claude Sonnet 4.6）
  │  - 审查代码质量
  │  - 检查潜在 bug
  │  - 提出改进建议
  │
  ▼
阶段 3: 编写测试 + 集成（Claude Haiku 4.5）
  │  - 补充测试用例
  │  - 运行测试验证
  │  - 修复发现的问题
  │
  ▼
汇总报告
```

### 自定义流水线

用户可以自定义流水线的阶段和代理分配：
- "第一阶段用 Opus 写前端，第二阶段用 Sonnet 做 UI 审查"

### 执行步骤

1. **定义流水线**：确定阶段、每个阶段的代理、阶段间的输入输出
2. **顺序执行**：每个阶段完成后，将结果传递给下一个阶段

**Agent Tool 调用示例（流水线模式）：**

```
# 阶段1: 编写代码（Opus）
Agent(subagent_type="general-purpose", model="opus", description="阶段1-编写代码", prompt="在 {repo_path} 中实现功能：{任务描述}。编写功能代码和基础测试。")

# 阶段2: 代码审查（Sonnet）— 等阶段1完成后执行
Agent(subagent_type="code-reviewer", model="sonnet", description="阶段2-代码审查", prompt="审查 {repo_path} 中刚刚实现的代码。检查代码质量、潜在 bug、改进建议。")

# 阶段3: 补充测试（Haiku）— 等阶段2完成后执行
Agent(subagent_type="general-purpose", model="haiku", description="阶段3-测试集成", prompt="在 {repo_path} 中为已审查通过的代码补充测试用例并运行测试。")
```

> **关键**：流水线模式下，Agent 必须按顺序执行（后一阶段的 prompt 中需包含前一阶段的结果）。

3. **质量门控**：每个阶段可以"通过"或"打回"（前一阶段需要修改）
   - 如果代码审查发现严重问题 → 将审查意见传回阶段 1 的代理重新修改
   - 打回最多 2 轮，超过 2 轮直接向用户报告，由用户决定如何处理
4. **最终汇总**：所有阶段完成后生成汇总报告

---

## 任务管理

### 查看任务列表

```bash
lark-cli base +record-list --table-id "<tasks_table_id>" --filter '...状态 != "已完成" AND 状态 != "已取消"'
```

### 更新任务状态

```bash
lark-cli base +record-upsert --table-id "<tasks_table_id>" --json '{
  "任务描述": "...",
  "状态": "已完成",
  "结果摘要": "...",
  "完成时间": "..."
}'
```

### 创建飞书任务追踪

对于重要的开发任务，同时创建飞书任务：

```bash
lark-cli task +create --summary "[多代理] {任务描述}" --description "工作模式：{模式}\n分配代理：{代理列表}"
```

---

## 通知

### 任务完成通知

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --markdown "..."
```

通知格式：

**竞赛模式：**
```
竞赛完成！

任务：{任务描述}
参与代理：{N} 个
总耗时：{X} 秒

推荐方案：{代理名称}（评分 {分}/10）
理由：{一句话}

查看详细对比：{多维表格链接}
```

**分工模式：**
```
分工任务全部完成！

总任务数：{N} 个
总耗时：{X} 秒（并行加速 {倍数}x）

| 任务 | 代理 | 状态 | 耗时 |
|------|------|------|------|
| 任务A | Agent1 | 完成 | {X}s |
| 任务B | Agent2 | 完成 | {Y}s |
| 任务C | Agent3 | 完成 | {Z}s |
```

**流水线模式：**
```
流水线执行完成！

任务：{任务描述}
总耗时：{X} 秒

| 阶段 | 代理 | 状态 | 备注 |
|------|------|------|------|
| 编写代码 | Opus 4.6 | 通过 | — |
| 代码审查 | Sonnet 4.6 | 通过 | 3个改进建议 |
| 测试集成 | Haiku 4.5 | 通过 | 12个测试通过 |

查看完整报告：{多维表格链接}
```

---

## GitHub 仓库集成

本 Skill 的代理默认在当前工作目录下操作代码。如需操作 GitHub 仓库：

### 指定仓库路径

在 Agent Tool 的 `prompt` 中明确指定仓库路径：

```
Agent(description="修复bug", prompt="在 D:/projects/my-app 仓库中修复...")
```

### Git 操作

Agent Tool 的子代理可以使用 git 命令进行版本控制：

```bash
# 创建分支（各代理独立分支，避免冲突）
git checkout -b agent/contest-a

# 提交代码
git add src/
git commit -m "feat: 实现XX功能（竞赛方案A）"

# 推送到远程
git push -u origin agent/contest-a
```

### 创建 Pull Request

代理完成代码后，可通过 `gh` CLI 创建 PR：

```bash
# 创建 PR
gh pr create --title "feat: XX功能实现" --body "$(cat <<'EOF'
## 方案说明
{代理生成的方案描述}

## 变更内容
- {变更列表}
EOF
)"
```

> **注意**：推送到远程仓库和创建 PR 会影响共享状态，需征得用户确认后再执行。

### Worktree 隔离模式

使用 `isolation: "worktree"` 让各代理在独立 git worktree 中工作，互不干扰：

```
Agent(subagent_type="general-purpose", isolation="worktree", description="竞赛方案A", prompt="...")
Agent(subagent_type="general-purpose", isolation="worktree", description="竞赛方案B", prompt="...")
```

完成对比后，保留优胜方案的 worktree，删除其他方案。

---

## 使用建议

1. **简单任务**（单一功能、明确需求）→ 竞赛模式，2 个代理即可
2. **批量任务**（多个独立 bug、多个小功能）→ 分工模式，按任务数量分配代理
3. **复杂功能**（新模块、核心功能）→ 流水线模式，确保质量
4. **代理选择建议**：
   - 复杂逻辑、架构设计 → Opus
   - 代码审查、文档 → Sonnet
   - 测试、简单修改 → Haiku

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `im +messages-send` | `im:message:send_as_bot` |
| `task +create` | `task:task:write` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
- [`lark-task`](../lark-task/SKILL.md) — `+create` 详细用法
