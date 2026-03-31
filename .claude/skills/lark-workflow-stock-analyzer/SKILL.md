---
name: lark-workflow-stock-analyzer
version: 1.0.0
description: "股票研究助手：四层架构（自选股监控/财报提醒/深度研究/自动选股），多维表格管理持仓和自选股，AI多角色并行分析生成报告。当用户需要'股票分析'、'自选股'、'研究XX股票'、'持仓快照'、'财报分析'、'选股'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 股票研究助手工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我分析一下苹果的股票" / "研究一下 AAPL"
- "我的自选股有哪些？" / "添加/删除自选股"
- "生成持仓快照" / "今天赚了多少"
- "这周有哪些财报发布？" / "分析财报"
- "帮我选几只好股票" / "技术面选股"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,docs,im,sheets
```

## 工作流总览

```
四层架构：

第一层：自选股监控
  base +record-list ──► 检查目标价 ──► WebSearch获取行情 ──► im +messages-send（到达通知）

第二层：财报提醒
  WebSearch ──► 查询财报日程 ──► 到期时自动分析 ──► doc +create（财报摘要）

第三层：深度研究
  启动 4 个 AI 分析师并行 ──► 各自分析 ──► 汇总报告 ──► doc +create

第四层：自动选股
  用户设定条件 ──► WebSearch 全市场扫描 ──► 筛选匹配 ──► 报告
```

---

## 数据初始化

### Base 表结构

**表 1：自选股 (watchlist)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 股票代码 | 文本 | 如 AAPL、600519.SH |
| 股票名称 | 文本 | 如 Apple、贵州茅台 |
| 市场 | 单选 | 美股/A股/港股/其他 |
| 目标价 | 数字 | 用户设定的目标价格 |
| 目标方向 | 单选 | 看涨(到达买入)/看跌(到达卖出) |
| 当前价 | 数字 | 最新行情价格（定期更新） |
| 上次更新 | 日期 | 价格最后更新时间 |
| 备注 | 文本 | 自由备注 |
| 加入时间 | 日期 | 加入自选股的时间 |

**表 2：持仓 (portfolio)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 股票代码 | 文本 | 如 AAPL |
| 股票名称 | 文本 | 如 Apple |
| 买入价 | 数字 | 平均买入价格 |
| 持仓数量 | 数字 | 持有股数 |
| 买入日期 | 日期 | 建仓日期 |
| 当前价 | 数字 | 最新价格 |
| 市值 | 公式 | 持仓数量 × 当前价 |
| 盈亏金额 | 公式 | 市值 - (买入价 × 持仓数量) |
| 盈亏比例 | 公式 | 盈亏金额 / (买入价 × 持仓数量) |

**表 3：分析记录 (analysis_log)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 股票代码 | 文本 | 如 AAPL |
| 分析类型 | 单选 | 深度研究/财报分析/日常监控/选股结果 |
| 分析日期 | 日期 | 分析时间 |
| 结论 | 单选 | 强烈看多/看多/中性/看空/强烈看空 |
| 信心评分 | 数字 | 1-10 |
| 看多理由 | 文本 | 主要看多论点 |
| 看空理由 | 文本 | 主要看空论点 |
| 建议 | 文本 | 操作建议 |
| 报告链接 | URL | 详细分析报告的飞书文档链接 |

**表 4：财报日程 (earnings_calendar)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 股票代码 | 文本 | |
| 股票名称 | 文本 | |
| 财报日期 | 日期 | 预计发布日期 |
| 财报类型 | 单选 | 季报/年报 |
| 已分析 | 单选 | 否/已分析 |
| 分析链接 | URL | 财报分析报告链接 |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "自选股" --fields '[
  {"name":"股票代码","type":"text"},
  {"name":"股票名称","type":"text"},
  {"name":"市场","type":"select","multiple":false,"options":[{"name":"美股"},{"name":"A股"},{"name":"港股"},{"name":"其他"}]},
  {"name":"目标价","type":"number"},
  {"name":"目标方向","type":"select","multiple":false,"options":[{"name":"看涨(到达买入)"},{"name":"看跌(到达卖出)"}]},
  {"name":"当前价","type":"number"},
  {"name":"上次更新","type":"datetime"},
  {"name":"备注","type":"text"},
  {"name":"加入时间","type":"datetime"}
]'
lark-cli base +table-create --name "持仓" --fields '[
  {"name":"股票代码","type":"text"},
  {"name":"股票名称","type":"text"},
  {"name":"买入价","type":"number"},
  {"name":"持仓数量","type":"number"},
  {"name":"买入日期","type":"datetime"},
  {"name":"当前价","type":"number"}
]'
# 注：市值、盈亏金额、盈亏比例 为公式字段，需单独用 +field-create 创建（type="formula" 需要引用其他字段 ID）
lark-cli base +table-create --name "分析记录" --fields '[
  {"name":"股票代码","type":"text"},
  {"name":"分析类型","type":"select","multiple":false,"options":[{"name":"深度研究"},{"name":"财报分析"},{"name":"日常监控"},{"name":"选股结果"}]},
  {"name":"分析日期","type":"datetime"},
  {"name":"结论","type":"select","multiple":false,"options":[{"name":"强烈看多"},{"name":"看多"},{"name":"中性"},{"name":"看空"},{"name":"强烈看空"}]},
  {"name":"信心评分","type":"number"},
  {"name":"看多理由","type":"text"},
  {"name":"看空理由","type":"text"},
  {"name":"建议","type":"text"},
  {"name":"报告链接","type":"link"}
]'
lark-cli base +table-create --name "财报日程" --fields '[
  {"name":"股票代码","type":"text"},
  {"name":"股票名称","type":"text"},
  {"name":"财报日期","type":"datetime"},
  {"name":"财报类型","type":"select","multiple":false,"options":[{"name":"季报"},{"name":"年报"}]},
  {"name":"已分析","type":"select","multiple":false,"options":[{"name":"否"},{"name":"已分析"}]},
  {"name":"分析链接","type":"link"}
]'
```

> **限流处理**：如果 +table-create 因限流部分失败，表仍会创建成功。此时：
> 1. 用 `base +table-list` 获取 table_id
> 2. 用 `base +field-create --table-id <id> --json '{"name":"字段名","type":"text"}'` 逐个补字段，每次间隔 1-2 秒

---

## 第一层：自选股监控

### 管理自选股

```bash
# 添加自选股
lark-cli base +record-upsert --table-id "<watchlist_table_id>" --json '{
  "股票代码": "AAPL",
  "股票名称": "Apple",
  "市场": "美股",
  "目标价": 200,
  "目标方向": "看涨"
}'

# 查看自选股列表
lark-cli base +record-list --table-id "<watchlist_table_id>"
```

### 价格检查

通过 AI WebSearch 获取最新行情：

**AI 处理：**
1. 获取自选股列表
2. 逐一搜索最新股价
3. 对比目标价，判断是否触发
4. 触发时发送飞书消息通知

### 通知格式

```
目标价到达通知！

{股票名称}（{股票代码}）
当前价格：${current_price}
目标价格：${target_price}
目标方向：{看涨/看跌}
```

---

## 第二层：财报提醒

### 查询财报日程

```bash
# 获取自选股列表
lark-cli base +record-list --table-id "<watchlist_table_id>"
```

**AI 处理：**
1. 通过 WebSearch 查询自选股中各公司的财报发布日期
2. 记录到财报日程表
3. 到发布日时自动触发财报分析

### 财报分析

当财报发布时：

```
AI 财报分析师处理：
1. 搜索最新财报数据（营收、利润、EPS、同比增长等）
2. 提取管理层关键表态
3. 分析市场反应（盘后/次日股价变动）
4. 与分析师预期对比（beat/miss）
5. 生成通俗易懂的财报摘要
```

### 财报摘要格式

```
## {公司名称}（{股票代码}）{季度}财报摘要

### 核心数据
| 指标 | 实际值 | 预期值 | 变化 |
|------|--------|--------|------|
| 营收 | $XX亿 | $XX亿 | +X% |
| EPS | $X.XX | $X.XX | +X% |

### 管理层说了什么
- {要点1}
- {要点2}

### 市场反应
- 盘后涨跌：{幅度}
- 分析师评价：{摘要}

### 综合评价
{总结性判断}
```

保存为飞书文档：
```bash
lark-cli docs +create --title "财报摘要：{公司} {季度}" --markdown "<内容>"
```

---

## 第三层：深度研究

### 启动多角色分析

对指定股票启动 4 个 AI 分析师并行工作：

**分析师 1：基本面分析师**
- 营收、利润增长趋势
- 毛利率、净利率变化
- 自由现金流
- 债务水平

**分析师 2：护城河分析师**
- 品牌价值
- 技术壁垒/专利
- 网络效应
- 转换成本
- 规模优势

**分析师 3：估值分析师**
- P/E、P/B、P/S 等相对估值
- DCF 绝对估值
- 与同行业对比
- 历史估值区间

**分析师 4：风险分析师**
- 业务风险
- 竞争风险
- 监管风险
- 宏观风险
- 技术颠覆风险

### 汇总报告

```
## {公司名称}（{股票代码}）深度研究报告

### 总览
| 维度 | 评分(1-10) | 摘要 |
|------|-----------|------|
| 基本面 | {分} | {一句话} |
| 护城河 | {分} | {一句话} |
| 估值 | {分} | {一句话} |
| 风险 | {分} | {一句话} |

### 看多理由
1. {理由}
2. {理由}
3. {理由}

### 看空理由
1. {理由}
2. {理由}
3. {理由}

### 信心评分：{分}/10
### 建议：{操作建议}
```

保存报告：
```bash
lark-cli docs +create --title "深度研究：{公司}" --markdown "<报告>"
lark-cli base +record-upsert --table-id "<analysis_table_id>" --json '{"字段名":"值"}'
```

---

## 第四层：自动选股

### 用户设定条件

用户描述选股条件：
- "帮我找技术面超卖的蓝筹股"
- "ROE 大于 15%、PE 小于 20 的公司"
- "最近一个月涨幅最大的科技股"

### AI 执行选股

```
1. 将自然语言条件转化为可执行的搜索策略
2. WebSearch 获取市场数据
3. 根据条件筛选匹配
4. 生成选股结果报告
```

### 结果报告

```
## 选股结果

### 筛选条件
{原始条件 + 转化后的可执行条件}

### 符合条件的股票
| 排名 | 股票 | 价格 | 涨跌幅 | 关键指标 |
|------|------|------|--------|---------|
| 1 | {股票} | ${价} | {涨跌}% | {指标} |

### 说明
{筛选逻辑说明和数据来源}
```

---

## 持仓快照

定期生成持仓概览：

```bash
lark-cli base +record-list --table-id "<portfolio_table_id>"
```

AI 计算汇总：
- 总市值
- 总盈亏金额和比例
- 最佳/最差表现股票
- 持仓分布

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `docs +create` | `docx:document:create` |
| `im +messages-send` | `im:message:send_as_bot` |
| `sheets +write` | `sheets:spreadsheet:write` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert` 详细用法
- [`lark-doc`](../lark-doc/SKILL.md) — `+create` 详细用法
- [`lark-im`](../lark-im/SKILL.md) — `+messages-send` 详细用法
- [`lark-sheets`](../lark-sheets/SKILL.md) — `+write` 详细用法
