---
name: lark-workflow-crm
version: 1.1.0
description: "微型团队CRM：基于飞书多维表格的客户管理、商机漏斗、跟进记录、看板视图、仪表盘和自动化提醒。当用户需要以下操作时触发：系统搭建（'创建CRM'、'搭建销售管理系统'、'新建CRM系统'）；客户管理（'新增客户'、'客户建档'、'客户列表'、'修改客户'、'删除客户'）；商机管理（'创建商机'、'商机列表'、'商机跟进'、'推进商机'、'赢单'、'丢单'）；跟进记录（'记录跟进'、'跟进记录'、'销售日报'、'沟通记录'）；漏斗查询（'销售漏斗'、'商机阶段'、'本月赢单'、'业绩统计'）；自动化提醒（'跟进提醒'、'超时预警'、'赢单报喜'、'周报'）；表单收集（'创建客户登记表'、'咨询表单'）；仪表盘（'销售看板'、'创建仪表盘'）。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 微型团队 CRM 工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我创建一个CRM系统" / "搭建销售管理" / "新建CRM"
- "新增客户：北京星辰科技，对接人王总" / "客户建档"
- "给北京星辰创建商机：ERP采购，50万" / "商机跟进"
- "跟进记录：今天电话联系了王总，约下周三面访"
- "看看销售漏斗" / "本月赢单了多少" / "业绩排行"
- "生成今天的销售日报" / "本周跟进汇总"
- "创建客户登记表单" / "咨询表单"
- "创建销售看板" / "销售仪表盘"
- "跟进提醒" / "超时预警" / "赢单报喜" / "每周销售简报"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,im,contact
```

## 系统架构

```
微型团队 CRM (lark-workflow-crm)
│
├── 数据层：飞书多维表格 (Bitable)
│   ├── 客户信息主表 (customers)        — 客户档案
│   ├── 商机管理表 (opportunities)      — 销售漏斗
│   └── 跟进记录表 (follow_ups)         — 沟通记录
│
├── 计算层
│   ├── 赢单概率建议 (formula)          — 根据商机阶段自动建议概率
│   ├── 商机天数 (formula)              — 距商机创建的天数
│   ├── 客户商机数量 (formula)          — 关联商机计数
│   ├── 客户累计赢单金额 (formula)       — 关联赢单金额汇总
│   └── 商机跟进次数 (formula)          — 关联跟进计数
│
├── 关联关系（双向）
│   ├── customers ↔ opportunities       — 一个客户多个商机
│   ├── customers ↔ follow_ups          — 跟进记录关联客户
│   └── opportunities ↔ follow_ups      — 跟进记录关联商机
│
├── 视图层
│   ├── 商机阶段看板 (Kanban)           — 按阶段分列拖拽
│   ├── 我的商机                        — 当前用户负责的商机
│   ├── 跟进时间线                      — 按时间倒序的跟进记录
│   └── 本月赢单                        — 当月赢单列表
│
├── 仪表盘
│   ├── 销售漏斗 (funnel)               — 各阶段商机数量
│   ├── 本月赢单金额 (statistics)        — SUM WHERE 阶段=赢单
│   ├── 商机总金额 (statistics)          — 在途商机金额
│   ├── 客户数量 (statistics)            — 总客户数
│   ├── 阶段分布 (pie)                  — 各阶段占比
│   ├── 负责人业绩排行 (column)          — 赢单金额排行
│   └── 本周跟进 (table)                — 本周跟进记录
│
├── 表单
│   └── 客户咨询登记表                  — 客户自助提交咨询
│
├── 交互层：用户自然语言指令
│   ├── 系统初始化 → 创建 Base + 表 + 字段 + 关联 + 视图 + 仪表盘 + 表单
│   ├── 日常操作 → 客户建档 / 商机管理 / 跟进记录
│   └── 数据分析 → 漏斗查询 / 业绩统计 / 销售日报
│
└── 通知层：飞书消息
    ├── 每日跟进提醒
    ├── 超时未跟进预警
    ├── 赢单报喜通知
    └── 每周销售简报
```

---

## Agent 核心规则

1. **只使用 `lark-cli base +...` 原子命令** — 不走 `lark-cli api` 直接调用
2. **建表必须先建空表再逐个加字段** — `+table-create` 不带 `--fields` 参数，先建空表再用 `+field-create` 逐个加字段，每次间隔 1.5 秒
3. **关联字段必须使用双向关联** — `bidirectional: true`，否则无法在公式中做跨表聚合
4. **跨表计算优先使用 formula 字段** — 公式字段是 lookup 的超集，能用 formula 就不用 lookup
5. **公式字段创建前先读 guide** — 读取 [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md)
6. **`+data-query` 的 table-id 放在 DSL 中** — 不支持 `--table-id` 参数，需放在 `--dsl` 的 `datasource.table.tableId` 中
7. **`+data-query` 只支持存储字段** — 不支持公式字段、lookup 字段、关联字段做聚合
8. **写记录前先拿字段结构** — `+field-list` 确认字段名和类型
9. **表名/字段名精确匹配** — 禁止凭自然语言猜测，必须来自 `+table-list` / `+field-list` 返回值
10. **`+xxx-list` 禁止并发** — 所有 list 命令串行执行
11. **写入串行 + 间隔** — `+record-upsert` 串行，间隔 0.5-1 秒
12. **分页拉取** — `+record-list --limit 200`，用 offset 分页
13. **跟进记录中提及阶段变更时同步更新商机** — 用户说"推进到方案报价"时，更新对应商机的商机阶段字段
14. **创建仪表盘前先读 guide** — 读取 [`../lark-base/references/dashboard-block-data-config.md`](../lark-base/references/dashboard-block-data-config.md)
15. **创建表单前先读 guide** — 读取 [`../lark-base/references/lark-base-form-create.md`](../lark-base/references/lark-base-form-create.md)

---

## Phase 1：系统初始化

### Step 1.1: 创建 Base

```bash
lark-cli base +base-create --name "微型团队CRM" --as user
```

记录返回的 `base_token`（后续所有命令需要）。

### Step 1.2: 创建数据表（先建空表，间隔 2 秒）

> **IMPORTANT**：不要使用 `--fields` 参数一次性创建多字段，容易触发限流。

```bash
# 表 1：客户信息主表
lark-cli base +table-create --base-token "<base_token>" --name "客户信息" --as user
sleep 2

# 表 2：商机管理表
lark-cli base +table-create --base-token "<base_token>" --name "商机管理" --as user
sleep 2

# 表 3：跟进记录表
lark-cli base +table-create --base-token "<base_token>" --name "跟进记录" --as user
```

### Step 1.3: 获取各表 table_id

```bash
lark-cli base +table-list --base-token "<base_token>" --as user
```

记录各表的 `table_id`。

### Step 1.4: 逐个添加存储字段（每创建一个间隔 1.5 秒）

#### 客户信息表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"客户名称"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"客户类型","multiple":false,"options":[{"name":"个人客户"},{"name":"企业客户"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"联系人"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"手机号","style":{"type":"phone"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"邮箱","style":{"type":"email"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"行业","multiple":false,"options":[{"name":"互联网"},{"name":"金融"},{"name":"教育"},{"name":"制造业"},{"name":"零售"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"公司规模","multiple":false,"options":[{"name":"1-10人"},{"name":"11-50人"},{"name":"51-200人"},{"name":"200人以上"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"来源渠道","multiple":false,"options":[{"name":"主动咨询"},{"name":"转介绍"},{"name":"展会"},{"name":"线上广告"},{"name":"社群"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"地址"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"user","name":"负责人","multiple":false}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"重要程度","multiple":false,"options":[{"name":"高"},{"name":"中"},{"name":"低"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"select","name":"客户状态","multiple":false,"options":[{"name":"活跃"},{"name":"沉睡"},{"name":"流失"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" --json '{"type":"text","name":"备注"}' --as user
sleep 2
```

#### 商机管理表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"text","name":"商机名称"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"number","name":"商机金额","style":{"type":"currency","precision":2,"currency_code":"CNY"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"select","name":"商机阶段","multiple":false,"options":[{"name":"待接触"},{"name":"需求确认"},{"name":"方案报价"},{"name":"商务谈判"},{"name":"赢单"},{"name":"丢单"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"user","name":"负责人","multiple":false}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"datetime","name":"预计成交日期","style":{"format":"yyyy-MM-dd"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"text","name":"竞争对手"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"select","name":"丢单原因","multiple":false,"options":[{"name":"价格"},{"name":"功能"},{"name":"竞品"},{"name":"无预算"},{"name":"项目取消"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"select","name":"商机来源","multiple":false,"options":[{"name":"新开发"},{"name":"老客户增购"},{"name":"转介绍"}]}' --as user
sleep 2

# 添加系统字段：创建时间（公式字段"商机天数"依赖此字段）
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" --json '{"type":"created_at","name":"创建时间","style":{"format":"yyyy-MM-dd HH:mm"}}' --as user
```

#### 跟进记录表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"select","name":"跟进方式","multiple":false,"options":[{"name":"电话"},{"name":"微信"},{"name":"面访"},{"name":"邮件"},{"name":"视频会议"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"text","name":"跟进摘要"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"text","name":"客户反馈"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"text","name":"下一步计划"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"datetime","name":"下次跟进时间","style":{"format":"yyyy-MM-dd HH:mm"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"user","name":"跟进人","multiple":false}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"select","name":"商机阶段变更","multiple":false,"options":[{"name":"无变更"},{"name":"推进到下一阶段"},{"name":"退回上一阶段"},{"name":"赢单"},{"name":"丢单"}]}' --as user
sleep 1.5

# 添加系统字段：创建时间（公式字段"最后跟进时间"依赖此字段）
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" --json '{"type":"created_at","name":"创建时间","style":{"format":"yyyy-MM-dd HH:mm"}}' --as user
```

### Step 1.5: 创建双向关联字段（每创建一个间隔 2 秒）

> **CRITICAL**：关联字段必须使用 `bidirectional: true`，否则无法在公式/lookup 中做跨表聚合。

```bash
# 商机管理表 → 客户信息表
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"type":"link","name":"关联客户","link_table":"客户信息","bidirectional":true,"bidirectional_link_field_name":"关联商机"}' \
  --as user
sleep 2

# 跟进记录表 → 客户信息表
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --json '{"type":"link","name":"关联客户","link_table":"客户信息","bidirectional":true,"bidirectional_link_field_name":"跟进记录"}' \
  --as user
sleep 2

# 跟进记录表 → 商机管理表
lark-cli base +field-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --json '{"type":"link","name":"关联商机","link_table":"商机管理","bidirectional":true,"bidirectional_link_field_name":"跟进记录"}' \
  --as user
sleep 2
```

### Step 1.6: 创建公式字段

> **IMPORTANT**：创建公式字段前 MUST 先读取 [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md)，创建后追加 `--i-have-read-guide`

#### 商机管理表：赢单概率建议

根据商机阶段自动建议赢单概率。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"type":"formula","name":"赢单概率建议","expression":"SWITCH([商机阶段], \"待接触\", 10, \"需求确认\", 30, \"方案报价\", 50, \"商务谈判\", 75, \"赢单\", 100, 0)"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 商机管理表：商机天数

距商机创建的天数。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"type":"formula","name":"商机天数","expression":"DAYS(NOW(), [创建时间])"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 客户信息表：客户商机数量

通过关联字段跨表计数。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" \
  --json '{"type":"formula","name":"商机数量","expression":"COUNTA([关联商机])"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 客户信息表：客户累计赢单金额

通过公式跨表条件聚合。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" \
  --json '{"type":"formula","name":"累计赢单金额","expression":"SUM([关联商机].[商机金额].FILTER([关联商机].[商机阶段] = \"赢单\"))"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 客户信息表：最后跟进时间

取跟进记录中最近一次跟进时间。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<customers_table_id>" \
  --json '{"type":"formula","name":"最后跟进时间","expression":"MAX([跟进记录].[创建时间])"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

> **注意**：跟进记录表的 `创建时间` 是 `created_at` 系统字段，公式中通过双向关联字段 `[跟进记录]` 引用。确保跟进记录表已创建 `created_at` 系统字段（Step 1.4 中已添加）。

#### 商机管理表：跟进次数

通过关联字段跨表计数。

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"type":"formula","name":"跟进次数","expression":"COUNTA([跟进记录])"}' \
  --i-have-read-guide \
  --as user
```

> 如果公式中使用 `created_at` 系统字段，需要用 `+table-get` 获取实际系统字段名（可能叫"创建时间"），确保公式中的字段名与 `+field-list` 返回值一致。

### Step 1.7: 创建视图

#### 视图 1：商机阶段看板（Kanban）

```bash
lark-cli base +view-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"name":"商机阶段看板","type":"kanban"}' \
  --as user
sleep 2

# 设置看板分组：按商机阶段分组
lark-cli base +view-set-group --base-token "<base_token>" --table-id "<opportunities_table_id>" --view-id "<kanban_view_id>" \
  --json '[{"field":"商机阶段"}]' \
  --as user
```

#### 视图 2：跟进时间线

```bash
lark-cli base +view-create --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --json '{"name":"跟进时间线","type":"grid"}' \
  --as user
sleep 2

# 设置排序：按创建时间倒序
lark-cli base +view-set-sort --base-token "<base_token>" --table-id "<follow_ups_table_id>" --view-id "<timeline_view_id>" \
  --json '[{"field_name":"创建时间","desc":true}]' \
  --as user
```

#### 视图 3：本月赢单

```bash
lark-cli base +view-create --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"name":"本月赢单","type":"grid"}' \
  --as user
sleep 2

# 设置筛选：商机阶段 = 赢单
lark-cli base +view-set-filter --base-token "<base_token>" --table-id "<opportunities_table_id>" --view-id "<won_view_id>" \
  --json '{"conjunction":"and","conditions":[{"field_name":"商机阶段","operator":"is","value":["赢单"]}]}' \
  --as user
```

### Step 1.8: 创建仪表盘

> **IMPORTANT**：创建仪表盘 Block 前先读取 [`../lark-base/references/dashboard-block-data-config.md`](../lark-base/references/dashboard-block-data-config.md)

```bash
# 创建仪表盘
lark-cli base +dashboard-create --base-token "<base_token>" --name "销售看板" --as user
```

记录返回的 `dashboard_id`。

#### Block 1：销售漏斗

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "销售漏斗" --type funnel \
  --data-config '{"table_name":"商机管理","count_all":true,"group_by":[{"field_name":"商机阶段","mode":"integrated"}]}' \
  --as user
sleep 1.5
```

#### Block 2：本月赢单金额

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "本月赢单金额" --type statistics \
  --data-config '{"table_name":"商机管理","series":[{"field_name":"商机金额","rollup":"SUM"}],"filter":{"conjunction":"and","conditions":[{"field_name":"商机阶段","operator":"is","value":"赢单"}]}}' \
  --as user
sleep 1.5
```

#### Block 3：商机总金额（在途）

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "商机总金额" --type statistics \
  --data-config '{"table_name":"商机管理","series":[{"field_name":"商机金额","rollup":"SUM"}],"filter":{"conjunction":"and","conditions":[{"field_name":"商机阶段","operator":"isNot","value":"赢单"},{"field_name":"商机阶段","operator":"isNot","value":"丢单"}]}}' \
  --as user
sleep 1.5
```

#### Block 4：客户数量

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "客户总数" --type statistics \
  --data-config '{"table_name":"客户信息","count_all":true}' \
  --as user
sleep 1.5
```

#### Block 5：阶段分布

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "阶段分布" --type pie \
  --data-config '{"table_name":"商机管理","count_all":true,"group_by":[{"field_name":"商机阶段","mode":"integrated"}]}' \
  --as user
sleep 1.5
```

#### Block 6：负责人业绩排行

```bash
lark-cli base +dashboard-block-create --base-token "<base_token>" --dashboard-id "<dashboard_id>" \
  --name "负责人业绩排行" --type column \
  --data-config '{"table_name":"商机管理","series":[{"field_name":"商机金额","rollup":"SUM"}],"group_by":[{"field_name":"负责人","mode":"integrated"}],"filter":{"conjunction":"and","conditions":[{"field_name":"商机阶段","operator":"is","value":"赢单"}]}}' \
  --as user
sleep 1.5
```

### Step 1.9: 创建表单

在客户信息表上创建客户咨询登记表单，方便客户自助提交信息。

```bash
lark-cli base +form-create --base-token "<base_token>" --table-id "<customers_table_id>" \
  --name "客户咨询登记" \
  --description "欢迎填写咨询信息，我们会尽快与您联系" \
  --as user
```

表单创建后，可使用 `+form-questions-list` 查看已有问题，再用 `+form-questions-create` 添加或调整问题。

---

## Phase 2：客户管理

### 2.1 新增客户

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<customers_table_id>" \
  --json '{"客户名称":"北京星辰科技有限公司","客户类型":"企业客户","联系人":"王总","手机号":"138xxxx","邮箱":"wang@xx.com","行业":"互联网","公司规模":"51-200人","来源渠道":"展会","负责人":[{"id":"ou_xxx"}],"重要程度":"高","备注":""}' \
  --as user
```

**Agent 处理逻辑**：
1. 解析用户输入：提取客户名称、客户类型、联系人、手机、邮箱、行业、规模、来源、负责人、重要程度、备注
2. 未提供的字段留空
3. 行业/公司规模/来源渠道需匹配已有选项，不在选项中的需提醒用户
4. 人员字段需要 user_id，可通过 `contact +search-user` 查找

**新增客户回复格式**：
```
客户创建成功！

客户名称：北京星辰科技有限公司
客户类型：企业客户
联系人：王总
手机号：138xxxx
行业：互联网
负责人：当前用户
```

### 2.2 修改客户信息

```bash
# 先查询获取 record_id
lark-cli base +record-list --base-token "<base_token>" --table-id "<customers_table_id>" --as user

# 更新客户信息（Delta 更新，只更新用户指定的字段）
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<customers_table_id>" \
  --record-id "<record_id>" \
  --json '{"重要程度":"高","备注":"重点客户"}' \
  --as user
```

### 2.3 查询客户列表

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<customers_table_id>" \
  --limit 200 --as user
```

**Agent 处理**：
- 按用户指定的条件筛选（行业、负责人、重要程度等）
- 整理成易读表格格式

### 2.4 删除客户

```bash
lark-cli base +record-delete --base-token "<base_token>" --table-id "<customers_table_id>" \
  --record-id "<record_id>" --yes --as user
```

---

## Phase 3：商机管理

### 3.1 创建商机

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --json '{"商机名称":"企业IM系统采购","关联客户":[{"id":"<customer_record_id>"}],"商机金额":500000,"商机阶段":"待接触","负责人":[{"id":"ou_xxx"}],"预计成交日期":"2026-06-30","竞争对手":"","商机来源":"新开发"}' \
  --as user
```

> **关联字段值格式**：关联字段的值为 `[{id: "recXXXX"}]` 对象数组格式。
> **日期格式**：使用 `YYYY-MM-DD HH:mm:ss` 字符串格式。

**Agent 处理逻辑**：
1. 解析用户输入：提取商机名称、关联客户、金额、阶段、负责人、预计成交日期等
2. 如果用户提供了客户名称，先在客户表中查找对应的 record_id
3. 客户不存在时，询问是否要同时创建客户记录
4. 默认阶段为"待接触"，默认商机来源为"新开发"

**创建商机回复格式**：
```
商机创建成功！

商机名称：企业IM系统采购
关联客户：北京星辰科技
商机金额：¥500,000.00
商机阶段：待接触
负责人：当前用户
预计成交日期：2026-06-30
```

### 3.2 推进商机阶段

当用户在跟进记录中提及阶段变更，或直接要求推进商机时：

```bash
# 更新商机阶段
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --record-id "<opportunity_record_id>" \
  --json '{"商机阶段":"方案报价"}' \
  --as user
```

**Agent 处理逻辑**：
1. 用户说"推进到方案报价"→ 更新商机阶段为"方案报价"
2. 用户说"赢单了"→ 更新商机阶段为"赢单"
3. 用户说"丢单了"→ 更新商机阶段为"丢单"，并询问丢单原因
4. 阶段变更后自动更新赢单概率建议（formula 字段自动计算）

**阶段顺序**：待接触 → 需求确认 → 方案报价 → 商务谈判 → 赢单/丢单

### 3.3 查询商机列表

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --limit 200 --as user
```

**Agent 处理**：
- 按阶段、负责人、客户等条件筛选
- 按商机金额排序

### 3.4 销售漏斗查询

```bash
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<opportunities_table_id>"}},
    "dimensions": [{"field_name":"商机阶段","alias":"stage"}],
    "measures": [
      {"field_name":"商机金额","aggregation":"sum","alias":"total_amount"},
      {"field_name":"商机名称","aggregation":"count","alias":"count"}
    ]
  }' \
  --as user
```

**回复格式**：
```
销售漏斗

| 阶段 | 数量 | 金额 |
|------|------|------|
| 待接触 | 12 | ¥1,200,000 |
| 需求确认 | 8 | ¥800,000 |
| 方案报价 | 5 | ¥500,000 |
| 商务谈判 | 3 | ¥450,000 |
| 赢单 | 2 | ¥200,000 |
| 丢单 | 1 | ¥100,000 |
```

---

## Phase 4：跟进记录

### 4.1 添加跟进记录

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --json '{"关联客户":[{"id":"<customer_record_id>"}],"关联商机":[{"id":"<opportunity_record_id>"}],"跟进方式":"电话","跟进摘要":"电话联系了王总，对方对方案比较满意","客户反馈":"认可方案，但价格偏高","下一步计划":"下周三面访确认细节","下次跟进时间":"2026-04-16 14:00","跟进人":[{"id":"ou_xxx"}],"商机阶段变更":"无变更"}' \
  --as user
```

**Agent 处理逻辑**：
1. 解析用户输入：提取跟进方式、摘要、客户反馈、下一步计划、下次跟进时间、阶段变更
2. 查找关联客户和商机的 record_id
3. **关键**：如果用户提到阶段变更（如"对方同意推进到方案报价"），同步更新对应商机的商机阶段
4. 日期默认当天

**跟进记录回复格式**：
```
跟进记录已保存！

客户：北京星辰科技
商机：企业IM系统采购
跟进方式：电话
跟进摘要：电话联系了王总，对方对方案比较满意
下一步计划：下周三面访确认细节
下次跟进时间：2026-04-16 14:00
```

### 4.2 查询跟进记录

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --limit 200 --as user
```

### 4.3 销售日报

```
Agent 处理逻辑：
1. 查询今天所有跟进记录（按创建时间筛选当天）
2. 查询今天新增的客户（按创建时间筛选当天）
3. 查询今天新增的商机
4. 查询今天发生阶段变更的商机
5. 生成结构化日报
```

**日报格式**：
```
销售日报 — 2026-04-13

一、今日跟进（3 条）
| 客户 | 商机 | 方式 | 摘要 | 阶段变更 |
|------|------|------|------|---------|
| 北京星辰 | IM采购 | 电话 | 方案满意 | → 方案报价 |
| 上海悦达 | 数据服务 | 微信 | 初步接触 | 无 |

二、新增客户（1 家）
- 深圳华创科技，互联网行业，来自线上广告

三、新增商机（2 个）
- 深圳华创科技：云服务采购 ¥300,000（待接触）
- 北京星辰科技：IM系统采购 ¥500,000（待接触）

四、阶段变更（1 个）
- 北京星辰 IM采购：待接触 → 方案报价

五、今日提醒
- 1 个商机超 7 天未跟进
- 2 个客户待跟进
```

---

## Phase 5：数据分析

### 5.1 业绩排行

```bash
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<opportunities_table_id>"}},
    "dimensions": [{"field_name":"负责人","alias":"owner"}],
    "measures": [
      {"field_name":"商机金额","aggregation":"sum","alias":"total_amount"},
      {"field_name":"商机名称","aggregation":"count","alias":"count"}
    ],
    "filters": {"type": 1, "conjunction": "and", "conditions": [{"field_name":"商机阶段","operator":"is","value": ["赢单"]}]}
  }' \
  --as user
```

### 5.2 本月赢单统计

```bash
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<opportunities_table_id>"}},
    "measures": [
      {"field_name":"商机金额","aggregation":"sum","alias":"total_amount"},
      {"field_name":"商机名称","aggregation":"count","alias":"count"}
    ],
    "filters": {"type": 1, "conjunction": "and", "conditions": [{"field_name":"商机阶段","operator":"is","value": ["赢单"]}]}
  }' \
  --as user
```

### 5.3 商机转化率

```bash
# 先获取各阶段数量
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<opportunities_table_id>"}},
    "dimensions": [{"field_name":"商机阶段","alias":"stage"}],
    "measures": [{"field_name":"商机名称","aggregation":"count","alias":"count"}]
  }' \
  --as user
```

Agent 计算各阶段转化率：
```
商机转化分析

| 阶段 | 数量 | 转化率 |
|------|------|--------|
| 待接触 | 12 | 100%（入口） |
| 需求确认 | 8 | 67% |
| 方案报价 | 5 | 42% |
| 商务谈判 | 3 | 25% |
| 赢单 | 2 | 17% |

整体赢单率：17%（2/12）
```

---

## Phase 6：自动化提醒

### 6.1 每日跟进提醒

查询今天需要跟进的内容：

```bash
# 查询下次跟进时间 = 今天的跟进记录
lark-cli base +record-list --base-token "<base_token>" --table-id "<follow_ups_table_id>" \
  --limit 200 --as user
# AI 筛选：下次跟进时间 = 今天

# 查询预计成交日期 = 今天且未关单的商机
lark-cli base +record-list --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --limit 200 --as user
# AI 筛选：预计成交日期 = 今天 AND 商机阶段 NOT IN (赢单, 丢单)
```

发送飞书消息：

```bash
lark-cli im +messages-send --chat-id "<chat_id>" \
  --msg-type "interactive" \
  --content '{
    "config": {"wide_screen_mode": true},
    "header": {"title": {"tag": "plain_text", "content": "今日跟进提醒"}, "template": "blue"},
    "elements": [
      {"tag": "div", "text": {"tag": "lark_md", "content": "**跟进提醒**\n- 客户A：下次跟进时间到，计划：XXX\n\n**商机提醒**\n- 商机B：预计今天成交，金额：XXX"}}
    ]
  }' \
  --as user
```

> **重要**：发送消息前需先读取 [`../lark-im/SKILL.md`](../lark-im/SKILL.md) 了解消息发送规则。

### 6.2 超时未跟进预警

```bash
# 查询超过 7 天未跟进的商机
lark-cli base +record-list --base-token "<base_token>" --table-id "<opportunities_table_id>" \
  --limit 200 --as user
# AI 筛选：商机阶段 NOT IN (赢单, 丢单) AND 跟进次数 > 0 AND 最后跟进时间 > 7 天前
```

**消息格式**：
```
超时未跟进预警

以下商机已超过 7 天未跟进：

| 商机 | 客户 | 阶段 | 未跟进天数 |
|------|------|------|-----------|
| 商机A | 客户A | 方案报价 | 8 天 |
| 商机B | 客户B | 需求确认 | 12 天 |
```

### 6.3 赢单报喜

当商机阶段变更为「赢单」时，发送飞书群消息通知：

```bash
lark-cli im +messages-send --chat-id "<chat_id>" \
  --msg-type "interactive" \
  --content '{
    "config": {"wide_screen_mode": true},
    "header": {"title": {"tag": "plain_text", "content": "赢单喜报！"}, "template": "green"},
    "elements": [
      {"tag": "div", "text": {"tag": "lark_md", "content": "商机：XX公司 ERP 采购\n金额：¥100,000\n负责人：@用户\n商机周期：XX天"}}
    ]
  }' \
  --as user
```

### 6.4 每周销售简报

```
Agent 处理逻辑：
1. 汇总上周新增客户数（按创建时间筛选）
2. 汇总上周新增商机数和金额
3. 汇总上周赢单数和金额
4. 汇总当前漏斗各阶段数量和金额
5. 生成结构化简报并发送
```

**简报格式**：
```
周销售简报（上周 2026-04-07 ~ 2026-04-13）

一、关键指标
- 新增客户：5 家
- 新增商机：8 个，总金额 ¥500,000
- 赢单：3 个，金额 ¥200,000
- 丢单：1 个，金额 ¥50,000

二、当前漏斗
| 阶段 | 数量 | 金额 |
|------|------|------|
| 待接触 | 12 | ¥1,200,000 |
| 需求确认 | 8 | ¥800,000 |
| 方案报价 | 5 | ¥500,000 |
| 商务谈判 | 3 | ¥450,000 |

三、需关注
- 3 个商机超 7 天未跟进
- 2 个商机预计本周成交
```

---

## Phase 7：表单管理

### 7.1 创建客户咨询登记表单

```bash
lark-cli base +form-create --base-token "<base_token>" --table-id "<customers_table_id>" \
  --name "客户咨询登记" \
  --description "欢迎填写咨询信息，我们会尽快与您联系" \
  --as user
```

### 7.2 查看和配置表单问题

```bash
# 查看已有问题
lark-cli base +form-questions-list --base-token "<base_token>" --table-id "<customers_table_id>" --form-id "<form_id>" --as user

# 添加/修改问题
lark-cli base +form-questions-create --base-token "<base_token>" --table-id "<customers_table_id>" --form-id "<form_id>" \
  --json '[{"name":"客户名称","type":"input"},{"name":"联系电话","type":"input"},{"name":"咨询内容","type":"textarea"}]' \
  --as user
```

---

## 意图 → 命令索引

| 意图 | 推荐命令 | 备注 |
|------|---------|------|
| 创建CRM系统 | `+base-create` → 3 × `+table-create` → 逐个 `+field-create` → 双向关联 → formula → 视图 → 仪表盘 → 表单 | 按 Step 1 顺序执行 |
| 新增客户 | `+record-upsert` → customers 表 | |
| 修改客户 | `+record-list` → `+record-upsert` → customers 表 | 先查 record_id |
| 删除客户 | `+record-delete` → customers 表 | 带 `--yes` |
| 查询客户 | `+record-list` → customers 表 | AI 过滤 |
| 创建商机 | `+record-list`（查客户）→ `+record-upsert` → opportunities 表 | 关联客户 |
| 推进商机 | `+record-upsert` → opportunities 表 | 更新商机阶段 |
| 查看商机 | `+record-list` → opportunities 表 | |
| 销售漏斗 | `+data-query` → opportunities 表 | 按阶段聚合 |
| 添加跟进 | `+record-upsert` → follow_ups 表 | 如有阶段变更同步更新商机 |
| 销售日报 | `+record-list` → follow_ups + customers + opportunities 表 | AI 汇总 |
| 业绩排行 | `+data-query` → opportunities 表 | 赢单金额按负责人聚合 |
| 跟进提醒 | `+record-list` → follow_ups + opportunities → `im +messages-send` | |
| 超时预警 | `+record-list` → opportunities → `im +messages-send` | |
| 赢单报喜 | `im +messages-send` | |
| 每周简报 | `+record-list` + `+data-query` → `im +messages-send` | |
| 创建表单 | `+form-create` → `+form-questions-create` | |
| 创建看板 | `+dashboard-create` → `+dashboard-block-create` | 先读 dashboard guide |

---

## 常见错误速查

| 错误场景 | 原因 | 解决方案 |
|---------|------|---------|
| `+table-create --fields` 限流 800004135 | 一次性创建多字段触发限流 | 先建空表，再用 `+field-create` 逐个加字段，间隔 1.5 秒 |
| 公式字段跨表查询返回全表结果 | 关联字段非双向 | 关联字段必须 `bidirectional: true` |
| 单向关联不能改为双向 | API 限制 unsafe_operation_blocked | 删除旧关联字段，重新创建双向关联 |
| `+data-query` unknown flag `--table-id` | 该命令不支持 `--table-id` | 把 table-id 放在 `--dsl` 的 `datasource.table.tableId` 中 |
| `+data-query` 不支持某字段聚合 | 公式/lookup/关联字段不在白名单 | 只对存储字段做聚合 |
| `+data-query` 聚合 `counta` 不支持 | `+data-query` 仅支持 `count` 不支持 `counta` | 改用 `"aggregation":"count"` |
| 关联字段值写入失败 | 格式错误 | 关联字段值必须是 `[{id: "recXXX"}]` 对象数组 |
| 日期字段写入失败 | 格式错误 | 使用 `YYYY-MM-DD HH:mm:ss` 字符串格式 |
| 人员字段写入失败 | 格式错误 | 使用 `[{id: "ou_xxx"}]` 对象数组 |
| 公式字段创建失败 | 未读 guide 或表达式有误 | 先读 formula-field-guide.md |
| 表名/字段名不匹配 | 凭自然语言猜测 | 用 `+table-list` / `+field-list` 获取真实名称 |
| 并发报错 1254291 | list 命令并发 | 所有 `+xxx-list` 串行执行 |
| 仪表盘 Block 创建失败 | 使用 `--json` 而非独立参数 | 改用 `--name` + `--type` + `--data-config` 独立参数 |
| `+view-create` 不支持 `--name` | 需用 `--json` 传参 | 改用 `--json '{"name":"xxx","type":"grid"}'` |
| `+view-set-group` 字段名 key 错误 | JSON key 是 `field` 不是 `field_name` | 改用 `[{"field":"字段名"}]` |
| 公式引用「创建时间」返回 null | 表没有 `created_at` 系统字段 | 初始化时用 `+field-create` 手动添加 `type:"created_at"` 字段 |
| Base 自带默认"数据表" | 创建 Base 时自动生成 | 忽略该表，只操作客户信息/商机管理/跟进记录三张表 |
| 消息发送失败 | chat_id 无效 | 用 `im +chat-list` 确认 chat_id |
| 权限不足 | scope 未开通 | 参考 lark-shared 处理权限问题 |

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +base-create` | `bitable:app` |
| `base +table-create` | `bitable:app` |
| `base +field-create` | `bitable:app` |
| `base +field-list` | `bitable:app:readonly` |
| `base +record-list` | `bitable:app:readonly` |
| `base +record-upsert` | `bitable:app` |
| `base +record-delete` | `bitable:app` |
| `base +data-query` | `bitable:app:readonly` |
| `base +view-create` | `bitable:app` |
| `base +view-set-filter` | `bitable:app` |
| `base +view-set-sort` | `bitable:app` |
| `base +dashboard-create` | `bitable:app` |
| `base +dashboard-block-create` | `bitable:app` |
| `base +form-create` | `bitable:app` |
| `base +form-questions-list` | `bitable:app:readonly` |
| `base +form-questions-create` | `bitable:app` |
| `im +messages-send` | `im:message:send_as_bot` |
| `contact +search-user` | `contact:employee.id:readonly` |

---

## 参考

- [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`../lark-base/SKILL.md`](../lark-base/SKILL.md) — 多维表格原子操作
- [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md) — 公式字段写法（创建公式字段前必读）
- [`../lark-base/references/lookup-field-guide.md`](../lark-base/references/lookup-field-guide.md) — lookup 字段配置规则
- [`../lark-base/references/lark-base-shortcut-field-properties.md`](../lark-base/references/lark-base-shortcut-field-properties.md) — 字段属性 JSON 规范
- [`../lark-base/references/lark-base-shortcut-record-value.md`](../lark-base/references/lark-base-shortcut-record-value.md) — 记录值格式规范
- [`../lark-base/references/lark-base-data-query.md`](../lark-base/references/lark-base-data-query.md) — 聚合分析 DSL
- [`../lark-base/references/dashboard-block-data-config.md`](../lark-base/references/dashboard-block-data-config.md) — 仪表盘 Block 配置
- [`../lark-base/references/lark-base-form-create.md`](../lark-base/references/lark-base-form-create.md) — 表单创建
- [`../lark-base/references/lark-base-view-set-filter.md`](../lark-base/references/lark-base-view-set-filter.md) — 视图筛选配置
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 飞书消息发送
- [`../lark-contact/SKILL.md`](../lark-contact/SKILL.md) — 通讯录人员查询
