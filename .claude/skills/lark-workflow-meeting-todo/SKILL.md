---
name: lark-workflow-meeting-todo
version: 1.0.0
description: "会议待办自动追踪：从飞书妙记中提取待办事项，通过飞书消息确认后创建飞书任务，定期检查完成情况。当用户需要'会议待办'、'会后的待办'、'追踪待办'、'提取会议行动项'、'整理会议纪要待办'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 会议待办自动追踪工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我整理最近会议的待办事项" / "会议后有什么要做的"
- "提取这个会议的行动项" / "会上答应了什么事"
- "检查会议待办的完成情况" / "会议承诺兑现了没"
- "把会议待办同步到飞书任务"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain vc,task,im
lark-cli auth login --domain vc,drive   # 读取纪要文档正文
```

## 工作流

```
{时间范围}
  │
  ├── vc +search ──────────────► 会议列表（meeting_ids）
  │
  ├── vc +notes ───────────────► 纪要文档 tokens
  │
  ├── docs +fetch ─────────────► 读取纪要文档内容（提取待办）
  │     或
  ├── minutes minutes get ─────► 读取妙记 AI 产物（待办、总结）
  │
  ▼
AI 分析提取待办列表
  ├── 我方待办（我答应做的）
  └── 对方待办（对方承诺做的）
          │
          ▼
im +messages-send ──► 发送确认请求到用户
          │
          ▼
task +create ──► 确认后创建飞书任务
          │
          ▼
base +record-upsert ──► 记录追踪状态
```

---

## 模式一：批量提取（指定时间范围）

### Step 1: 确定时间范围

默认**过去 7 天**。推断规则："今天"→当天，"这周"→本周一~now，"昨天"→昨天整天。

> 日期转换必须调用系统命令（如 `date`），不要心算。

### Step 2: 查询会议记录

```bash
lark-cli vc +search --start "<YYYY-MM-DD>" --end "<YYYY-MM-DD>" --format json --page-size 30
```

- 搜索时间范围最大为 1 个月，超出需拆分查询
- 有 `page_token` 时必须继续翻页，收集所有 `id` 字段
- 记录每个会议的 `meeting_id`、`subject`、`start_time`

### Step 3: 获取纪要产物

```bash
# 批量获取会议纪要信息（单次最多 50 个 meeting_id）
lark-cli vc +notes --meeting-ids "id1,id2,...,idN"
```

返回结果包含：
- `note_doc_token` — 纪要文档 token（含 AI 总结、待办）
- `verbatim_doc_token` — 逐字稿文档 token
- `minute_token` — 妙记 token

对于每个会议纪要，提取待办内容：

**方式 A：从纪要文档中提取（推荐，内容更丰富）**
```bash
lark-cli docs +fetch --doc "<note_doc_token>"
```

**方式 B：从妙记 AI 产物中提取（更快）**
```bash
lark-cli minutes minutes get --params '{"minute_token": "<minute_token>"}'
lark-cli vc +notes --minute-tokens "<minute_token>"
```

### Step 4: AI 分析提取待办

阅读纪要文档内容后，AI 从中提取两类待办：

**我方待办（我答应别人要做的）：**
- 关键词模式："我会..."、"我负责..."、"我来跟进..."、"回头我发给你..."、"我回头发邮件..."
- 提取：待办内容、涉及对象、截止暗示

**对方待办（别人答应要做的）：**
- 关键词模式："他说..."、"对方承诺..."、"XX 会提供..."、"下周给回复..."
- 提取：待办内容、承诺人、截止暗示

**输出格式：**

```
## 会议待办提取结果

### 会议：{会议主题}（{日期}）

#### 我方待办
| # | 待办内容 | 涉及对象 | 建议截止日 |
|---|---------|---------|-----------|
| 1 | 把报告发给张三 | 张三 | 本周五 |

#### 对方待办
| # | 待办内容 | 承诺人 | 建议截止日 |
|---|---------|--------|-----------|
| 1 | 提供设计稿 | 李四 | 下周三 |
```

### Step 5: 发送确认请求

将提取的待办以消息发送给用户确认：

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --msg-type interactive --content '<飞书卡片JSON>'
```

卡片内容包含：
- 每条待办的简述
- "确认创建任务" / "跳过" 按钮
- "全部确认" / "全部跳过" 快捷操作

### Step 6: 创建飞书任务

用户确认后，为每条待办创建飞书任务：

```bash
lark-cli task +create --summary "<待办内容>" --due "<截止日>" --description "来源会议：{会议主题}（{日期}）"
```

### Step 7: 记录追踪状态

将待办信息记录到多维表格（需先初始化 Base 表，见"数据初始化"部分）：

```bash
lark-cli base +record-upsert --table-id "<table_id>" --json '{"字段名":"值"}'
```

---

## 模式二：单会议提取

用户指定某个会议时：

```bash
# 通过会议标题搜索
lark-cli vc +search --keyword "<会议主题>" --start "<date>" --end "<date>"
```

后续流程与模式一相同，但只处理这一个会议。

---

## 模式三：检查完成情况

### Step 1: 获取追踪记录

```bash
lark-cli base +record-list --table-id "<table_id>" --filter '...确认状态 = "已确认"'
```

### Step 2: 检查任务状态

```bash
lark-cli task +get-my-tasks --query "<待办关键词>"
```

### Step 3: 验证"发邮件"类承诺

如果待办内容涉及"发邮件"，检查邮件是否已发送：

```bash
lark-cli mail +messages-search --query "<邮件关键词>" --from "me"
```

> **降级说明**：如果用户未启用飞书邮箱（`mail` 命令返回错误码 1230003），邮件验证步骤跳过，在追踪状态中标注"邮件验证不可用，需人工确认"。

### Step 4: 更新追踪状态

```bash
lark-cli base +record-upsert --table-id "<table_id>" --json '{"字段名":"值"}'
```

---

## 模式四：自动归档

超过 14 天未完成的待办自动标记为"已归档"：

```bash
lark-cli base +record-list --table-id "<table_id>" --filter '...确认状态 = "已确认" AND 创建时间 < 14天前'
```

对每条记录更新状态为"已归档"，并通过飞书消息提醒用户。

---

## 数据初始化

首次使用时，创建多维表格追踪数据：

### Base 表结构

**表名：会议待办追踪**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 会议主题 | 文本 | 来源会议的标题 |
| 会议日期 | 日期 | 会议时间 |
| 待办内容 | 文本 | 提取的待办描述 |
| 待办类型 | 单选 | 我方待办 / 对方待办 |
| 负责人/承诺人 | 文本 | 谁来做 |
| 建议截止日 | 日期 | 根据上下文推断 |
| 确认状态 | 单选 | 待确认 / 已确认 / 已跳过 / 已完成 / 已归档 |
| 关联任务ID | 文本 | 飞书任务的 task_id |
| 会议ID | 文本 | vc meeting_id |
| 创建时间 | 日期 | 记录创建时间 |

### 初始化命令

```bash
# 1. 创建多维表格
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "会议待办追踪" --fields '[
  {"name":"会议主题","type":"text"},
  {"name":"会议日期","type":"datetime"},
  {"name":"待办内容","type":"text"},
  {"name":"待办类型","type":"select","multiple":false,"options":[{"name":"我方待办"},{"name":"对方待办"}]},
  {"name":"负责人/承诺人","type":"text"},
  {"name":"建议截止日","type":"datetime"},
  {"name":"确认状态","type":"select","multiple":false,"options":[{"name":"待确认"},{"name":"已确认"},{"name":"已跳过"},{"name":"已完成"},{"name":"已归档"}]},
  {"name":"关联任务ID","type":"text"},
  {"name":"会议ID","type":"text"},
  {"name":"创建时间","type":"datetime"}
]'

# 2. 记录 base_token 和 table_id 到后续使用
```

> 首次使用时向用户确认是否创建追踪表。如果用户已有 Base，可以指定已有 Base URL。
>
> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 学习模式

用户多次拒绝某类待办时，记录拒绝模式：

- 在多维表中增加"用户偏好"记录
- 下次提取时，AI 自动过滤相似类型的待办
- 示例："会议纪要类记录"被多次拒绝 → 下次自动跳过"记录会议纪要"类待办

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `vc +search` | `vc:meeting:read` |
| `vc +notes` | `vc:meeting:read` |
| `minutes.minutes.get` | `minutes:minutes:readonly` |
| `docs +fetch` | `docx:document:read` |
| `task +create` | `task:task:write` |
| `task +get-my-tasks` | `task:task:read` |
| `im +messages-send` | `im:message:send_as_bot` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `mail +messages-search` | `mail:mail:read` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-vc`](../lark-vc/SKILL.md) — `+search`、`+notes` 详细用法
- [`lark-minutes`](../lark-minutes/SKILL.md) — 妙记基础信息
- [`lark-doc`](../lark-doc/SKILL.md) — `+fetch` 详细用法
- [`lark-task`](../lark-task/SKILL.md) — `+create`、`+get-my-tasks` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
- [`lark-base`](../lark-base/SKILL.md) — `+record-list`、`+record-upsert` 详细用法
- [`lark-mail`](../lark-mail/SKILL.md) — `+messages-search` 详细用法
- [`lark-workflow-meeting-summary`](../lark-workflow-meeting-summary/SKILL.md) — 会议纪要汇总（可联动）
