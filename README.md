# 飞书 AI 效率系统 — 22 大工作流 Skill

> 基于 Claude Code Skill + 飞书 CLI (lark-cli) 构建的个人 AI 效率基础设施。
> 将 22 大实战用例完整迁移到飞书生态，用飞书各模块实现所有工作流功能。
> 特别感谢 linuxdo 的佬友支持：https://linux.do/t/topic/1868703

## 目录

- [项目简介](#项目简介)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [系统架构](#系统架构)
- [SaaS → 飞书映射](#saas--飞书映射)
- [Skill 详解](#skill-详解)
  - [用法 1: 个人 CRM](#1-个人-crm)
  - [用法 2: 会议待办追踪](#2-会议待办追踪)
  - [用法 3: 个人知识库](#3-个人知识库)
  - [用法 4: 社交媒体追踪](#4-社交媒体追踪)
  - [用法 5: AI 商业顾问团](#5-ai-商业顾问团)
  - [用法 6: 内容创作管道](#6-内容创作管道)
  - [用法 7: 每日晨间简报](#7-每日晨间简报)
  - [用法 8: 股票研究助手](#8-股票研究助手)
  - [用法 9: 多编程代理协同](#9-多编程代理协同)
  - [用法 10: 健康日记](#10-健康日记)
  - [用法 11: 任务状态播报](#11-任务状态播报)
  - [用法 12: 日程冲突检测](#12-日程冲突检测)
  - [用法 13: 群消息日报](#13-群消息日报)
  - [用法 14: 会议纪要分发](#14-会议纪要分发)
  - [用法 15: 新人入职向导](#15-新人入职向导)
  - [用法 16: 文档权限管家](#16-文档权限管家)
  - [用法 17: 智能表单收集](#17-智能表单收集)
  - [用法 18: 跨表数据同步](#18-跨表数据同步)
  - [用法 19: 群公告管理](#19-群公告管理)
  - [用法 20: 审批催办机器人](#20-审批催办机器人)
  - [用法 21: 进销存 ERP](#21-进销存-erp)
  - [用法 22: 团队 CRM](#22-团队-crm)
- [Skill 间数据联动关系](#skill-间数据联动关系)
- [定时自动化配置](#定时自动化配置)
- [权限清单](#权限清单)
- [常见问题](#常见问题)

---

## 项目简介

本项目的核心思想：**用飞书作为统一的数据底座和交互界面，用 Claude Code 作为智能调度中枢，把原本分散在 Gmail、Telegram、Todoist、Slack、Asana 等十几个 SaaS 工具中的个人效率工作流，全部收敛到飞书一个平台内运行。**

每个工作流被封装为一个 **Claude Code Skill**（SKILL.md），通过自然语言对话或指令触发，调用 `lark-cli` 操作飞书 API 完成数据读写、消息通知、文档生成等操作。

本项目包含：
- **22 个工作流 Skill**：面向实际办公场景的端到端自动化流程
- **17 个原子 Skill**：封装飞书各模块 API 的基础能力单元

注意： 如果需要抓取社交网络数据，那么我推荐下载如下 skills:
https://github.com/apify/agent-skills

https://github.com/liangdabiao/tikhub_api_skill

## 环境要求

| 组件 | 要求 |
|------|------|
| Claude Code | CLI 工具，用于执行 Agent 和 Skill |
| lark-cli | 飞书命令行工具，已全局安装 |
| 飞书应用 | App ID `cli_a94d22643c38dbcd`，需在开发者后台配置 |
| 用户授权 | 已通过 `lark-cli auth login` 完成登录授权 |
| Node.js | lark-cli 依赖 |

## 快速开始

### 1. 确认环境就绪

```bash
# 检查 lark-cli 版本
lark-cli --version

# 检查登录状态（一次授权所有模块）
lark-cli auth login --domain im,calendar,task,mail,base,docs,wiki,drive,vc,minutes,contact,sheets

# 检查 Skill 是否就位
ls .claude/skills/lark-workflow-*/
```

### 2. 选择一个 Skill 开始

每个 Skill 首次使用时会自动引导你完成数据初始化（创建多维表格等）。直接用自然语言触发即可：

```
# 想看今天的安排
你：今天有什么安排？

# 想记录饮食
你：午餐吃了红烧牛肉面和一杯冰美式

# 想保存一篇文章
你：帮我保存这篇文章到知识库 https://example.com/article

# 想催一下审批
你：帮我催一下那个报销审批

# 想查找空闲时间开会
你：帮我找一个明天下午张三和李四都空闲的时间
```

### 3. 按需初始化

首次使用某个 Skill 时，AI 会提示你创建所需的多维表格。你也可以主动初始化：

```
你：帮我初始化个人CRM系统
你：初始化健康日记
你：设置晨间简报
```

---

## 系统架构

```
┌─────────────────────────────────────────────────┐
│                  用户 (自然语言)                    │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│                  Claude Code                      │
│     (意图识别 → Skill 路由 → 任务编排 → 输出)     │
└──────────┬───────────────────────────┬───────────┘
           │                           │
     ┌─────┴─────┐              ┌─────┴──────┐
     │ 22个      │              │ 17个       │
     │ Workflow  │              │ 原子 Skills│
     │ Skills    │              │            │
     └─────┬─────┘              └─────┬──────┘
           │                         │
           ▼                         ▼
┌──────────────────────────────────────────────────┐
│                   飞书开放平台                       │
│  IM | 日历 | 任务 | 邮箱 | 文档 | 知识库 |          │
│  多维表格 | 电子表格 | 通讯录 | 妙记 | 云空间        │
└──────────────────────────────────────────────────┘
```

## SaaS → 飞书映射

| 原始 SaaS | 飞书替代 | 用法 |
|-----------|---------|------|
| Gmail | 飞书邮箱 | CRM、晨间简报 |
| Google Calendar | 飞书日历 | CRM、晨间简报、日程冲突检测 |
| Telegram | 飞书消息 (IM) | 所有 Skill 的通知通道 |
| Todoist / Asana | 飞书任务 | 会议待办、内容管道、任务播报 |
| Slack | 飞书消息 (群聊) | 内容管道、群消息日报 |
| Fathom | 飞书妙记 / 视频会议 | 会议待办追踪、会议纪要分发 |
| Notion / 各种知识库 | 飞书知识库 + 云文档 | 个人知识库 |
| Airtable / 各类表格 | 飞书多维表格 (Base) | 所有 Skill 的数据存储 |
| Google Forms / 金数据 | 飞书多维表格表单 | 智能表单收集 |
| 钉钉审批 / 泛微 | 飞书审批 | 审批催办机器人 |
| Yahoo Finance | AI WebSearch + 多维表格 | 股票研究助手 |
| 各平台 Social API | AI WebSearch + 多维表格 | 社交媒体追踪 |

---

## Skill 详解

---

### 1. 个人 CRM

> 自动扫描飞书邮箱和日历，提取联系人并维护关系热度分，支持自然语言查询和超期提醒。

**文件位置：** `.claude/skills/lark-workflow-personal-crm/SKILL.md`

**触发方式：** 对话中说以下任意关键词

```
"联系人管理"  "CRM"  "谁联系过我"  "多久没联系"  "联系人热度"  "关系管理"
```

**使用示例：**

```
你：帮我建立个人CRM系统
AI：首次使用需要创建数据表，现在开始初始化...
    [创建多维表格：联系人表 + 互动记录表]

你：从邮箱导入过去一个月的联系人
AI：正在扫描邮箱...发现 15 个新联系人，23 条互动记录。
    已自动过滤广告邮件和系统通知。

你：上个月谁联系过我？
AI：[查询互动记录表]
    上个月你与以下 8 人有互动：
    1. 张三（XX科技）— 邮件 3 次，最后 3/25
    2. 李四（YY投资）— 会议 1 次 + 邮件 1 次，最后 3/20
    ...

你：有没有超过3个月没联系的重要关系？
AI：[查询热度分 < 30 且重要程度 = 高]
    发现 3 个需要跟进的联系人：
    | 姓名 | 公司 | 最后联系 | 热度分 |
    |------|------|---------|--------|
    | 赵六 | ZZ集团 | 95天前 | 8  |
```

**热度分计算规则：**

| 维度 | 权重 | 计算方式 |
|------|------|---------|
| 频率分 | 40分 | 过去90天互动次数，0次=0分，20次+=40分 |
| 近期分 | 40分 | 最后互动距今天数，7天内=40分，90天+=0分 |
| 重要度分 | 20分 | 高=20分，中=12分，低=4分 |

**所需权限：** `mail:mail:read` `calendar:calendar.event:read` `contact:employee.id:readonly` `bitable:app` `bitable:app:read` `bitable:app:write` `im:message:send_as_bot`

---

### 2. 会议待办追踪

> 从飞书妙记中提取待办事项，通过飞书消息确认后自动创建飞书任务，定期检查完成情况。

**文件位置：** `.claude/skills/lark-workflow-meeting-todo/SKILL.md`

**触发方式：**

```
"会议待办"  "会后的待办"  "追踪待办"  "提取会议行动项"  "整理会议纪要待办"
```

**使用示例：**

```
你：帮我整理最近一周会议的待办事项
AI：[搜索过去7天的会议]
    找到 5 场会议，其中 3 场有纪要。
    ## 会议：Q2产品规划会（3/28）
    ### 我方待办
    | # | 待办内容 | 建议截止日 |
    |---|---------|-----------|
    | 1 | 把产品路线图发给张三 | 本周五 |
    | 2 | 调研竞品定价策略 | 下周三 |

你：确认 1 和 2
AI：已创建 2 个飞书任务：
    - [ ] 把产品路线图发给张三（截止：周五）
    - [ ] 调研竞品定价策略（截止：下周三）
```

**所需权限：** `vc:meeting:read` `minutes:minutes:readonly` `docx:document:read` `task:task:write` `task:task:read` `im:message:send_as_bot` `bitable:app:read` `bitable:app:write` `mail:mail:read`

---

### 3. 个人知识库

> 通过飞书消息发送链接，AI 自动抓取生成摘要和标签，存入飞书知识库，支持自然语言搜索和内容关联分析。

**文件位置：** `.claude/skills/lark-workflow-knowledge-base/SKILL.md`

**触发方式：**

```
"知识库"  "保存文章"  "搜索知识库"  "之前保存过"  "知识管理"
```

**使用示例：**

```
你：帮我保存这篇文章 https://example.com/ai-trends-2026
AI：正在抓取文章内容...
    标题：2026年AI行业十大趋势
    标签：AI、趋势、2026、行业分析、大模型
    发现 1 条关联内容：你三周前保存过《大模型成本下降分析》
    ✓ 已保存到飞书知识库。

你：关于 AI 芯片的文章有哪些？
AI：找到 3 条相关内容：
    1. AI芯片市场竞争格局分析（3/15保存）...
    2. 国产AI芯片突围之路（3/08保存）...
```

**所需权限：** `docx:document:create` `docx:document:read` `wiki:node:read` `im:message:send_as_bot` `im:message:readonly` `bitable:app` `bitable:app:read` `bitable:app:write` `drive:drive:read`

---

### 4. 社交媒体追踪

> 维护社交媒体账号配置，通过 AI 获取公开数据生成每日快照，支持长期趋势分析和跨平台对比。

**文件位置：** `.claude/skills/lark-workflow-social-tracker/SKILL.md`

**触发方式：**

```
"社交数据"  "平台数据"  "粉丝趋势"  "内容表现"  "数据追踪"
```

**使用示例：**

```
你：添加我的 YouTube 账号 @mychannel
AI：✓ 已添加 YouTube 账号 @mychannel

你：YouTube 今天 1000 粉丝了，帮我记录一下
AI：✓ 已记录 YouTube 快照：粉丝 1000（较上次 +12）

你：这周哪个平台增长最快？
AI：## 社交媒体趋势报告
    | 平台 | 起始粉丝 | 结束粉丝 | 增长率 |
    |------|---------|---------|--------|
    | YouTube | 950 | 1000 | +5.3% |
    | Instagram | 500 | 550 | +10% |
```

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `sheets:spreadsheet:create` `sheets:spreadsheet:write` `im:message:send_as_bot`

---

### 5. AI 商业顾问团

> 8 个 AI 专家角色每晚自动分析业务数据，汇总去重后发送编号清单到飞书，支持回复编号展开详细分析。

**文件位置：** `.claude/skills/lark-workflow-business-advisor/SKILL.md`

**触发方式：**

```
"商业分析"  "业务报告"  "顾问分析"  "夜间分析"  "业务建议"
```

**使用示例：**

```
你：帮我分析一下最近的业务状况
AI：## 今日商业顾问团报告
    ### 建议清单
    1. 【关系】张三（XX科技）已 45 天未联系，建议主动跟进。
    2. 【财务】投资组合本周下跌 5%，建议检查持仓比例。
    3. 【运营】7 个任务已过期未完成，其中 3 个超过 14 天。

你：第 3 条详细说说
AI：## 第 3 条展开分析
    ### 详细分析
    - 创意库中有 4 个"方案已完成"状态但未执行
    ### 建议行动步骤
    1. 优先执行创意库中评分最高的选题
    2. 本周至少完成 2 个待执行任务
```

**8 个顾问角色：** 财务顾问、营销顾问、增长顾问、运营顾问、内容策略顾问、关系/BD 顾问、竞争/行业顾问、综合风险顾问

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `docx:document:create` `im:message:send_as_bot` `task:task:read` `calendar:calendar.event:read`

---

### 6. 内容创作管道

> 在飞书群聊中标记内容创意，自动研究话题、检查重复选题、评估价值、生成完整方案。

**文件位置：** `.claude/skills/lark-workflow-content-pipeline/SKILL.md`

**触发方式：**

```
"内容创意"  "灵感记录"  "选题评估"  "内容方案"  "创意管道"
```

**使用示例：**

```
你：我有个内容创意，想写一篇关于 AI Agent 在企业落地的文章
AI：正在处理你的创意...
    ## 创意评估
    | 维度 | 评分 | 说明 |
    |------|------|------|
    | 时效性 | 8/10 | AI Agent 是当下热门话题 |
    | 差异化 | 6/10 | 已有不少类似内容，需找到独特角度 |
    | **总分** | **7.6/10** | |

你：是的，创建任务
AI：已创建任务：创作「AI Agent 落地实战」
    创意库中状态已更新为「执行中」。
```

**所需权限：** `im:message:readonly` `im:message:send_as_bot` `task:task:write` `docx:document:create` `drive:drive:read` `bitable:app` `bitable:app:read` `bitable:app:write`

---

### 7. 每日晨间简报

> 整合飞书日历、任务、邮件、CRM 联系人等数据，生成今日综合简报。

**文件位置：** `.claude/skills/lark-workflow-morning-brief/SKILL.md`

**触发方式：**

```
"晨间简报"  "今日安排"  "今天做什么"  "早报"  "今日概览"
```

**使用示例：**

```
你：今天有什么安排？
AI：## 2026年3月31日晨间简报（星期二）

    ### 日程安排（共 3 场会议）
    | 时间 | 事件 | 组织者 | 状态 |
    |------|------|--------|------|
    | 09:00-10:00 | Q2预算审批 | 张三 | 已接受 |
    | 14:00-15:00 | 技术方案讨论 | 李四 | 待确认 |

    ### 待办事项（共 4 项）
    - [ ] 发送产品路线图给张三（截止：今天）⚠️ 即将到期
    - [ ] 调研竞品定价策略（截止：周四）

    ### 小结
    - 张三的预算邮件需要今天回复
    - 空闲时段：10:00-14:00、15:00-16:00
```

**所需权限：** `calendar:calendar.event:read` `task:task:read` `mail:mail:read` `im:message:send_as_bot` `bitable:app:read`

---

### 8. 股票研究助手

> 四层架构：自选股监控、财报提醒、深度研究（4个AI分析师并行）、自动选股。

**文件位置：** `.claude/skills/lark-workflow-stock-analyzer/SKILL.md`

**触发方式：**

```
"股票分析"  "自选股"  "研究XX股票"  "持仓快照"  "财报分析"  "选股"
```

**使用示例：**

```
你：帮我深度分析一下 NVDA
AI：[启动4个AI分析师并行研究]
    ## NVIDIA (NVDA) 深度研究报告
    | 维度 | 评分 | 摘要 |
    |------|------|------|
    | 基本面 | 9 | 营收同比增长 122% |
    | 护城河 | 9 | CUDA 生态壁垒极高 |
    | 估值 | 6 | P/E 65x 高于历史均值 |
    | 风险 | 5 | 依赖 AI 投资周期 |
    ### 建议：当前持仓可继续持有，等待回调至 $120 以下分批建仓。
```

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `docx:document:create` `im:message:send_as_bot` `sheets:spreadsheet:write`

---

### 9. 多编程代理协同

> 支持竞赛模式、分工模式、流水线模式三种工作模式，通过飞书追踪任务和结果。

**文件位置：** `.claude/skills/lark-workflow-multi-agent-dev/SKILL.md`

**触发方式：**

```
"并行开发"  "多代理"  "竞赛模式"  "分工模式"  "流水线协作"  "代码对比"
```

**使用示例：**

```
你：用竞赛模式帮我实现一个用户登录模块
AI：## 竞赛结果：用户登录模块
    ### 方案 A（Opus）评分：8.5/10 — JWT + Refresh Token
    ### 方案 B（Sonnet）评分：7.0/10 — Session-based
    推荐：方案 A

你：用流水线模式帮我重构这个模块
AI：阶段 1（Opus）：重构代码... ✓
    阶段 2（Sonnet）：代码审查... ✓（发现 3 个改进建议）
    阶段 3（Haiku）：补充测试... ✓（12 个测试全部通过）
```

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `im:message:send_as_bot` `task:task:write`

---

### 10. 健康日记

> 通过飞书消息发送食物照片，AI 识别食物并记录，追踪饮食和身体状态关联。

**文件位置：** `.claude/skills/lark-workflow-health-diary/SKILL.md`

**触发方式：**

```
"记录饮食"  "健康日记"  "吃了什么"  "身体不舒服"  "健康追踪"
```

**使用示例：**

```
你：[发送一张午餐照片到飞书]
AI：看起来午餐是：红烧牛肉面 + 一杯冰美式
    请确认食物和份量。

你：下午胃有点不舒服
AI：⚠️ 提醒：你今天喝了冰美式，之前 3 次喝冰美式后 1-2 小时内都出现了胃部不适。

你：生成这周的健康报告
AI：## 健康周报
    ### 关联发现
    | 食物 | 状态 | 出现次数 | 关联强度 |
    |------|------|---------|---------|
    | 冰美式 | 胃部不适 | 3次 | 高 |
```

**所需权限：** `im:message:readonly` `im:message.resource:readonly` `im:message:send_as_bot` `bitable:app` `bitable:app:read` `bitable:app:write` `docx:document:create`

---

### 11. 任务状态播报

> 自动获取飞书任务列表，按状态分组（未开始/进行中/已完成/逾期）生成进度报告，支持推送到飞书群聊或保存为文档。

**文件位置：** `.claude/skills/lark-workflow-task-digest/SKILL.md`

**触发方式：**

```
"任务播报"  "任务状态"  "任务进度"  "今天有哪些任务"  "任务汇报"
```

**使用示例：**

```
你：帮我播报一下任务状态
AI：[获取任务列表]
    ## 任务状态播报
    ### 逾期（2 项）⚠️
    | 任务 | 截止日期 | 逾期天数 |
    |------|---------|---------|
    | 提交周报 | 3/28 | 3 天 |
    | 回复客户邮件 | 3/30 | 1 天 |

    ### 进行中（3 项）
    | 任务 | 截止日期 | 进度 |
    |------|---------|------|
    | 产品方案设计 | 4/5 | 60% |
    | 用户调研 | 4/3 | 30% |

    ### 未开始（1 项）
    | 任务 | 截止日期 |
    |------|---------|
    | 准备季度汇报 | 4/10 |

    ### 今日已完成（2 项）
    | 任务 | 完成时间 |
    |------|---------|
    | 修复登录 Bug | 10:30 |
    | 更新设计稿 | 14:15 |

你：发送到项目群
AI：✓ 已发送任务播报到项目群。
```

**所需权限：** `task:task:read` `im:message:send_as_bot` `docx:document:create`

---

### 12. 日程冲突检测

> 查询多个参会人的忙闲状态，推荐共同可用的会议时间段，一键创建日程。

**文件位置：** `.claude/skills/lark-workflow-meeting-finder/SKILL.md`

**触发方式：**

```
"找个时间开会"  "日程冲突"  "空闲时间"  "安排会议"  "什么时候都有空"
```

**使用示例：**

```
你：帮我找明天下午张三和李四都空闲的 1 小时
AI：[查询张三、李四的忙闲状态]

    ## 时间推荐（明天下午）

    | 时段 | 张三 | 李四 | 推荐 |
    |------|------|------|------|
    | 14:00-15:00 | 空闲 | 空闲 | ★★★ 推荐 |
    | 15:00-16:00 | 会议 | 空闲 | ✗ |
    | 16:00-17:00 | 空闲 | 空闲 | ★★ 备选 |
    | 17:00-18:00 | 空闲 | 会议 | ✗ |

    推荐时段：明天 14:00-15:00

你：就用 14:00-15:00 吧，主题是 Q2 产品规划会
AI：✓ 已创建日程「Q2 产品规划会」
    时间：明天 14:00-15:00
    参会人：张三、李四
```

**所需权限：** `calendar:calendar:read` `calendar:calendar.event:read` `calendar:freebusy:read` `contact:user.base:readonly` `im:message:send_as_bot`

---

### 13. 群消息日报

> 拉取指定群聊的消息，AI 提取关键信息（决策/待办/问题/讨论），生成结构化日报。

**文件位置：** `.claude/skills/lark-workflow-chat-digest/SKILL.md`

**触发方式：**

```
"群消息日报"  "群聊总结"  "群里今天聊了什么"  "群消息摘要"  "聊天记录整理"
```

**使用示例：**

```
你：帮我生成产品讨论群今天的消息日报
AI：[拉取群聊消息，分页处理]
    ## 群消息日报 — 产品讨论群（3/31）

    ### 关键决策
    1. Q2 优先做移动端适配，PC 端延后 — @张三 拍板

    ### 待办事项
    | 事项 | 负责人 | 截止 |
    |------|--------|------|
    | 输出移动端方案 | 李四 | 4/5 |
    | 评估第三方 SDK | 王五 | 4/3 |

    ### 未解决问题
    1. 接口鉴权方案待定，@赵六 需要和安全团队确认

    ### 活跃讨论
    - 技术选型：React Native vs Flutter（15 条消息）
    - 设计稿评审：首页改版方案（8 条消息）

    共处理 87 条消息，4 人参与讨论。

你：保存为文档
AI：✓ 已保存为飞书文档。
```

**所需权限：** `im:chat:readonly` `im:message:readonly` `im:message:send_as_bot` `docx:document:create`

---

### 14. 会议纪要分发

> 获取已完成会议的纪要内容，区分参会者和缺席者，分别发送完整纪要和摘要版。

**文件位置：** `.claude/skills/lark-workflow-meeting-notes/SKILL.md`

**触发方式：**

```
"分发会议纪要"  "发送纪要"  "会议总结分发"  "会后通知"  "纪要发给没参加的人"
```

**使用示例：**

```
你：帮我分发今天 Q2 规划会的纪要
AI：[获取会议详情和参会人列表]
    [区分出席者（5人）和缺席者（2人）]

    已发送纪要：
    ✓ 出席者（5人）→ 完整纪要
    ✓ 缺席者（2人）→ 摘要版（含关键决策 + 待办事项）

    ### 缺席者收到的内容：
    ## Q2 规划会 — 会议摘要
    ### 关键决策
    - 移动端优先，Q2 必须上线 v2.0
    - 新增 2 个 HC 用于客户端团队
    ### 待办事项
    | 事项 | 负责人 | 截止 |
    |------|--------|------|
    | 输出技术方案 | 李四 | 4/5 |
    | 准备招聘 JD | HR | 4/3 |
```

**所需权限：** `calendar:calendar:readonly` `calendar:event:readonly` `docx:document:readonly` `im:message:send_as_bot`

---

### 15. 新人入职向导

> 五系统联动：发欢迎消息、拉入相关群聊、创建入职任务清单、设置日历提醒，一站式完成新人引导。

**文件位置：** `.claude/skills/lark-workflow-onboard/SKILL.md`

**触发方式：**

```
"新人入职"  "入职引导"  "欢迎新同事"  "入职任务"  "入职流程"
```

**使用示例：**

```
你：新同事王五入职了，帮我走一下入职流程
AI：[查找王五的通讯录信息]
    [发送欢迎消息] ✓
    [拉入公司大群] ✓
    [拉入项目组群] ✓
    [创建入职任务清单] ✓
    [设置日历提醒] ✓

    ## 入职引导已完成

    ### 欢迎消息
    已发送到王五的飞书，包含：
    - 团队介绍和联系方式
    - 常用文档和系统链接

    ### 群聊（已拉入 2 个群）
    - 公司全员群
    - 产品研发群

    ### 入职任务清单（5 项）
    | # | 任务 | 截止日期 |
    |---|------|---------|
    | 1 | 完成入职信息登记 | 入职当天 |
    | 2 | 阅读团队文档 | 入职第 1 周 |
    | 3 | 安装开发环境 | 入职第 1 周 |
    | 4 | 与导师 1:1 沟通 | 入职第 3 天 |
    | 5 | 提交第一个 PR | 入职第 2 周 |

    ### 日历提醒
    - 明天 10:00：新员工欢迎会
    - 下周一 14:00：导师见面会
```

**所需权限：** `im:message:send_as_bot` `im:chat` `contact:user.base:readonly` `task:task:write` `calendar:calendar.event:write`

---

### 16. 文档权限管家

> 批量设置飞书文档权限，支持按角色或人员批量授权（可编辑/只读/完整权限），自动通知被授权用户。

**文件位置：** `.claude/skills/lark-workflow-doc-perm/SKILL.md`

**触发方式：**

```
"设置文档权限"  "批量授权"  "文档权限管理"  "分享文档给团队"  "把文档分享给XX"
```

**使用示例：**

```
你：把产品方案文档分享给张三、李四可编辑，王五只读
AI：[查找用户 open_id]
    [解析文档 token]

    正在设置权限...
    ✓ 张三 — 可编辑
    ✓ 李四 — 可编辑
    ✓ 王五 — 只读

    [通知被授权用户] ✓

    已通知 3 位同事：
    - 张三、李四收到可编辑权限通知
    - 王五收到只读权限通知
```

**支持的知识库链接自动解析：** 提供飞书知识库链接时，自动解析为真实文档 token 并设置权限。

**所需权限：** `drive:drive:write` `wiki:node:read` `contact:user.base:readonly` `im:message:send_as_bot`

---

### 17. 智能表单收集

> AI 根据需求自动设计表单字段，创建多维表格表单视图，配置问题，生成填写链接，分发到飞书群聊收集信息。

**文件位置：** `.claude/skills/lark-workflow-form-builder/SKILL.md`

**触发方式：**

```
"创建表单"  "信息收集"  "制作问卷"  "收集反馈"  "报名表"  "数据收集"
```

**使用示例：**

```
你：帮我创建一个需求收集表单
AI：根据需求，我设计了以下字段方案：

    | 字段名 | 类型 | 是否必填 |
    |--------|------|---------|
    | 需求标题 | 单行文本 | 是 |
    | 提出人 | 单行文本 | 是 |
    | 优先级 | 单选（高/中/低） | 是 |
    | 需求描述 | 多行文本 | 是 |
    | 期望上线日期 | 日期 | 否 |

    确认这个方案吗？

你：确认
AI：[创建数据表] ✓
    [创建表单视图] ✓
    [配置表单问题] ✓
    [获取表单链接] ✓

    ✓ 表单创建完成！
    表单链接：https://xxx.feishu.cn/base/xxx/form/xxx

    需要分发表单链接到哪个群聊？

你：发到产品需求群
AI：✓ 已分发表单到产品需求群。
```

**所需权限：** `bitable:app` `bitable:app:readonly` `im:message:send_as_bot`

---

### 18. 跨表数据同步

> 支持不同多维表格间的数据同步，AI 自动分析字段映射关系，智能去重，批量写入差异数据。

**文件位置：** `.claude/skills/lark-workflow-bitable-sync/SKILL.md`

**触发方式：**

```
"数据同步"  "跨表同步"  "表间复制"  "数据迁移"  "合并表格"
```

**使用示例：**

```
你：把表 A 的数据同步到表 B
AI：[获取表 A 字段结构]
    [获取表 B 字段结构]

    ## 字段映射
    | 表 A 字段 | 表 B 字段 | 匹配度 |
    |----------|----------|--------|
    | 客户名称 | 公司名 | 高 |
    | 联系方式 | 手机号 | 高 |
    | 备注 | 描述 | 中 |

    确认映射关系？

你：确认
AI：[分页拉取表 A 数据] 共 156 条
    [比对去重] 新增 23 条，重复 133 条
    [写入表 B] ✓

    ## 同步结果
    - 源表记录：156 条
    - 新增写入：23 条
    - 重复跳过：133 条（基于公司名+手机号去重）
    - 同步完成 ✓
```

**所需权限：** `bitable:app` `bitable:app:readonly`

---

### 19. 群公告管理

> 查看、设置、追加群公告内容，支持 Markdown 转 HTML，自动通知群成员公告更新。

**文件位置：** `.claude/skills/lark-workflow-announce/SKILL.md`

**触发方式：**

```
"群公告"  "发布公告"  "更新公告"  "设置群公告"  "追加公告"
```

**使用示例：**

```
你：看看产品群现在的公告
AI：## 当前群公告
    本周迭代计划 v3.2
    1. 完成登录模块重构
    2. 修复 3 个 P1 Bug

你：在公告末尾追加一条：周五下午 3 点全员会议
AI：[读取当前公告和 revision]
    [合并旧公告 + 新内容]
    ✓ 公告已更新，追加了新通知。

你：发个通知让大家知道
AI：✓ 已 @全体成员 发送公告更新通知。
```

**注意：** 群公告内容使用 HTML 格式，AI 会自动将 Markdown 转为 HTML。

**所需权限：** `im:chat:readonly` `im:chat` `im:message:send_as_bot`

---

### 20. 审批催办机器人

> 自动查询待审批实例，定位当前审批人，发送催办提醒，支持逐级升级催办策略（私聊 → 群 @ → 通知上级）。

**文件位置：** `.claude/skills/lark-workflow-approval-reminder/SKILL.md`

**触发方式：**

```
"催审批"  "催办"  "审批提醒"  "提醒审批人"  "跟进审批流程"  "审批到谁了"
```

**使用示例：**

```
你：帮我催一下那个报销审批，实例 ID 是 xxx
AI：[查询审批实例状态]
    ## 审批信息
    审批单：差旅报销 - 张三
    编号：202604090001
    状态：审批中
    当前审批人：李四（部门经理）

    [发送催办消息给李四] ✓

    催办消息已发送：
    "您好，您有一笔审批单待处理：
    审批单：差旅报销 - 张三
    请您尽快处理。"

你：审批到谁了？实例 ID 是 yyy
AI：[查询审批实例]
    审批单：采购申请 - 服务器
    当前状态：审批中
    审批流程：
    1. ✅ 发起人：王五（已提交 3/28）
    2. ✅ 直属上级：赵六（已通过 3/29）
    3. ⏳ 财务审批：钱七（待处理，已等待 48 小时）⚠️

    该审批已等待 48 小时，建议升级催办：
    1. 在财务群 @钱七
    2. 通知赵六协助推动

    是否升级催办？
```

**升级催办策略：**

| 等级 | 触发条件 | 方式 |
|------|---------|------|
| 第一级 | 正常催办 | 私聊消息 |
| 第二级 | 超过 24 小时 | 群内 @催办 |
| 第三级 | 超过 48 小时 | 通知上级/HR |

**所需权限：** `approval:instance:readonly` `contact:user.base:readonly` `im:message:send_as_bot`

---

### 21. 进销存 ERP

> 基于飞书多维表格搭建轻量进销存系统，管理货品台账、入库出库流水、库存预警和供应商信息。

**文件位置：** `.claude/skills/lark-workflow-erp-inventory/SKILL.md`

**触发方式：**

```
"进销存"  "库存管理"  "入库"  "出库"  "查库存"  "货品管理"  "供应商"
```

**使用示例：**

```
你：帮我创建一个进销存系统
AI：首次使用需要创建数据表，现在开始初始化...
    [创建多维表格：货品库存台账 + 入库记录表 + 出库记录表 + 供应商表]
    [配置字段、双向关联、Lookup 和公式字段]
    ✓ 进销存系统初始化完成！

你：新增货品：米色T恤，规格 S/M/L，最低库存 20
AI：[创建货品记录]
    ✓ 已添加货品：米色T恤（S/M/L），最低库存 20

你：入库：米色T恤-S 10件，单价50
AI：[查询货品是否存在]
    [创建入库记录]
    ✓ 入库成功：米色T恤-S，数量 +10，当前库存 10

你：出库：米色T恤-S 3件，用途领用
AI：[查询当前库存：10件，充足]
    [创建出库记录]
    ✓ 出库成功：米色T恤-S，数量 -3，剩余库存 7

你：查一下哪些货品库存不足
AI：## 低库存预警
    | 货品 | 当前库存 | 最低库存 | 状态 |
    |------|---------|---------|------|
    | 米色T恤-M | 5 | 20 | ⚠️ 低库存 |
    | 黑色卫衣-L | 0 | 15 | 🔴 缺货 |

你：本月入库汇总
AI：## 本月入库汇总
    | 货品 | 入库次数 | 总数量 | 总金额 |
    |------|---------|--------|--------|
    | 米色T恤-S | 3 | 50 | ¥2,500 |
    | 黑色卫衣-L | 1 | 20 | ¥1,800 |
    合计：4 次，70 件，¥4,300
```

**核心功能：**
- 4 张数据表：货品库存台账、入库记录表、出库记录表、供应商表
- 自动库存计算：Lookup 累计入库/出库 + 公式计算剩余库存
- 出库前校验：自动检查库存是否充足，不足则拦截
- 低库存预警：公式自动判断库存状态，支持定时通知
- 数据分析：入库汇总、出库统计、滞销分析、库存周转

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `im:message:send_as_bot`

---

### 22. 团队 CRM

> 面向小团队的微型销售 CRM，管理客户档案、商机漏斗、跟进记录，配自动提醒和销售仪表盘。

**文件位置：** `.claude/skills/lark-workflow-crm/SKILL.md`

**触发方式：**

```
"客户管理"  "商机跟进"  "销售漏斗"  "客户建档"  "赢单丢单"  "跟进提醒"  "销售日报"
```

**使用示例：**

```
你：帮我初始化团队CRM系统
AI：首次使用需要创建数据表，现在开始初始化...
    [创建多维表格：客户信息表 + 商机管理表 + 跟进记录表]
    [配置字段、双向关联、公式和 Lookup 字段]
    [创建商机阶段看板 + 仪表盘 + 客户咨询登记表单]
    ✓ 团队 CRM 系统初始化完成！

你：帮我建一个客户：北京星辰科技，对接人王总，手机138xxxx，互联网行业，100人左右，来自展会，重要程度高
AI：[解析提取各字段值]
    [写入客户信息表]
    ✓ 客户已建档：北京星辰科技有限公司

你：给北京星辰科技创建一个商机：企业IM系统采购，预计金额50万，阶段待接触
AI：[查找客户记录]
    [创建商机并关联客户]
    ✓ 商机已创建：企业IM系统采购，金额 ¥500,000，阶段：待接触

你：跟进记录：今天电话联系了王总，对方对方案比较满意，约下周三面访确认细节
AI：[创建跟进记录并关联客户和商机]
    ✓ 跟进记录已保存，下次跟进时间：下周三

你：看看当前的销售漏斗
AI：## 销售漏斗
    待接触 (5) → 需求确认 (3) → 方案报价 (2) → 商务谈判 (1) → 赢单 (8) → 丢单 (2)
    商机总金额：¥3,200,000（进行中 ¥1,500,000）

你：本月赢单了多少？
AI：## 本月赢单
    | 商机 | 客户 | 金额 | 负责人 |
    |------|------|------|--------|
    | ERP系统采购 | 星辰科技 | ¥500,000 | 张三 |
    | 云服务续约 | 蓝海集团 | ¥200,000 | 李四 |
    合计：2 单，¥700,000

你：生成今天的销售日报
AI：## 销售日报（2026/04/13）
    ### 今日跟进（3 条）
    - 张三 电话联系了星辰科技王总，推进到方案报价阶段
    - 李四 微信回复了蓝海集团，对方要重新评估预算
    ...
    ### 商机动态
    - 新增商机 1 个：蓝海集团 数据中台建设
    - 阶段推进 1 个：星辰科技 ERP采购 → 方案报价
```

**核心功能：**
- 3 张数据表：客户信息、商机管理、跟进记录
- 6 阶段漏斗：待接触 → 需求确认 → 方案报价 → 商务谈判 → 赢单/丢单
- 商机阶段看板（Kanban）：拖拽切换阶段
- 销售仪表盘：销售漏斗、赢单金额、阶段分布、负责人业绩排行
- 自动赢单概率：按阶段自动建议赢单概率（10%/30%/50%/75%/100%）
- 客户咨询登记表单：可分享链接收集客户信息
- 自动化提醒：每日跟进提醒、超时未跟进预警、赢单报喜、每周销售简报

**所需权限：** `bitable:app` `bitable:app:read` `bitable:app:write` `im:message:send_as_bot` `contact:employee.id:readonly`

---

## Skill 间数据联动关系

```
lark-workflow-personal-crm ─────┐
   (联系人数据)                 │
                                ├──► lark-workflow-morning-brief
lark-workflow-social-tracker ───┤   (简报读取各模块数据)
   (社交数据)                   │
                                ├──► lark-workflow-business-advisor
lark-workflow-content-pipeline ─┤   (顾问团读取所有模块数据)
   (创意/内容数据)              │
                                │
lark-workflow-stock-analyzer ───┘
   (投资数据)

lark-workflow-task-digest ────────► lark-task (读取任务状态)
lark-workflow-meeting-notes ──────► lark-calendar + lark-vc (获取会议信息)
lark-workflow-onboard ────────────► lark-task + lark-im + lark-calendar (多系统联动)
lark-workflow-doc-perm ───────────► lark-wiki + lark-drive (文档权限)
lark-workflow-form-builder ───────► lark-base (创建表单)
lark-workflow-bitable-sync ───────► lark-base (跨表数据同步)
lark-workflow-announce ───────────► lark-im (群公告管理)
lark-workflow-approval-reminder ──► lark-api + lark-im (审批催办)
lark-workflow-chat-digest ────────► lark-im (群消息)
lark-workflow-meeting-finder ─────► lark-calendar + lark-contact (日程冲突)
lark-workflow-meeting-todo ───────► lark-task (创建飞书任务)
lark-workflow-content-pipeline ──► lark-task (创建飞书任务)
lark-workflow-morning-brief ─────► lark-im (发送简报消息)
lark-workflow-business-advisor ──► lark-im + lark-doc (发送清单 + 存档报告)
lark-workflow-knowledge-base ────► lark-wiki + lark-doc (知识库存储)
lark-workflow-health-diary ──────► lark-doc (周报文档)
lark-workflow-stock-analyzer ────► lark-doc (分析报告)
lark-workflow-erp-inventory ─────► lark-base (进销存数据)
lark-workflow-crm ───────────────► lark-base + lark-im (客户/商机/跟进数据 + 通知)
```

**建议启用顺序：**

1. **先启用独立 Skill**（不依赖其他 Skill）：健康日记、社交媒体追踪、任务状态播报、日程冲突检测
2. **再启用基础 Skill**：个人 CRM、知识库、内容管道
3. **然后启用复合 Skill**：会议待办、股票分析、会议纪要分发、新人入职向导
4. **最后启用汇总 Skill**（需要其他 Skill 提供数据）：晨间简报、商业顾问团
5. **工具型 Skill** 随时可用：文档权限管家、智能表单收集、跨表数据同步、群公告管理、审批催办、群消息日报
6. **多代理协同**随时可用（与飞书 Skill 无强依赖）
7. **业务系统 Skill** 按需启用：进销存 ERP、团队 CRM

---

## 定时自动化配置

部分 Skill 支持设置定时自动执行，使用 Claude Code 的 CronCreate 功能：

### 每日晨间简报（工作日 8:00）

```
CronCreate: cron="0 8 * * 1-5", prompt="帮我生成今天的晨间简报并发送到飞书", recurring=true
```

### 社交媒体数据采集（每天 22:30）

```
CronCreate: cron="30 22 * * *", prompt="采集今天的社交媒体数据快照，记录到多维表格", recurring=true
```

### 商业顾问团分析（工作日 22:00）

```
CronCreate: cron="0 22 * * 1-5", prompt="运行商业顾问团分析，生成今日报告并发送到飞书", recurring=true
```

### 健康周报（每周日 20:00）

```
CronCreate: cron="0 20 * * 0", prompt="生成本周的健康报告并保存到飞书文档", recurring=true
```

### 任务状态播报（工作日 9:00）

```
CronCreate: cron="0 9 * * 1-5", prompt="播报今天的任务状态，发送到项目群", recurring=true
```

### 群消息日报（工作日 18:00）

```
CronCreate: cron="0 18 * * 1-5", prompt="生成产品讨论群今天的消息日报", recurring=true
```

### CRM 每日跟进提醒（工作日 9:00）

```
CronCreate: cron="0 9 * * 1-5", prompt="检查今日需要跟进的客户和商机，发送提醒给对应负责人", recurring=true
```

### CRM 每周销售简报（每周一 9:00）

```
CronCreate: cron="0 9 * * 1", prompt="生成上周销售简报，汇总新增客户、赢单情况、漏斗数据，发送到销售群", recurring=true
```

> CronCreate 创建的定时任务会在 Claude Code 运行时自动触发。如需持久化，请设置 `durable: true`。

---

## 权限清单

以下列出所有 Skill 需要的飞书开放平台权限 scope，需要在[飞书开发者后台](https://open.feishu.cn)配置：

### 必备权限（几乎所有 Skill 都需要）

| scope | 说明 | 涉及 Skill |
|-------|------|-----------|
| `bitable:app` | 创建多维表格 | 全部（数据初始化） |
| `bitable:app:read` | 读取多维表格 | 全部（数据查询） |
| `bitable:app:write` | 写入多维表格 | 全部（数据记录） |
| `im:message:send_as_bot` | 机器人发送消息 | 全部（通知推送） |

### 模块权限（按需开启）

| scope | 说明 | 涉及 Skill |
|-------|------|-----------|
| `calendar:calendar.event:read` | 读取日历 | 晨间简报、CRM、商业顾问团、日程冲突检测、会议纪要分发 |
| `calendar:calendar.event:write` | 创建日程 | 日程冲突检测、新人入职 |
| `calendar:freebusy:read` | 查询忙闲 | 日程冲突检测 |
| `task:task:read` | 读取任务 | 晨间简报、商业顾问团、任务状态播报 |
| `task:task:write` | 创建/更新任务 | 会议待办、内容管道、多代理、新人入职 |
| `mail:mail:read` | 读取邮件 | 晨间简报、CRM |
| `docx:document:create` | 创建文档 | 知识库、商业顾问团、股票分析、健康日记、内容管道、任务播报、群消息日报 |
| `docx:document:read` | 读取文档 | 会议待办、知识库、会议纪要分发 |
| `drive:drive:read` | 读取云空间 | 知识库、内容管道 |
| `drive:drive:write` | 写入云空间 | 文档权限管家 |
| `vc:meeting:read` | 读取会议 | 会议待办 |
| `minutes:minutes:readonly` | 读取妙记 | 会议待办 |
| `wiki:node:read` | 读取知识库节点 | 知识库、文档权限管家 |
| `contact:user.base:readonly` | 查询通讯录 | CRM、日程冲突检测、文档权限管家、审批催办、新人入职 |
| `contact:employee.id:readonly` | 查询员工ID | 团队 CRM |
| `approval:instance:readonly` | 读取审批实例 | 审批催办 |
| `im:message:readonly` | 读取消息 | 知识库、内容管道、健康日记、群消息日报 |
| `im:message.resource:readonly` | 下载消息资源(图片) | 健康日记 |
| `im:chat:readonly` | 读取群信息 | 群消息日报、群公告管理 |
| `im:chat` | 管理群聊 | 群公告管理、新人入职 |
| `sheets:spreadsheet:create` | 创建电子表格 | 社交追踪 |
| `sheets:spreadsheet:write` | 写入电子表格 | 社交追踪、股票分析 |

### 一键授权

```bash
lark-cli auth login --domain im,calendar,task,mail,base,docs,wiki,drive,vc,minutes,contact,sheets
```

> **注意：** 审批 API 需要额外通过 `--scope` 参数授权：`lark-cli auth login --domain im,contact --scope "approval:instance:readonly"`

---

## 常见问题

### Q: 首次使用某个 Skill 时报错怎么办？

A: 大概率是权限未授权。运行以下命令：

```bash
lark-cli auth login --domain <需要的模块>
```

或者按错误提示中的 `hint` 字段执行。

### Q: 数据存在哪里？

A: 所有 Skill 的持久化数据都存储在**飞书多维表格 (Base)** 中。首次使用时 AI 会引导你创建表格，你也可以指定已有的 Base。数据归属你的飞书账号，可在飞书中直接查看和编辑。

### Q: 如何更换接收通知的飞书群聊？

A: 对 AI 说"把通知发送到 XXX 群"，AI 会通过 `im +chat-search` 搜索群聊 ID。

### Q: Skill 之间数据不互通怎么办？

A: 确保：
1. 相关 Skill 都已完成数据初始化（创建了多维表格）
2. 使用同一个 Base 存储数据（AI 会提示你选择或创建）

### Q: 定时任务不执行怎么办？

A: 定时任务依赖 Claude Code 进程运行。确保 Claude Code 正在运行中。如需持久化定时任务，在创建时设置 `durable: true`。

### Q: 社交媒体和股票数据不准确？

A: 当前阶段这些数据通过 AI WebSearch 获取公开信息，可能存在延迟或不准确。你可以：
1. 手动录入数据（直接告诉 AI 数值）
2. 等待后续版本接入各平台官方 API

### Q: 如何删除/重置某个 Skill 的数据？

A: 直接在飞书中打开对应的多维表格，删除记录或整个表即可。重新使用时会自动引导重新初始化。

### Q: Windows 下 `lark-cli api` 调用报路径错误？

A: Windows Git Bash 会将 `/open-apis/...` 转换为本地路径。在命令前加上 `MSYS_NO_PATHCONV=1` 即可：

```bash
MSYS_NO_PATHCONV=1 lark-cli api GET "/open-apis/approval/v4/instances/xxx" --as user
```

---

## 文件结构

```
.claude/skills/
├── lark-workflow-personal-crm/SKILL.md       # 用法1: 个人CRM
├── lark-workflow-meeting-todo/SKILL.md       # 用法2: 会议待办追踪
├── lark-workflow-knowledge-base/SKILL.md     # 用法3: 个人知识库
├── lark-workflow-social-tracker/SKILL.md     # 用法4: 社交媒体追踪
├── lark-workflow-business-advisor/SKILL.md   # 用法5: AI商业顾问团
├── lark-workflow-content-pipeline/SKILL.md   # 用法6: 内容创作管道
├── lark-workflow-morning-brief/SKILL.md      # 用法7: 每日晨间简报
├── lark-workflow-stock-analyzer/SKILL.md     # 用法8: 股票研究助手
├── lark-workflow-multi-agent-dev/SKILL.md    # 用法9: 多编程代理协同
├── lark-workflow-health-diary/SKILL.md       # 用法10: 健康日记
├── lark-workflow-task-digest/SKILL.md        # 用法11: 任务状态播报
├── lark-workflow-meeting-finder/SKILL.md     # 用法12: 日程冲突检测
├── lark-workflow-chat-digest/SKILL.md        # 用法13: 群消息日报
├── lark-workflow-meeting-notes/SKILL.md      # 用法14: 会议纪要分发
├── lark-workflow-onboard/SKILL.md            # 用法15: 新人入职向导
├── lark-workflow-doc-perm/SKILL.md           # 用法16: 文档权限管家
├── lark-workflow-form-builder/SKILL.md       # 用法17: 智能表单收集
├── lark-workflow-bitable-sync/SKILL.md       # 用法18: 跨表数据同步
├── lark-workflow-announce/SKILL.md           # 用法19: 群公告管理
├── lark-workflow-approval-reminder/SKILL.md  # 用法20: 审批催办机器人
├── lark-workflow-erp-inventory/SKILL.md      # 用法21: 进销存ERP
├── lark-workflow-crm/SKILL.md                # 用法22: 团队CRM
├── lark-workflow-meeting-summary/SKILL.md    # (已有) 会议纪要汇总
├── lark-workflow-standup-report/SKILL.md     # (已有) 日程待办摘要
├── lark-im/SKILL.md                          # (底层) 消息收发
├── lark-mail/SKILL.md                        # (底层) 邮件
├── lark-calendar/SKILL.md                    # (底层) 日历
├── lark-task/SKILL.md                        # (底层) 任务
├── lark-vc/SKILL.md                          # (底层) 会议
├── lark-minutes/SKILL.md                     # (底层) 妙记
├── lark-doc/SKILL.md                         # (底层) 文档
├── lark-wiki/SKILL.md                        # (底层) 知识库
├── lark-base/SKILL.md                        # (底层) 多维表格
├── lark-sheets/SKILL.md                      # (底层) 电子表格
├── lark-contact/SKILL.md                     # (底层) 通讯录
├── lark-drive/SKILL.md                       # (底层) 云空间
├── lark-event/SKILL.md                       # (底层) 事件订阅
├── lark-whiteboard/SKILL.md                  # (底层) 白板
├── lark-shared/SKILL.md                      # (底层) 共享基础
├── lark-skill-maker/SKILL.md                 # (底层) Skill创建器
└── lark-openapi-explorer/SKILL.md            # (底层) OpenAPI探索
```
