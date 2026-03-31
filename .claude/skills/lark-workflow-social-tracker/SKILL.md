---
name: lark-workflow-social-tracker
version: 1.0.0
description: "社交媒体数据追踪：维护社交媒体账号配置，通过AI获取公开数据生成每日快照，支持长期趋势分析。当用户需要'社交数据'、'平台数据'、'粉丝趋势'、'内容表现'、'数据追踪'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 社交媒体数据追踪工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我记录今天的社交数据" / "数据快照"
- "这周哪个平台增长最快？" / "粉丝趋势"
- "最近哪条内容表现最好？" / "内容表现分析"
- "YouTube 和 Instagram 的互动趋势有什么不同？" / "跨平台对比"
- "添加/删除社交账号" / "账号管理"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,sheets,im
```

> **数据来源说明：** 框架阶段，社交媒体数据通过以下方式获取：
> 1. **AI WebSearch**：搜索公开可用的社交媒体数据（如 Social Blade）
> 2. **用户手动录入**：用户直接提供数据，AI 记录到多维表格
> 3. **API 接入（未来）**：预留外部 API 接入点，后续可接入各平台官方 API

## 工作流总览

```
账号管理：
  base +record-upsert ──► 维护账号配置

数据采集：
  ├── WebSearch ──────────► 获取公开数据（框架阶段）
  ├── 用户手动输入 ────────► 直接录入数据
  └── [未来] API 调用 ────► 官方 API

数据存储：
  base +record-upsert ──► 每日快照 + 内容表现

趋势分析：
  base +record-list ──► 历史数据 ──► AI 分析趋势
  sheets +write ──────► 生成分析报表

通知：
  im +messages-send ──► 里程碑/异常通知
```

---

## 数据初始化

### Base 表结构

**表 1：账号配置 (social_accounts)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 平台 | 单选 | YouTube/Instagram/Twitter(X)/TikTok/微信公众号/B站/其他 |
| 账号名 | 文本 | 账号显示名 |
| 账号ID | 文本 | 平台唯一标识（@handle 或数字ID） |
| 主页链接 | URL | 账号主页 URL |
| 类别 | 单选 | 个人/品牌/公司/项目 |
| 状态 | 单选 | 活跃/暂停/已关闭 |
| 备注 | 文本 | 自由备注 |
| 追踪开始日期 | 日期 | 开始追踪的日期 |

**表 2：每日快照 (daily_snapshots)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 平台 | 文本 | 平台名称 |
| 账号ID | 文本 | 关联账号 |
| 快照日期 | 日期 | 数据日期 |
| 粉丝数 | 数字 | 总粉丝/关注者数 |
| 关注数 | 数字 | 关注了多少账号 |
| 内容总数 | 数字 | 发布的内容总数 |
| 新增粉丝 | 数字 | 当日新增粉丝数（与前一天对比） |
| 新增内容 | 数字 | 当日发布的内容数 |
| 互动率 | 数字 | 当日互动率（%），如可获取 |
| 总点赞 | 数字 | 所有内容总点赞数 |
| 总评论 | 数字 | 所有内容总评论数 |
| 备注 | 文本 | 当日特别事件（如上了热门） |
| 数据来源 | 单选 | WebSearch/手动录入/API |

**表 3：内容表现 (content_performance)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 平台 | 文本 | 平台名称 |
| 账号ID | 文本 | 关联账号 |
| 内容ID | 文本 | 内容的唯一标识 |
| 内容标题 | 文本 | 标题或描述 |
| 内容类型 | 单选 | 视频/图文/直播/短视频/推文/文章 |
| 发布日期 | 日期 | 发布时间 |
| 点赞数 | 数字 | 当日点赞 |
| 评论数 | 数字 | 当日评论 |
| 分享数 | 数字 | 当日分享/转发 |
| 播放数 | 数字 | 播放/浏览次数 |
| 互动率 | 数字 | 互动率（%） |
| 备注 | 文本 | 如上了热门/被推荐等 |
| 数据来源 | 单选 | WebSearch/手动录入/API |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "社交账号" --fields '[
  {"name":"平台","type":"select","multiple":false,"options":[{"name":"YouTube"},{"name":"Instagram"},{"name":"Twitter(X)"},{"name":"TikTok"},{"name":"微信公众号"},{"name":"B站"},{"name":"其他"}]},
  {"name":"账号名","type":"text"},
  {"name":"账号ID","type":"text"},
  {"name":"主页链接","type":"link"},
  {"name":"类别","type":"select","multiple":false,"options":[{"name":"个人"},{"name":"品牌"},{"name":"公司"},{"name":"项目"}]},
  {"name":"状态","type":"select","multiple":false,"options":[{"name":"活跃"},{"name":"暂停"},{"name":"已关闭"}]},
  {"name":"备注","type":"text"},
  {"name":"追踪开始日期","type":"datetime"}
]'
lark-cli base +table-create --name "每日快照" --fields '[
  {"name":"平台","type":"text"},
  {"name":"账号ID","type":"text"},
  {"name":"快照日期","type":"datetime"},
  {"name":"粉丝数","type":"number"},
  {"name":"关注数","type":"number"},
  {"name":"内容总数","type":"number"},
  {"name":"新增粉丝","type":"number"},
  {"name":"新增内容","type":"number"},
  {"name":"互动率","type":"number"},
  {"name":"总点赞","type":"number"},
  {"name":"总评论","type":"number"},
  {"name":"备注","type":"text"},
  {"name":"数据来源","type":"select","multiple":false,"options":[{"name":"WebSearch"},{"name":"手动录入"},{"name":"API"}]}
]'
lark-cli base +table-create --name "内容表现" --fields '[
  {"name":"平台","type":"text"},
  {"name":"账号ID","type":"text"},
  {"name":"内容ID","type":"text"},
  {"name":"内容标题","type":"text"},
  {"name":"内容类型","type":"select","multiple":false,"options":[{"name":"视频"},{"name":"图文"},{"name":"直播"},{"name":"短视频"},{"name":"推文"},{"name":"文章"}]},
  {"name":"发布日期","type":"datetime"},
  {"name":"点赞数","type":"number"},
  {"name":"评论数","type":"number"},
  {"name":"分享数","type":"number"},
  {"name":"播放数","type":"number"},
  {"name":"互动率","type":"number"},
  {"name":"备注","type":"text"},
  {"name":"数据来源","type":"select","multiple":false,"options":[{"name":"WebSearch"},{"name":"手动录入"},{"name":"API"}]}
]'
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 模式一：账号管理

### 添加账号

```bash
lark-cli base +record-upsert --table-id "<accounts_table_id>" --json '{
  "平台": "YouTube",
  "账号名": "MyChannel",
  "账号ID": "@mychannel",
  "主页链接": "https://youtube.com/@mychannel",
  "类别": "个人",
  "状态": "活跃",
  "追踪开始日期": "2026-03-31"
}'
```

### 查看账号列表

```bash
lark-cli base +record-list --table-id "<accounts_table_id>" --filter '...状态 = "活跃"'
```

---

## 模式二：每日快照

### 方式 A：WebSearch 获取（框架阶段）

**AI 处理：**
1. 获取活跃账号列表
2. 通过 WebSearch 搜索各账号的公开数据
3. 搜索关键词如："@handle Social Blade"、"@handle subscribers count"
4. 提取粉丝数、内容数等数据
5. 与前一天快照对比，计算增量

### 方式 B：手动录入

用户直接提供数据：
- "YouTube 今天 1000 粉丝了"
- "Instagram 昨天涨了 50 个粉丝"

**AI 处理：**
1. 解析用户提供的数字
2. 查找前一天的快照
3. 计算增量
4. 记录到快照表

### 记录快照

```bash
lark-cli base +record-upsert --table-id "<snapshots_table_id>" --json '{
  "平台": "YouTube",
  "账号ID": "@mychannel",
  "快照日期": "2026-03-31",
  "粉丝数": 1000,
  "关注数": 50,
  "内容总数": 45,
  "新增粉丝": 12,
  "新增内容": 1,
  "数据来源": "WebSearch"
}'
```

---

## 模式三：内容表现记录

### 记录内容数据

```bash
lark-cli base +record-upsert --table-id "<content_table_id>" --json '{
  "平台": "YouTube",
  "账号ID": "@mychannel",
  "内容ID": "abc123",
  "内容标题": "如何使用飞书CLI",
  "内容类型": "视频",
  "发布日期": "2026-03-30",
  "点赞数": 150,
  "评论数": 30,
  "分享数": 10,
  "播放数": 5000
}'
```

---

## 模式四：趋势分析

### 跨平台对比

```bash
# 获取指定时间范围的快照
lark-cli base +record-list --table-id "<snapshots_table_id>" --filter '...快照日期 >= "30天前"'
```

**AI 分析：**
1. 计算各平台的增长率和增长趋势
2. 对比不同平台的互动率
3. 识别增长最快的平台和最慢的平台
4. 分析发布频率与增长的关系

### 输出格式

```
## 社交媒体趋势报告（{日期范围}）

### 增长概览
| 平台 | 起始粉丝 | 结束粉丝 | 增长率 | 新增内容 | 互动率 |
|------|---------|---------|--------|---------|--------|
| YouTube | 800 | 1000 | +25% | 4 | 3.2% |
| Instagram | 500 | 550 | +10% | 10 | 5.1% |
| Twitter | 200 | 230 | +15% | 25 | 1.8% |

### 关键发现
- 增长最快：YouTube（+25%）
- 互动率最高：Instagram（5.1%）
- 发布最活跃：Twitter（25条内容）
- 建议：YouTube虽然增长快，但内容产出较少，考虑增加发布频率

### 内容表现 TOP 5
| 平台 | 内容标题 | 点赞 | 评论 | 互动率 |
|------|---------|------|------|--------|
| YT | 如何使用飞书CLI | 150 | 30 | 3.6% |
...
```

### 生成报表（可选）

将趋势分析保存为飞书电子表格：

```bash
lark-cli sheets +create --title "社交媒体趋势 ({日期范围})" --data "<表格数据>"
```

---

## 模式五：里程碑通知

当检测到重要变化时发送飞书消息通知：

- 粉丝数达到整数里程碑（1000、5000、10000...）
- 单日新增粉丝创历史新高
- 某内容表现异常好（互动率超过平均值 3 倍）
- 出现负增长（掉粉）

```bash
lark-cli im +messages-send --chat-id "<chat_id>" --markdown "<通知内容>"
```

---

## 模式六：定时快照

使用 CronCreate 设置每日自动采集：

```
# 每天晚上 22:30 自动采集社交媒体数据
CronCreate: cron="30 22 * * *", prompt="采集今天的社交媒体数据快照，记录到多维表格", recurring=true
```

---

## 未来扩展：API 接入

以下为预留的外部 API 接入点，后续可逐步实现：

| 平台 | API | 数据类型 |
|------|-----|---------|
| YouTube | YouTube Data API v3 | 粉丝数、播放量、互动 |
| Instagram | Instagram Graph API | 粉丝数、互动、Stories 数据 |
| Twitter/X | Twitter API v2 | 粉丝数、推文互动 |
| TikTok | TikTok Content Posting API | 粉丝数、视频数据 |
| B站 | B站 API | 粉丝数、播放量、互动 |
| 微信公众号 | 微信公众平台 API | 粉丝数、阅读量 |

> 当 API 接入后，将 `数据来源` 字段从 "WebSearch" 更新为 "API"，快照采集将变为自动化。

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `sheets +create` | `sheets:spreadsheet:create` |
| `sheets +write` | `sheets:spreadsheet:write` |
| `im +messages-send` | `im:message:send_as_bot` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-sheets`](../lark-sheets/SKILL.md) — `+create`、`+write` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
- [`lark-workflow-business-advisor`](../lark-workflow-business-advisor/SKILL.md) — 商业顾问团（数据联动）
