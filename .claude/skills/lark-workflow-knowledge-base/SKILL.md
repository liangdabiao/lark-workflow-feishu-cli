---
name: lark-workflow-knowledge-base
version: 1.0.0
description: "个人知识库：通过飞书消息保存内容（文章/视频/推文/PDF），AI自动抓取生成摘要标签，存入飞书知识库，支持自然语言搜索和内容关联分析。当用户需要'知识库'、'保存文章'、'搜索知识库'、'之前保存过'、'知识管理'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 个人知识库工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我保存这篇文章" / "把这个链接存到知识库"
- "关于 AI 芯片的文章有哪些？" / "搜索知识库"
- "我之前保存过一个关于 Nvidia 的视频，帮我找出来"
- "最近保存了什么内容？" / "知识库概览"
- "把新内容和之前保存的关联起来"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain wiki,docs,im,base,drive
```

## 工作流总览

```
保存内容：
  用户发送链接/内容 ──► AI 抓取内容 ──► 生成摘要+标签
                                      │
                                      ▼
                              doc +create (知识库) ──► base +record-upsert (索引)

搜索内容：
  自然语言查询 ──► base +record-list ──► 匹配结果
                                       │
                                       ▼
                               doc +fetch (详细内容)

关联分析：
  新保存内容 ──► base +record-list (已有标签/关键词) ──► 发现关联
                                                        │
                                                        ▼
                                               im +messages-send (通知关联)
```

---

## 数据初始化

### 首次设置

1. **创建/指定知识库空间**

```bash
# 如果用户已有知识库空间，记录 space_id
# 否则在飞书中创建一个个人知识库空间
```

2. **创建索引多维表格**

**表名：知识库索引 (kb_index)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 标题 | 文本 | 内容标题 |
| URL | URL | 原始链接 |
| 内容类型 | 单选 | 文章/视频/推文/PDF/播客/其他 |
| 来源平台 | 单选 | 网站/YouTube/Twitter/X/公众号/其他 |
| 标签 | 多选 | AI 自动生成的标签 |
| 摘要 | 文本 | AI 生成的内容摘要（200字以内） |
| 关键观点 | 文本 | 核心观点或结论 |
| 文档链接 | URL | 飞书文档链接 |
| 文档Token | 文本 | 飞书文档 doc_token |
| 保存时间 | 日期 | 入库时间 |
| 关联条目 | 关联 | 关联到其他知识库条目 |
| 评分 | 数字 | 用户评分（1-5） |
| 备注 | 文本 | 用户备注 |

### 初始化命令

```bash
# 创建索引表
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "知识库索引" --fields '[
  {"name":"标题","type":"text"},
  {"name":"URL","type":"link"},
  {"name":"内容类型","type":"select","multiple":false,"options":[{"name":"文章"},{"name":"视频"},{"name":"推文"},{"name":"PDF"},{"name":"播客"},{"name":"其他"}]},
  {"name":"来源平台","type":"select","multiple":false,"options":[{"name":"网站"},{"name":"YouTube"},{"name":"Twitter/X"},{"name":"公众号"},{"name":"其他"}]},
  {"name":"标签","type":"text"},
  {"name":"摘要","type":"text"},
  {"name":"关键观点","type":"text"},
  {"name":"文档链接","type":"link"},
  {"name":"文档Token","type":"text"},
  {"name":"保存时间","type":"datetime"},
  {"name":"评分","type":"number"},
  {"name":"备注","type":"text"}
]'
# 注：关联条目字段需单独用 +field-create 创建（type="lookup" 需要关联目标表 ID）
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 模式一：保存内容

### Step 1: 获取内容

**从飞书消息接收：**
```bash
# 查看用户发送的消息
lark-cli im +chat-messages-list --chat-id "<chat_id>"
```

**直接从对话中获取 URL：**
用户在对话中发送链接，AI 直接提取 URL。

### Step 2: 抓取内容

使用 AI 能力抓取链接内容：

- **文章 URL** → 抓取全文内容
- **YouTube 视频** → 抓取标题、描述、字幕/转录
- **Twitter/X 推文** → 抓取完整帖子串（不只单条）
- **PDF URL** → 下载并提取文本
- **播客** → 抓取标题、描述、时间戳摘要

### Step 3: AI 生成摘要和标签

对抓取到的内容，AI 自动处理：

1. **生成摘要**：200字以内，概括核心内容
2. **提取关键观点**：3-5个要点
3. **生成标签**：3-8个标签，基于内容和主题
4. **判断内容类型**：文章/视频/推文/PDF/其他

### Step 4: 检查重复

```bash
lark-cli base +record-list --table-id "<table_id>" --filter '...URL = "<目标URL>"'
```

如果 URL 已存在，提示用户"该内容已在知识库中"，并展示已有记录。

### Step 5: 发现关联内容

基于标签和关键词，搜索已有知识库中的相关内容：

```bash
lark-cli base +record-list --table-id "<table_id>" --filter '...标签 CONTAINS_ANY [标签1, 标签2]'
```

### Step 6: 保存到飞书

**创建飞书文档（知识库中）：**
```bash
lark-cli docs +create --title "<内容标题>" --markdown "<格式化后的全文内容>"
```

**更新索引表：**
```bash
lark-cli base +record-upsert --table-id "<table_id>" --json '{
  "标题": "...",
  "URL": "...",
  "内容类型": "文章",
  "标签": ["标签1", "标签2"],
  "摘要": "...",
  "关键观点": "...",
  "文档链接": "...",
  "保存时间": "..."
}'
```

### Step 7: 通知用户

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --markdown "已保存到知识库！"
```

通知内容包括：
- 标题和摘要
- 自动生成的标签
- 发现的关联内容（如有）
- 飞书文档链接

---

## 模式二：搜索内容

### 自然语言搜索

将用户的自然语言查询转换为标签/关键词搜索：

```bash
# 基于关键词搜索
lark-cli base +record-list --table-id "<table_id>" --filter '...摘要 CONTAINS "AI芯片"'

# 基于标签搜索
lark-cli base +record-list --table-id "<table_id>" --filter '...标签 CONTAINS_ANY ["AI", "芯片", "Nvidia"]'

# 基于时间搜索
lark-cli base +record-list --table-id "<table_id>" --filter '...保存时间 >= "2026-03-01"'
```

### 搜索结果展示

```
找到 N 条相关内容：

### 1. {标题}
- 类型：文章 | 平台：36氪
- 标签：AI、芯片、GPU
- 摘要：{摘要内容}
- 保存时间：2026-03-15
- [查看详情]({飞书文档链接})

### 2. {标题}
...
```

---

## 模式三：关联分析

当保存新内容时，自动与已有内容进行关联分析：

**AI 处理逻辑：**
1. 提取新内容的标签和关键词
2. 在索引表中搜索相同/相似标签的条目
3. 对内容摘要进行语义相似度比较
4. 生成关联说明："你三周前存过一篇类似的观点，当时是从另一个角度分析的"

**发现关联时：**
```bash
lark-cli base +record-upsert --table-id "<table_id>" --json '{
  "关联条目": "已有关联条目ID"
}'
```

---

## 模式四：批量导入

支持一次性导入多条内容：

1. 用户发送多个链接（一行一个）
2. AI 逐一抓取和处理
3. 批量创建飞书文档
4. 批量更新索引表
5. 生成导入报告

---

## 模式五：定期整理

用户可要求定期整理知识库：

- 合并重复内容
- 更新过时的标签
- 生成知识库统计报告（总条目数、按类型分布、按标签分布、最近保存趋势）

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `docs +create` | `docx:document:create` |
| `docs +fetch` | `docx:document:read` |
| `wiki spaces get_node` | `wiki:node:read` |
| `im +messages-send` | `im:message:send_as_bot` |
| `im +chat-messages-list` | `im:message:readonly` |
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `drive metas batch_query` | `drive:drive:read` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-doc`](../lark-doc/SKILL.md) — `+create`、`+fetch`、`+search` 详细用法
- [`lark-wiki`](../lark-wiki/SKILL.md) — 知识库节点管理
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send`、`+chat-messages-list` 详细用法
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-drive`](../lark-drive/SKILL.md) — 文件元数据查询
