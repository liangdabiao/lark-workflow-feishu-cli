---
name: lark-workflow-task-digest
version: 1.0.0
description: "任务状态播报员：查询任务列表并按状态分类统计，生成进度报告，通过飞书消息推送或保存为文档。当用户需要'任务进度'、'任务报告'、'任务状态'、'待办汇报'、'项目进度报告'、'任务概览'、'今日任务完成情况'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 任务状态播报员工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "今天任务完成情况怎么样" / "任务进度"
- "生成本周任务周报" / "任务报告"
- "看看我有哪些待办" / "待办汇报"
- "项目进度报告" / "任务概览"
- "帮我播报一下任务状态"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain task
# 如需推送消息或保存文档，还需：
lark-cli auth login --domain task,im,docs
```

## 工作流

```
{时间范围 / 状态筛选}
  │
  ├── task +get-my-tasks [--page-all] ──► 任务列表
  │
  ├── AI 分类统计 ─────────────────────► 按状态/优先级/截止日分组
  │     ├── 未开始
  │     ├── 进行中
  │     ├── 已完成
  │     └── 已逾期（高亮）
  │
  ├── AI 生成进度报告 ─────────────────► 结构化 Markdown
  │
  ├── [可选] im +messages-send ────────► 推送到飞书
  │
  └── [可选] docs +create ─────────────► 保存为文档
```

---

## Step 1: 确定时间范围

根据用户请求确定查询范围：

| 用户说 | 范围 |
|--------|------|
| 今天 / 今日 | 当天 |
| 本周 | 本周一 ~ 今天 |
| 本月 | 本月 1 号 ~ 今天 |

> **注意**：日期计算使用系统命令 `date`，不要心算。

```bash
# 获取今天的截止范围（示例）
date -d "today 23:59:59" +%s   # Unix timestamp
```

## Step 2: 获取任务列表

```bash
# 默认：返回分配给当前用户的未完成任务（最多 20 条）
lark-cli task +get-my-tasks

# 获取全部未完成任务（超过 20 条时）
lark-cli task +get-my-tasks --page-all

# 按截止日期筛选（推荐，减少数据量）
lark-cli task +get-my-tasks --due-end "2026-04-09T23:59:59+08:00"

# 按创建时间筛选
lark-cli task +get-my-tasks --created-at "2026-04-01"

# 包含已完成的任务
lark-cli task +get-my-tasks --complete

# 搜索特定任务
lark-cli task +get-my-tasks --query "<关键词>"
```

> **注意**：不带过滤条件时可能返回大量历史待办，建议用 `--due-end` 过滤。如果数据量仍过大（超过上下文限制），AI 汇总时只展示**近 30 天内创建的**，其余折叠为"其他 N 项历史待办"。

## Step 3: AI 生成进度报告

将 Step 2 的结果按以下结构整理：

```markdown
## {日期范围} 任务进度报告

### 统计概览
| 状态 | 数量 |
|------|------|
| 未开始 | N |
| 进行中 | N |
| 已完成 | N |
| 已逾期 | N |
| **总计** | **N** |

### 逾期任务（需关注）
| 任务 | 负责人 | 截止日期 | 逾期天数 |
|------|--------|---------|---------|

### 进行中任务
| 任务 | 负责人 | 截止日期 | 进度 |
|------|--------|---------|------|

### 已完成任务
| 任务 | 完成时间 |
|------|---------|
```

**统计规则：**
- 逾期 = 截止日期 < 当前时间 且 状态 != 已完成
- 完成率 = 已完成 / 总计 × 100%
- 逾期任务**高亮显示**（用红色标注）

## Step 4: 推送报告（可选）

### 发送到飞书群

```bash
lark-cli im +messages-send \
  --chat-id "<chat_id>" \
  --markdown "{报告内容}" \
  --as bot
```

### 发送给个人

```bash
lark-cli im +messages-send \
  --user-id "<open_id>" \
  --markdown "{报告内容}" \
  --as bot
```

### 保存为文档

```bash
lark-cli docs +create \
  --title "任务进度报告 ({日期})" \
  --markdown "{报告内容}" \
  --as user
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 无任务数据 | 返回"当前没有符合条件的任务" |
| 数据量过大 | 只展示近 30 天数据，历史数据折叠 |
| im 推送失败 | 仅输出到对话，告知用户手动发送 |
| docs 保存失败 | 仅输出 Markdown 到对话 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `task +get-my-tasks` | `task:task:readonly` |
| `im +messages-send` | `im:message` |
| `docs +create` | `docx:document:create` |

## 参考

- [`../lark-task/SKILL.md`](../lark-task/SKILL.md) — 任务管理原子操作
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 消息发送
- [`../lark-doc/SKILL.md`](../lark-doc/SKILL.md) — 文档创建
