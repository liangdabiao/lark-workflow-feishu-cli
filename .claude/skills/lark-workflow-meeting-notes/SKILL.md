---
name: lark-workflow-meeting-notes
version: 1.0.0
description: "智能会议纪要分发：获取已完成会议的纪要内容，根据参会状态智能分发——完整纪要发给参会者，摘要版发给缺席者。当用户需要'分发会议纪要'、'发送纪要'、'会议总结分发'、'会后通知'、'纪要发给没参加的人'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 智能会议纪要分发工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "会议结束了，帮我整理纪要发给大家" / "分发会议纪要"
- "把今天的会议记录发给没参加的人" / "会议总结分发"
- "帮我发一下刚才那个会的纪要" / "发送纪要"
- "会后通知参会人" / "会后通知"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,vc,docs,im
```

## 工作流

```
{event_id 或 会议名称/时间}
  │
  ├── calendar events get ──────────► 获取会议详情 + 参会人列表
  │
  ├── [有纪要] docs +fetch ─────────► 读取纪要内容
  │     或 vc 获取纪要文档
  │
  ├── AI 区分参会人 ────────────────► 出席者 / 缺席者
  │
  ├── AI 生成摘要版本 ──────────────► 缺席者版（摘要+决策+待办）
  │
  ├── im +messages-send ────────────► 完整纪要 → 出席者
  │
  └── im +messages-send ────────────► 摘要 + 行动项 → 缺席者
```

---

## Step 1: 获取会议详情

如果用户提供了 event_id，直接查询；否则先通过日程列表查找：

```bash
# 直接查询
lark-cli calendar events get --params '{"calendar_id":"primary","event_id":"<event_id>","need_attendee":true}' --as user
```

从返回结果中提取：
- `summary` — 会议主题
- `start_time` / `end_time` — 会议时间
- 参会人列表和 RSVP 状态

### 获取参会人状态

```bash
lark-cli calendar event.attendees list \
  --params '{"calendar_id":"primary","event_id":"<event_id>","page_size":50}' \
  --as user
```

根据 `rsvp_status` 分类：
- `accept` → 出席者
- `tentative` → 待定（按缺席处理）
- `decline` / `needs_action` → 缺席者

## Step 2: 获取会议纪要

### 方式一：从飞书文档获取

如果会议关联了纪要文档，使用 `docs +fetch`：

```bash
lark-cli docs +fetch --doc "<doc_url 或 token>" --as user
```

### 方式二：从妙记/视频会议获取

如果会议有视频会议记录，可以通过 vc 获取纪要：

```bash
# 查询会议记录
lark-cli vc --help  # 查看可用命令
```

### 方式三：用户提供纪要内容

如果用户直接提供了纪要文本或要求 AI 根据会议主题生成纪要草稿，直接使用用户提供的内容。

> **注意**：如果会议没有纪要文档，告知用户"该会议暂无纪要文档"，并询问是否需要根据会议主题生成纪要草稿。

## Step 3: AI 整理纪要内容

### 完整版（给出席者）

保留纪要的全部内容，整理为结构化格式。

### 摘要版（给缺席者）

AI 从完整纪要中提取关键信息：

```markdown
## {会议主题} — 会议摘要

**时间**：{日期时间}
**参会人**：{N} 人出席，{N} 人缺席

### 关键决策
- {决策1}
- {决策2}

### 待办事项
| 事项 | 负责人 | 截止日期 |
|------|--------|---------|

### 重要讨论
- {讨论要点1}
- {讨论要点2}

> 完整纪要已上传至飞书文档：{链接}
```

## Step 4: 分发纪要

### 给出席者（完整版）

```bash
lark-cli im +messages-send \
  --user-id "<attendee_open_id>" \
  --markdown "{完整纪要内容}" \
  --as bot
```

### 给缺席者（摘要版）

```bash
lark-cli im +messages-send \
  --user-id "<absentee_open_id>" \
  --markdown "{摘要版内容}" \
  --as bot
```

> **注意**：发送消息时每次间隔 1-2 秒，避免触发频率限制。如果参会人数超过 20 人，分批发送。

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 找不到会议 | 提示用户提供 event_id 或会议时间 |
| 无纪要文档 | 询问用户是否需要生成纪要草稿 |
| 参会人信息获取失败 | 直接发送给发起人，让发起人手动转发 |
| 消息发送频率限制 | 分批发送（每批 10 人，间隔 2 秒） |
| 某人发送失败 | 记录失败列表，最后汇总告知用户 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `calendar events get` | `calendar:calendar:readonly`, `calendar:event:readonly` |
| `calendar event.attendees list` | `calendar:calendar:readonly` |
| `docs +fetch` | `docx:document:readonly` |
| `im +messages-send` | `im:message` |

## 参考

- [`../lark-calendar/SKILL.md`](../lark-calendar/SKILL.md) — 日历原子操作
- [`../lark-vc/SKILL.md`](../lark-vc/SKILL.md) — 视频会议
- [`../lark-doc/SKILL.md`](../lark-doc/SKILL.md) — 文档操作
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 消息发送
