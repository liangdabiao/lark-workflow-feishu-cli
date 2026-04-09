---
name: lark-workflow-form-builder
version: 1.0.0
description: "智能表单收集器：创建多维表格和数据表单，AI 根据需求自动设计字段，配置表单问题，生成填写链接，分发到飞书群聊收集信息。当用户需要'创建表单'、'信息收集'、'制作问卷'、'收集反馈'、'报名表'、'数据收集'、'周报表单'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 智能表单收集器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我创建一个需求收集表单" / "创建表单"
- "收集一下大家的周报" / "信息收集"
- "制作一个报名表单" / "报名表"
- "帮我做一个反馈问卷" / "制作问卷"
- "创建一个会议报名表" / "数据收集"

## 前置条件

需要 **user 身份**（创建表格/表单）和 **bot 身份**（发消息分发表单）。

```bash
lark-cli auth login --domain base,im
```

## 工作流

```
{表单主题 + 字段需求}
  │
  ├── AI 设计表单结构 ─────────────► 根据主题生成字段方案（用户确认）
  │
  ├── base +table-create ──────────► 创建数据表 + 字段
  │     └── [降级] +field-create 逐个创建字段
  │
  ├── base +form-create ───────────► 创建表单视图
  │
  ├── base +form-questions-create ─► 配置表单问题（max 10/次）
  │
  ├── base +form-get ──────────────► 获取表单链接
  │
  └── im +messages-send ───────────► 分发表单链接到群聊
```

---

## Step 1: AI 设计表单结构

根据用户描述的表单用途，AI 自动设计字段方案。例如用户说"创建一个需求收集表单"，生成以下方案：

```json
[
  {"name": "需求标题", "type": "text"},
  {"name": "提出人", "type": "text"},
  {"name": "优先级", "type": "select", "options": ["高", "中", "低"]},
  {"name": "需求描述", "type": "text"},
  {"name": "期望上线日期", "type": "datetime"},
  {"name": "附件", "type": "attachment"}
]
```

**与用户确认字段方案后再执行创建操作。**

## Step 2: 创建数据表

### 方式一：一步创建（含字段）

```bash
lark-cli base +table-create \
  --base-token "<base_token>" \
  --name "<表名>" \
  --fields '[
    {"name":"需求标题","type":"text"},
    {"name":"提出人","type":"text"},
    {"name":"优先级","type":"select","multiple":false,"options":[{"name":"高"},{"name":"中"},{"name":"低"}]},
    {"name":"需求描述","type":"text"},
    {"name":"期望上线日期","type":"datetime"}
  ]' \
  --as user
```

### 方式二：降级（表创建成功但字段失败时）

如果 `+table-create` 部分成功（表创建但字段创建失败，错误码 800004135）：

```bash
# 1. 获取表 ID
lark-cli base +table-list --base-token "<base_token>" --as user

# 2. 逐个创建字段（间隔 1-2 秒）
lark-cli base +field-create \
  --base-token "<base_token>" \
  --table-id "<table_id>" \
  --json '{"field_name":"需求标题","type":1}' \
  --as user
sleep 1

lark-cli base +field-create \
  --base-token "<base_token>" \
  --table-id "<table_id>" \
  --json '{"field_name":"提出人","type":1}' \
  --as user
sleep 1
```

## Step 3: 创建表单视图

```bash
lark-cli base +form-create \
  --base-token "<base_token>" \
  --table-id "<table_id>" \
  --name "<表单名称>" \
  --description "请填写以下信息，提交后将自动记录到数据表中。" \
  --as user
```

从返回结果中提取 `form_id`。

## Step 4: 配置表单问题

每次最多创建 10 个问题。超过 10 个时分批创建。

```bash
lark-cli base +form-questions-create \
  --base-token "<base_token>" \
  --table-id "<table_id>" \
  --form-id "<form_id>" \
  --questions '[
    {"type":"text","title":"需求标题","required":true},
    {"type":"text","title":"提出人","required":true},
    {"type":"select","title":"优先级","required":true,"multiple":false,"options":[{"name":"高","hue":"Red"},{"name":"中","hue":"Yellow"},{"name":"低","hue":"Green"}]},
    {"type":"text","title":"需求描述","required":true},
    {"type":"text","title":"期望上线日期","required":false}
  ]' \
  --as user
```

**支持的 question type：**

| type | 说明 | 可选 style |
|------|------|-----------|
| `text` | 单行文本 | `plain`, `phone`, `url`, `email` |
| `number` | 数字 | `rating`（评分） |
| `select` | 单选/多选 | — |
| `datetime` | 日期时间 | `format: "yyyy/MM/dd"` |
| `user` | 选择人员 | — |
| `attachment` | 附件上传 | — |
| `location` | 位置 | — |

**评分字段示例：**
```json
{"type":"number","title":"满意度评分","style":{"type":"rating","icon":"star","min":1,"max":5},"required":true}
```

## Step 5: 获取表单链接

```bash
lark-cli base +form-get \
  --base-token "<base_token>" \
  --table-id "<table_id>" \
  --form-id "<form_id>" \
  --as user
```

从返回结果中提取表单的分享链接（`share_url` 或 `form_url`）。

## Step 6: 分发表单链接

### 发送到群聊

```bash
lark-cli im +messages-send \
  --chat-id "<chat_id>" \
  --markdown "## {表单名称}

{表单描述}

请点击以下链接填写：{表单链接}

截止时间：{截止日期}" \
  --as bot
```

### 发送给个人

```bash
lark-cli im +messages-send \
  --user-id "<open_id>" \
  --markdown "请填写 {表单名称}：{表单链接}" \
  --as bot
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| +table-create 字段失败 | 降级为 +field-create 逐个创建 |
| +form-create 失败 | 检查 base_token 和 table_id 是否正确 |
| 问题超过 10 个 | 分批创建，每次 10 个 |
| base_token 未提供 | 询问用户提供，或用 +base-create 新建 |
| 分发失败 | 输出表单链接让用户手动分享 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +table-create` | `bitable:app` |
| `base +field-create` | `bitable:app` |
| `base +form-create` | `bitable:app` |
| `base +form-questions-create` | `bitable:app` |
| `base +form-get` | `bitable:app:readonly` |
| `im +messages-send` | `im:message` |

## 参考

- [`../lark-base/SKILL.md`](../lark-base/SKILL.md) — 多维表格原子操作
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 消息发送
