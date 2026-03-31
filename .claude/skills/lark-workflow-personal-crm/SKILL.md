---
name: lark-workflow-personal-crm
version: 1.0.0
description: "个人CRM：自动扫描飞书邮箱和日历提取联系人，多维表格维护联系人档案和关系热度分，支持自然语言查询和超期提醒。当用户需要'联系人管理'、'CRM'、'谁联系过我'、'多久没联系'、'联系人热度'、'关系管理'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 个人 CRM 工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "上个月谁联系过我？" / "最近和谁有互动"
- "XX 公司的人我最后一次聊过什么？" / "我和张三的关系怎么样"
- "有没有超过 3 个月没联系的重要关系？" / "该跟进谁了"
- "帮我建立联系人档案" / "从邮箱/日历导入联系人"
- "这个人的热度分是多少" / "更新联系人信息"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain mail,calendar,contact,base,im
```

> **邮件模块降级说明**：如果用户未启用飞书邮箱（`mail` 命令返回错误码 1230003），则：
> - "从邮箱导入"模式不可用，跳过邮件扫描步骤
> - 改用"从日历导入"和"从通讯录补充"作为主要数据来源
> - 在简报中标注"邮箱未启用，联系人数据仅来自日历和通讯录"

## 工作流总览

```
数据导入：
  ├── mail +messages ───────► 扫描邮箱 → 提取联系人 + 互动记录
  ├── calendar +agenda ─────► 扫描日历 → 提取会议参与者 + 互动记录
  └── contact +search-user ─► 补充飞书通讯录信息

数据存储：
  └── base +record-upsert ──► 多维表格（联系人 + 互动记录）

数据查询：
  └── base +record-list ────► 自然语言查询联系人

智能提醒：
  └── base +data-query ────► 筛选冷门联系人
  └── im +messages-send ───► 飞书消息提醒
```

---

## 数据初始化

首次使用时，创建多维表格数据结构：

### Base 表结构

**表 1：联系人主表 (contacts)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 姓名 | 文本 | 联系人姓名 |
| 公司 | 文本 | 所在公司/组织 |
| 职位 | 文本 | 职位/角色 |
| 邮箱 | 文本 | 邮箱地址 |
| 手机 | 文本 | 联系方式 |
| 来源 | 单选 | 邮件/日历/通讯录/手动 |
| 关系类型 | 单选 | 客户/合作伙伴/同事/朋友/行业人脉/其他 |
| 重要程度 | 单选 | 高/中/低 |
| 热度分 | 数字 | 0-100，基于互动频率和最近互动时间计算 |
| 最后联系时间 | 日期 | 最近一次互动的时间 |
| 最后联系内容 | 文本 | 最近互动的摘要 |
| 备注 | 文本 | 自由备注 |
| 创建时间 | 日期 | 首次录入时间 |

**表 2：互动记录 (interactions)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 关联联系人 | 关联 | 关联到「联系人主表」 |
| 互动时间 | 日期 | 互动发生时间 |
| 互动类型 | 单选 | 邮件(发)/邮件(收)/会议/消息/电话/其他 |
| 互动摘要 | 文本 | 互动内容的简短描述 |
| 涉及主题 | 文本 | 讨论的话题或项目 |
| 邮件ID | 文件 | 关联的邮件 message_id（如有） |
| 会议ID | 文本 | 关联的 meeting_id（如有） |

### 初始化命令

```bash
# 1. 创建多维表格
# 注意：+table-create 可能部分成功（表创建成功但字段创建失败），需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "联系人" --fields '[
  {"name":"姓名","type":"text"},
  {"name":"公司","type":"text"},
  {"name":"职位","type":"text"},
  {"name":"邮箱","type":"text"},
  {"name":"手机","type":"text"},
  {"name":"来源","type":"select","multiple":false,"options":[{"name":"邮件"},{"name":"日历"},{"name":"通讯录"},{"name":"手动"}]},
  {"name":"关系类型","type":"select","multiple":false,"options":[{"name":"客户"},{"name":"合作伙伴"},{"name":"同事"},{"name":"朋友"},{"name":"行业人脉"},{"name":"其他"}]},
  {"name":"重要程度","type":"select","multiple":false,"options":[{"name":"高"},{"name":"中"},{"name":"低"}]},
  {"name":"热度分","type":"number"},
  {"name":"最后联系时间","type":"datetime"},
  {"name":"最后联系内容","type":"text"},
  {"name":"备注","type":"text"},
  {"name":"创建时间","type":"datetime"}
]'
lark-cli base +table-create --name "互动记录" --fields '[
  {"name":"互动时间","type":"datetime"},
  {"name":"互动类型","type":"select","multiple":false,"options":[{"name":"邮件(发)"},{"name":"邮件(收)"},{"name":"会议"},{"name":"消息"},{"name":"电话"},{"name":"其他"}]},
  {"name":"互动摘要","type":"text"},
  {"name":"涉及主题","type":"text"},
  {"name":"邮件ID","type":"text"},
  {"name":"会议ID","type":"text"}
]'
# 注：关联联系人字段需单独用 +field-create 创建（type="lookup" 需要关联目标表 ID）

# 2. 记录 base_token 和各 table_id
```

> 首次使用时提示用户提供 Base URL 或创建新的多维表格。
>
> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 模式一：数据导入

### 从邮箱导入

```bash
# 搜索指定时间范围的邮件
lark-cli mail +messages --folder INBOX --start-date "<YYYY-MM-DD>" --end-date "<YYYY-MM-DD>"
```

**AI 处理逻辑：**
1. 从发件人和收件人中提取联系人信息（姓名、邮箱、公司签名）
2. 分析邮件主题和正文，生成互动摘要（1-2句话）
3. 识别互动主题（项目名、话题关键词）
4. 对每个新联系人：创建联系人记录
5. 对每个互动：创建互动记录并关联联系人

**过滤规则：**
- 自动过滤广告邮件、系统通知、noreply 地址
- 过滤内部的自动通知（CI/CD、日历邀请等）
- 保留有实质内容的人际沟通

### 从日历导入

```bash
# 获取指定时间范围的日程
lark-cli calendar +agenda --start "<ISO8601>" --end "<ISO8601>"
```

**AI 处理逻辑：**
1. 从日程的参与者列表中提取联系人
2. 使用 `contact +search-user` 补充飞书通讯录信息（职位、部门等）
3. 生成互动摘要（会议主题）
4. 创建/更新联系人记录和互动记录

### 从飞书通讯录补充

```bash
lark-cli contact +search-user --query "<联系人姓名>"
```

当联系人信息不完整时，通过飞书通讯录补充公司、职位等信息。

---

## 模式二：热度分计算

热度分计算公式（0-100）：

```
热度分 = 频率分(0-40) + 近期分(0-40) + 重要度分(0-20)

频率分（过去 90 天互动次数）：
  0次 → 0分
  1-3次 → 10分
  4-10次 → 20分
  11-20次 → 30分
  20次以上 → 40分

近期分（最后一次互动距今天数）：
  7天内 → 40分
  14天内 → 32分
  30天内 → 24分
  60天内 → 16分
  90天内 → 8分
  90天以上 → 0分

重要度分：
  高 → 20分
  中 → 12分
  低 → 4分
```

> 热度分在每次导入新互动后自动重新计算。

---

## 模式三：自然语言查询

支持的自然语言查询类型：

| 查询类型 | 示例 | 查询方式 |
|----------|------|---------|
| 按时间 | "上个月谁联系过我？" | `base +record-list` 互动记录表 + 时间过滤 |
| 按公司 | "XX 公司的人我聊过什么？" | `base +record-list` 联系人表 + 公司过滤 |
| 按人名 | "张三最近怎么样？" | `base +record-list` 联系人表 + 姓名过滤 |
| 按热度 | "该跟进谁了？" | `base +data-query` 热度分 < 30 的联系人 |
| 按主题 | "关于 AI 项目的讨论有哪些？" | `base +record-list` 互动记录表 + 主题关键词 |
| 综合查询 | "有没有超过3个月没联系的重要关系？" | `base +data-query` 最后联系 > 90天 AND 重要程度 = "高" |

### 查询执行示例

```bash
# 查询冷门联系人（聚合查询）
lark-cli base +data-query --base-token "<base_token>" --dsl '{
  "datasource": {"type": "table", "table": {"tableId": "<contacts_table_id>"}},
  "dimensions": [{"field_name": "姓名", "alias": "name"}, {"field_name": "重要程度", "alias": "importance"}],
  "measures": [{"field_name": "热度分", "aggregation": "avg", "alias": "avg_score"}],
  "filters": {"type": 1, "conjunction": "and", "conditions": [{"field_name": "重要程度", "operator": "is", "value": ["高"]}, {"field_name": "热度分", "operator": "isLess", "value": ["30"]}]},
  "shaper": {"format": "flat"}
}'

# 查询特定公司的联系人
lark-cli base +record-list --table-id "<contacts_table_id>" --filter '...公司 = "XX公司"'

# 查询最近互动
lark-cli base +record-list --table-id "<interactions_table_id>" --sort '互动时间 DESC' --page-size 20
```

---

## 模式四：智能提醒

### 定时提醒

筛选需要跟进的联系人：

```bash
lark-cli base +data-query --base-token "<base_token>" --dsl '{
  "datasource": {"type": "table", "table": {"tableId": "<contacts_table_id>"}},
  "dimensions": [{"field_name": "姓名", "alias": "name"}],
  "measures": [{"field_name": "热度分", "aggregation": "avg", "alias": "avg_score"}],
  "filters": {"type": 1, "conjunction": "and", "conditions": [{"field_name": "重要程度", "operator": "isNot", "value": ["低"]}, {"field_name": "热度分", "operator": "isLess", "value": ["30"]}]},
  "shaper": {"format": "flat"}
}'
```

生成提醒消息并通过飞书发送：

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --markdown "..."
```

提醒消息格式：
```
关系维护提醒

以下联系人已超过 60 天未联系：
| 姓名 | 公司 | 最后联系 | 热度分 | 建议 |
|------|------|---------|--------|------|
| 张三 | XX科技 | 75天前 | 12 | 发个消息打个招呼 |
| 李四 | YY投资 | 90天前 | 4 | 约个会议聊聊近况 |
```

---

## 模式五：手动操作

### 添加/更新联系人

```bash
lark-cli base +record-upsert --table-id "<contacts_table_id>" --json '{"字段名":"值"}'
```

### 手动添加互动记录

```bash
lark-cli base +record-upsert --table-id "<interactions_table_id>" --json '{"字段名":"值"}'
```

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `mail +messages` | `mail:mail:read` |
| `calendar +agenda` | `calendar:calendar.event:read` |
| `contact +search-user` | `contact:employee.id:readonly` |
| `contact +get-user` | `contact:employee.id:readonly` |
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `base +data-query` | `bitable:app:read` |
| `im +messages-send` | `im:message:send_as_bot` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-mail`](../lark-mail/SKILL.md) — 邮件读取、安全规则
- [`lark-calendar`](../lark-calendar/SKILL.md) — `+agenda` 详细用法
- [`lark-contact`](../lark-contact/SKILL.md) — `+search-user`、`+get-user` 详细用法
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert`、`+data-query` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
