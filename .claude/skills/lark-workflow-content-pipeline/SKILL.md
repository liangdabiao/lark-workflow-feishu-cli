---
name: lark-workflow-content-pipeline
version: 1.0.0
description: "内容创作管道：在飞书群聊中标记内容创意，自动研究话题、检查重复选题、评估价值、生成完整方案，创建飞书任务跟踪。当用户需要'内容创意'、'灵感记录'、'选题评估'、'内容方案'、'创意管道'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 内容创作管道工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "这是一个内容创意" / "标记为内容创意"
- "帮我评估这个选题" / "选题评估"
- "研究一下这个话题" / "话题调研"
- "生成内容方案" / "写个大纲"
- "我的创意库有哪些？" / "创意列表"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain im,task,wiki,base,docs
```

## 工作流总览

```
创意触发：
  用户标记消息/直接描述 ──► AI 读取上下文

话题研究：
  ├── WebSearch ──────────► 搜索最新动态
  ├── doc +search ─────────► 检查知识库重复选题
  └── base +record-list ───► 检查创意库重复

评估与方案：
  AI 评估价值 ──► 生成方案（标题/大纲/开头）

任务创建：
  task +create ──► 创建飞书任务跟踪
  base +record-upsert ──► 存入创意库

通知：
  im +messages-send ──► 回复完成通知
```

---

## 数据初始化

### Base 表结构

**表 1：创意库 (idea_library)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 创意标题 | 文本 | 创意的简短标题 |
| 来源 | 单选 | 群聊/对话/手动/其他 |
| 来源详情 | 文本 | 来源群聊名/对话上下文 |
| 原始描述 | 文本 | 创意的原始描述 |
| 话题领域 | 单选 | 科技/商业/AI/产品/其他 |
| 评估分数 | 数字 | 1-10 分（AI评估+用户调整） |
| 状态 | 单选 | 灵感/研究中/方案/执行中/已完成/已放弃 |
| 重复检查 | 单选 | 通过/疑似重复/重复 |
| 重复来源 | 文本 | 如果重复，指向已有内容 |
| 关联任务ID | 文本 | 飞书任务 task_id |
| 方案文档链接 | URL | 生成的方案文档链接 |
| 创建时间 | 日期 | 入库时间 |
| 更新时间 | 日期 | 最后更新时间 |

**表 2：创意研究 (idea_research)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 关联创意 | 关联 | 关联到「创意库」 |
| 研究来源 | 文本 | 搜索来源 URL 或描述 |
| 研究类型 | 单选 | 最新动态/竞品分析/用户讨论/数据支持 |
| 摘要 | 文本 | 研究内容摘要 |
| 相关性 | 单选 | 高/中/低 |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "创意库" --fields '[
  {"name":"创意标题","type":"text"},
  {"name":"来源","type":"select","multiple":false,"options":[{"name":"群聊"},{"name":"对话"},{"name":"手动"},{"name":"其他"}]},
  {"name":"来源详情","type":"text"},
  {"name":"原始描述","type":"text"},
  {"name":"话题领域","type":"select","multiple":false,"options":[{"name":"科技"},{"name":"商业"},{"name":"AI"},{"name":"产品"},{"name":"其他"}]},
  {"name":"评估分数","type":"number"},
  {"name":"状态","type":"select","multiple":false,"options":[{"name":"灵感"},{"name":"研究中"},{"name":"方案"},{"name":"执行中"},{"name":"已完成"},{"name":"已放弃"}]},
  {"name":"重复检查","type":"select","multiple":false,"options":[{"name":"通过"},{"name":"疑似重复"},{"name":"重复"}]},
  {"name":"重复来源","type":"text"},
  {"name":"关联任务ID","type":"text"},
  {"name":"方案文档链接","type":"link"},
  {"name":"创建时间","type":"datetime"},
  {"name":"更新时间","type":"datetime"}
]'
lark-cli base +table-create --name "创意研究" --fields '[
  {"name":"研究来源","type":"text"},
  {"name":"研究类型","type":"select","multiple":false,"options":[{"name":"最新动态"},{"name":"竞品分析"},{"name":"用户讨论"},{"name":"数据支持"}]},
  {"name":"摘要","type":"text"},
  {"name":"相关性","type":"select","multiple":false,"options":[{"name":"高"},{"name":"中"},{"name":"低"}]}
]'
# 注：关联创意字段需单独用 +field-create 创建（type="lookup" 需要关联目标表 ID）
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 模式一：创意触发与处理

### 方式 A：从飞书群聊标记

用户在群聊中标记某条消息为"内容创意"：

```bash
# 获取群聊消息及上下文
lark-cli im +chat-messages-list --chat-id "<chat_id>" --count 20
```

AI 读取消息上下文，理解创意的完整背景。

### 方式 B：直接描述

用户在对话中直接描述创意：
- "我觉得可以写一篇关于 AI Agent 在企业落地的文章"
- "这个话题不错，帮我做成选题"

### 处理流程

```
创意触发 ──► AI 理解背景 ──► 提炼核心创意
                                  │
                                  ▼
                          进入研究流程
```

---

## 模式二：话题研究

### Step 1: 搜索最新动态

使用 AI 能力搜索相关话题的最新动态：

- 搜索引擎搜索：最新文章、讨论、趋势
- 技术社区：相关技术进展

### Step 2: 检查重复选题

**检查创意库：**
```bash
lark-cli base +record-list --table-id "<idea_table_id>" --filter '...话题领域 = "<领域>"'
```

**检查知识库（联动知识库 Skill）：**
```bash
lark-cli docs +search --query "<话题关键词>"
```

### Step 3: 评估话题热度

AI 基于搜索结果评估：
- 该话题最近的讨论热度
- 是否有时效性（热点事件）
- 竞争对手是否已覆盖类似话题
- 目标受众的兴趣度

### Step 4: 记录研究结果

```bash
lark-cli base +record-upsert --table-id "<research_table_id>" --json '{"字段名":"值"}'
```

---

## 模式三：方案生成

### AI 评估创意价值

评估维度（每项 1-10 分）：

| 维度 | 权重 | 说明 |
|------|------|------|
| 时效性 | 25% | 是否是当下热点 |
| 差异化 | 25% | 与已有内容的差异程度 |
| 受众价值 | 20% | 对目标受众的价值 |
| 可执行性 | 15% | 创意的可执行程度 |
| 延展性 | 15% | 是否可以做成系列 |

### 生成内容方案

```
## 内容方案

### 基本信息
- 标题建议：{3个备选标题}
- 内容类型：{文章/视频/播客/系列}
- 目标字数：{建议字数}
- 预计耗时：{预估}

### 开头建议（Hook）
> {一个能抓住读者注意力的开头段落}

### 详细大纲
1. {第一章标题}
   - {要点1}
   - {要点2}
2. {第二章标题}
   - {要点1}
   - {要点2}
...

### 关键数据/引用
- {支撑论点的数据和引用}

### 发布建议
- 推荐平台：{平台列表}
- 最佳发布时间：{时间建议}
- 标签建议：{标签列表}

### 评估分数：{总分}/10
- 时效性：{分} | 差异化：{分} | 受众价值：{分}
- 可执行性：{分} | 延展性：{分}
```

### 保存方案为飞书文档

```bash
lark-cli docs +create --title "内容方案：{创意标题}" --markdown "<方案内容>"
```

---

## 模式四：任务创建

用户确认方案后，自动创建飞书任务跟踪：

```bash
lark-cli task +create --summary "创作：{创意标题}" --description "内容方案已生成，详见文档链接" --due "<建议完成日>"
```

更新创意库状态：

```bash
lark-cli base +record-upsert --table-id "<idea_table_id>" --json '{
  "创意标题": "...",
  "状态": "执行中",
  "关联任务ID": "...",
  "方案文档链接": "..."
}'
```

---

## 模式五：创意库管理

### 查看创意列表

```bash
lark-cli base +record-list --table-id "<idea_table_id>" --filter '...状态 != "已完成" AND 状态 != "已放弃"' --sort '评估分数 DESC'
```

### 更新创意状态

- "把这个创意标记为已完成" → 更新状态为"已完成"
- "放弃这个创意" → 更新状态为"已放弃"
- "调整评估分数" → 更新评估分数

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `im +chat-messages-list` | `im:message:readonly` |
| `im +messages-send` | `im:message:send_as_bot` |
| `task +create` | `task:task:write` |
| `docs +create` | `docx:document:create` |
| `docs +search` | `drive:drive:read` |
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-im`](../lark-im/SKILL.md) — `+chat-messages-list`、`+messages-send` 详细用法
- [`lark-task`](../lark-task/SKILL.md) — `+create` 详细用法
- [`lark-doc`](../lark-doc/SKILL.md) — `+create`、`+search` 详细用法
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-workflow-knowledge-base`](../lark-workflow-knowledge-base/SKILL.md) — 知识库联动（重复检查）
