---
name: lark-workflow-chat-digest
version: 1.0.0
description: "群消息日报生成器：拉取指定群聊在时间范围内的消息记录，AI 自动提取关键信息，生成结构化日报并可选保存为文档或发送到群聊。当用户需要'群日报'、'群消息摘要'、'生成日报'、'群聊总结'、'今日群消息回顾'、'帮我总结一下群里聊了什么'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 群消息日报生成器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我总结一下今天项目群聊了什么" / "群日报"
- "生成XX群的周报" / "群消息摘要"
- "这个群今天有什么重要消息" / "群聊总结"
- "帮我看看今天群里有没有重要通知" / "今日群消息回顾"
- "生成本周产品群的讨论摘要"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain im
# 如需保存文档，还需：
lark-cli auth login --domain im,docs
```

## 工作流

```
{群名称/群ID + 时间范围}
  │
  ├── im +chat-search ──────────────► 查找群聊（获取 chat_id）
  │
  ├── im +chat-messages-list ───────► 拉取群消息（分页，max 50/页）
  │     └── --start / --end 过滤时间范围
  │
  ├── AI 分析消息 ──────────────────► 提取关键信息
  │     ├── 话题分类
  │     ├── 决策记录
  │     ├── 待办事项
  │     ├── 问题与风险
  │     └── 重要通知
  │
  ├── AI 生成结构化日报 ────────────► Markdown 格式
  │
  ├── [可选] docs +create ──────────► 保存为飞书文档
  │
  └── [可选] im +messages-send ─────► 发送到群聊
```

---

## Step 1: 查找群聊

如果用户提供了群名称，先获取 chat_id：

```bash
lark-cli im +chat-search --query "<群名称>" --as user
```

从返回结果中提取 `chat_id`（`oc_xxx` 格式）。如果用户直接提供了 chat_id，跳过此步。

## Step 2: 拉取群消息

```bash
# 拉取指定时间范围内的消息（按时间倒序，最新在前）
lark-cli im +chat-messages-list \
  --chat-id "<chat_id>" \
  --start "2026-04-09T00:00:00+08:00" \
  --end "2026-04-09T23:59:59+08:00" \
  --page-size "50" \
  --sort "asc" \
  --as user
```

### 分页处理

每页最多 50 条消息。如果返回结果中有 `has_more: true`，用 `--page-token` 继续拉取：

```bash
# 第一页
lark-cli im +chat-messages-list --chat-id "oc_xxx" --start "..." --end "..." --page-size "50" --as user
# 返回 page_token 后，拉取下一页
lark-cli im +chat-messages-list --chat-id "oc_xxx" --start "..." --end "..." --page-size "50" --page-token "<token>" --as user
```

> **注意**：
> - 如果一天的消息量超过 500 条，建议只分析最近 200-300 条以避免超出上下文
> - 过滤掉纯表情回复、系统消息、图片/文件消息（除非用户特别要求）
> - 用 `--sort asc` 按时间正序排列，便于 AI 理解讨论脉络

## Step 3: AI 生成日报

将拉取到的消息交给 AI 分析，按以下结构生成日报：

```markdown
## {群名} 日报 — {日期}

### 概览
- 消息总数：N 条
- 活跃成员：N 人
- 主要话题：{2-3个关键词}

### 重要决策
| 决策内容 | 提出者 | 时间 |
|---------|--------|------|

### 待办事项
| 事项 | 负责人 | 来源 |
|------|--------|------|

### 话题讨论
#### 1. {话题标题}
- **摘要**：一句话概括
- **参与人**：A、B、C
- **结论**：{如有}

#### 2. {话题标题}
...

### 问题与风险
| 问题描述 | 提出者 | 状态 |
|---------|--------|------|

### 其他通知
- {通知1}
- {通知2}
```

**分析规则：**
- 忽略无意义的闲聊和纯表情消息
- 重点关注包含决策、待办、问题的消息
- 保留关键上下文，但去除冗余讨论
- 如果消息中包含 `@某人` 的提醒，在待办中标注

## Step 4: 输出日报（可选）

### 发送到群聊

```bash
lark-cli im +messages-send \
  --chat-id "<chat_id>" \
  --markdown "{日报内容}" \
  --as bot
```

### 发送给个人

```bash
lark-cli im +messages-send \
  --user-id "<open_id>" \
  --markdown "{日报内容}" \
  --as bot
```

### 保存为文档

```bash
lark-cli docs +create \
  --title "{群名} 日报 — {日期}" \
  --markdown "{日报内容}" \
  --as user
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 找不到群聊 | 提示用户确认群名称，或直接使用 chat_id |
| 群内无消息 | 返回"该时段内群内无消息" |
| 消息量过大 | 只分析最近 N 条，其余折叠为"N 条消息未分析" |
| im 推送失败 | 仅输出到对话，告知用户手动发送 |
| docs 保存失败 | 仅输出 Markdown 到对话 |
| 消息内容为图片/文件 | 在日报中标注"[图片/文件]"，建议用户手动查看 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `im +chat-search` | `im:chat:readonly` |
| `im +chat-messages-list` | `im:message:readonly` |
| `im +messages-send` | `im:message` |
| `docs +create` | `docx:document:create` |

## 参考

- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — IM 原子操作
- [`../lark-doc/SKILL.md`](../lark-doc/SKILL.md) — 文档创建
