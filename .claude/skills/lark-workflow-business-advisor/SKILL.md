---
name: lark-workflow-business-advisor
version: 1.0.0
description: "AI商业顾问团：8个AI专家角色每晚分析业务数据，汇总去重后发送编号清单到飞书，支持回复编号展开详细分析。当用户需要'商业分析'、'业务报告'、'顾问分析'、'夜间分析'、'业务建议'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# AI 商业顾问团工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我分析一下最近的业务状况" / "商业分析"
- "生成今日业务报告" / "顾问分析"
- "第 3 条详细说说" / "展开第 N 条建议"
- "昨晚的分析报告" / "查看顾问建议"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,docs,im,task,calendar
```

> 本 Skill 依赖其他工作流 Skill 的数据。需要以下 Skill 至少部分初始化：
> - `lark-workflow-personal-crm`（CRM 联系人数据）
> - `lark-workflow-social-tracker`（社交媒体数据）
> - `lark-workflow-content-pipeline`（创意/内容数据）
> - `lark-workflow-stock-analyzer`（投资/财务数据）
>
> 未初始化的 Skill 数据将被跳过，不影响其他分析。

## 工作流总览

```
数据采集（每晚自动）：
  ├── base +record-list (CRM) ──────► 联系人动态
  ├── base +record-list (社交追踪) ─► 数据表现
  ├── base +record-list (创意库) ───► 内容进度
  ├── base +record-list (分析记录) ─► 投资动态
  ├── task +get-my-tasks ───────────► 任务完成情况
  └── calendar +agenda ─────────────► 近期日程密度

AI 多角色分析：
  ├── 财务顾问 ──► 分析
  ├── 营销顾问 ──► 分析
  ├── 增长顾问 ──► 分析
  ├── 运营顾问 ──► 分析
  ├── 内容策略顾问 ──► 分析
  ├── 关系/BD顾问 ──► 分析
  ├── 竞争/行业顾问 ──► 分析
  └── 综合风险顾问 ──► 分析

汇总：
  AI 汇总去重排序 ──► 编号清单

输出：
  ├── doc +create ──► 存档报告
  ├── im +messages-send ──► 发送清单
  └── base +record-upsert ──► 保存分析记录
```

---

## 数据初始化

### Base 表结构

**表 1：顾问分析记录 (advisor_analysis)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 分析日期 | 日期 | 分析执行日期 |
| 顾问角色 | 单选 | 财务/营销/增长/运营/内容策略/关系BD/竞争行业/综合风险 |
| 分析摘要 | 文本 | 该顾问的核心发现（2-3句话） |
| 建议列表 | 文本 | 编号建议列表 |
| 数据来源 | 文本 | 使用了哪些数据源 |
| 优先级 | 数字 | 建议的重要性评分（1-10） |

**表 2：汇总建议 (advisor_summary)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 建议编号 | 文本 | 当日编号（如 "1"、"2"） |
| 建议内容 | 文本 | 去重后的建议描述 |
| 来源角色 | 文本 | 提出该建议的顾问角色 |
| 优先级 | 数字 | 重要性评分 |
| 用户反馈 | 单选 | 未读/已读/已展开/已执行/忽略 |
| 展开详情 | 文本 | 用户要求展开时的详细分析 |
| 分析日期 | 日期 | 所属分析日期 |
| 报告链接 | URL | 当日完整报告的文档链接 |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "顾问分析记录" --fields '[
  {"name":"分析日期","type":"datetime"},
  {"name":"顾问角色","type":"select","multiple":false,"options":[{"name":"财务"},{"name":"营销"},{"name":"增长"},{"name":"运营"},{"name":"内容策略"},{"name":"关系BD"},{"name":"竞争行业"},{"name":"综合风险"}]},
  {"name":"分析摘要","type":"text"},
  {"name":"建议列表","type":"text"},
  {"name":"数据来源","type":"text"},
  {"name":"优先级","type":"number"}
]'
lark-cli base +table-create --name "汇总建议" --fields '[
  {"name":"建议编号","type":"text"},
  {"name":"建议内容","type":"text"},
  {"name":"来源角色","type":"text"},
  {"name":"优先级","type":"number"},
  {"name":"用户反馈","type":"select","multiple":false,"options":[{"name":"未读"},{"name":"已读"},{"name":"已展开"},{"name":"已执行"},{"name":"忽略"}]},
  {"name":"展开详情","type":"text"},
  {"name":"分析日期","type":"datetime"},
  {"name":"报告链接","type":"link"}
]'
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## Step 1: 数据采集

根据已初始化的 Skill，拉取对应数据：

```bash
# CRM 数据（如果 personal-crm 已初始化）
lark-cli base +record-list --table-id "<crm_contacts_table_id>" --filter '...热度分 < 30'
lark-cli base +record-list --table-id "<crm_interactions_table_id>" --filter '...互动时间 >= "7天前"'

# 社交追踪数据（如果 social-tracker 已初始化）
lark-cli base +record-list --table-id "<social_snapshot_table_id>" --filter '...日期 >= "7天前"'

# 创意/内容数据（如果 content-pipeline 已初始化）
lark-cli base +record-list --table-id "<idea_table_id>" --filter '...状态 != "已完成"'

# 投资数据（如果 stock-analyzer 已初始化）
lark-cli base +record-list --table-id "<portfolio_table_id>"
lark-cli base +record-list --table-id "<analysis_table_id>" --filter '...分析日期 >= "7天前"'

# 任务和日程（始终可用）
lark-cli task +get-my-tasks --page-all
lark-cli calendar +agenda --start "7天前" --end "今天"
```

> **注意**：未初始化的 Skill 数据表不存在时，跳过该数据源即可，不影响其他分析。

---

## Step 2: 8 角色独立分析

### 1. 财务顾问

**关注数据：** 投资组合（stock-analyzer）、收入/支出趋势
**分析内容：**
- 投资组合表现（盈亏、波动）
- 收入趋势变化
- 支出是否合理
- 现金流健康度

### 2. 营销顾问

**关注数据：** 社交媒体数据（social-tracker）、内容表现
**分析内容：**
- 各平台互动趋势
- 哪些内容类型表现最好
- 内容发布节奏是否合理
- 受众增长/流失趋势

### 3. 增长顾问

**关注数据：** 粉丝增长、渠道获客
**分析内容：**
- 总体增长趋势
- 增长最快/最慢的渠道
- 增长率变化
- 与同行的对比

### 4. 运营顾问

**关注数据：** 任务完成率、日程密度
**分析内容：**
- 任务完成率（已完成/总任务）
- 日程安排是否合理（过度/空闲）
- 系统使用效率
- 流程瓶颈

### 5. 内容策略顾问

**关注数据：** 创意库（content-pipeline）、内容表现
**分析内容：**
- 选题质量分布
- 创意执行进度
- 跨平台内容效果对比
- 内容缺口（哪些领域尚未覆盖）

### 6. 关系/BD 顾问

**关注数据：** CRM 联系人（personal-crm）、会议记录
**分析内容：**
- 冷门联系人（需要跟进）
- 新增联系人趋势
- 关系深度分布
- 合作机会识别

### 7. 竞争/行业顾问

**关注数据：** 知识库（knowledge-base）、行业动态
**分析内容：**
- 行业趋势变化
- 竞争对手动态
- 新技术/新模式
- 需要关注的新领域

### 8. 综合风险顾问

**关注数据：** 全部数据
**分析内容：**
- 各维度的潜在风险
- 薄弱环节识别
- 需要优先处理的问题
- 预警信号

---

## Step 3: 汇总去重排序

AI "总秘书"角色汇总所有顾问的建议：

1. **去重**：合并不同顾问提出的相同或高度相似的建议
2. **优先级排序**：综合重要性和紧急程度
3. **编号**：为每条建议分配编号
4. **分类标注**：标注建议所属的领域

输出格式：

```
## 今日商业顾问团报告（{日期}）

### 总览
今日共收到 {N} 条独立建议，来自 {M} 位顾问。
数据来源：CRM({有/无})、社交追踪({有/无})、创意库({有/无})、投资({有/无})

### 建议清单

1. 【财务】你的投资组合本周下跌 5%，主要受科技股拖累。建议检查 AAPL 的持仓比例是否过高。
2. 【关系】张三（XX科技）已 45 天未联系，上次讨论了合作意向，建议主动跟进。
3. 【内容】本周内容发布频率下降 40%，建议安排时间完成创意库中的待执行选题。
4. 【运营】你有 7 个任务已过期未完成，其中 3 个超过 14 天，建议集中清理。
5. 【风险】竞品 YY 产品本周发布新功能，与你的产品路线图有重叠，建议关注。
...
```

---

## Step 4: 发送和存档

### 发送到飞书

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --markdown "<清单内容>"
```

### 存档报告

```bash
lark-cli docs +create --title "商业顾问团日报 ({日期})" --markdown "<完整报告>"
```

### 保存分析记录

```bash
lark-cli base +record-upsert --table-id "<analysis_table_id>" --json '{"字段名":"值"}'
lark-cli base +record-upsert --table-id "<summary_table_id>" --json '{"字段名":"值"}'
```

---

## 模式二：展开详细分析

用户回复编号（如"第 3 条详细说说"）时：

1. 查找该编号对应的建议和来源角色
2. 重新分析该角色的完整观点
3. 提供更详细的论据和数据支持
4. 给出具体的行动步骤

### 展开格式

```
## 第 {N} 条展开分析

### 原始建议
{原始建议内容}
### 来源顾问：{角色名}

### 详细分析
{展开的详细分析内容}

### 数据支撑
{具体数据和来源}

### 建议行动步骤
1. {步骤1}
2. {步骤2}
3. {步骤3}

### 预期效果
{如果执行该建议，预期会有什么改善}
```

---

## 模式三：定时执行

使用 CronCreate 设置每晚自动执行：

```
# 每晚 22:00 自动生成商业分析报告
CronCreate: cron="0 22 * * 1-5", prompt="运行商业顾问团分析，生成今日报告并发送到飞书", recurring=true
```

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `docs +create` | `docx:document:create` |
| `im +messages-send` | `im:message:send_as_bot` |
| `task +get-my-tasks` | `task:task:read` |
| `calendar +agenda` | `calendar:calendar.event:read` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-doc`](../lark-doc/SKILL.md) — `+create` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
- [`lark-task`](../lark-task/SKILL.md) — `+get-my-tasks` 详细用法
- [`lark-calendar`](../lark-calendar/SKILL.md) — `+agenda` 详细用法
- [`lark-workflow-personal-crm`](../lark-workflow-personal-crm/SKILL.md) — CRM 数据联动
- [`lark-workflow-social-tracker`](../lark-workflow-social-tracker/SKILL.md) — 社交数据联动
- [`lark-workflow-content-pipeline`](../lark-workflow-content-pipeline/SKILL.md) — 创意数据联动
- [`lark-workflow-stock-analyzer`](../lark-workflow-stock-analyzer/SKILL.md) — 投资数据联动
