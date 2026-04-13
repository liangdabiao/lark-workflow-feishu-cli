---
name: lark-workflow-erp-inventory
version: 1.1.0
description: "进销存ERP系统：基于飞书多维表格实现货品管理、入库、出库、库存预警、数据分析。当用户需要以下操作时触发：系统搭建（'创建进销存系统'、'搭建库存管理'、'新建进销存'）；货品管理（'新增货品'、'添加SKU'、'货品列表'、'修改货品'）；入库操作（'入库'、'进货'、'采购入库'、'登记入库'）；出库操作（'出库'、'领用'、'发货出库'、'登记出库'）；库存查询（'查库存'、'库存清单'、'剩余多少'、'库存不足'）；库存预警（'低库存预警'、'库存报警'、'缺货提醒'）；数据分析（'入库汇总'、'出库统计'、'库存报表'、'滞销分析'、'库存周转'）；供应商管理（'新增供应商'、'供应商列表'、'供货商'）；仪表盘（'库存看板'、'创建仪表盘'）。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 进销存 ERP 系统工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我创建一个进销存系统" / "搭建库存管理" / "新建进销存"
- "新增货品：米色T恤，规格S/M/L" / "添加SKU"
- "入库：米色T恤 10件，单价50" / "进货登记"
- "出库：米色T恤 3件，用途领用" / "发货出库"
- "查一下米色T恤的库存" / "库存清单" / "哪些货品库存不足"
- "低库存预警" / "库存报警" / "缺货提醒"
- "本月入库汇总" / "出库统计" / "库存报表" / "滞销分析"
- "新增供应商" / "供应商列表"
- "创建库存看板" / "库存仪表盘"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base,im,docs
```

## 系统架构

```
进销存 ERP (lark-workflow-erp-inventory)
│
├── 数据层：飞书多维表格 (Bitable)
│   ├── 货品库存台账 (products)     — SKU 主数据
│   ├── 入库记录表 (stock_in)       — 入库流水
│   ├── 出库记录表 (stock_out)      — 出库流水
│   └── 供应商表 (suppliers)        — 供应商信息（扩展）
│
├── 计算层
│   ├── 累计入库 (lookup)     — 从入库表按物品名称汇总入库数量
│   ├── 累计出库 (lookup)     — 从出库表按物品名称汇总出库数量
│   ├── 剩余库存 (formula)    — 累计入库 - 累计出库
│   ├── 库存状态 (formula)    — IF(剩余库存 ≤ 最低库存, "低库存", "正常")
│   └── 入库金额 (formula)    — 入库数量 × 入库单价
│
├── 关联关系
│   ├── 入库表.关联物品 ↔ 台账（双向）
│   ├── 出库表.关联物品 ↔ 台账（双向）
│   └── 台账.供应商 ↔ 供应商表（双向）
│
├── 交互层：用户自然语言指令
│   ├── 系统初始化 → 创建 Base + 表 + 字段 + 关联
│   ├── 日常操作 → 入库/出库/查询
│   └── 数据分析 → 聚合统计 + 报表
│
└── 通知层：飞书消息
    ├── 低库存预警通知
    └── 出库确认通知
```

---

## Agent 核心规则

1. **出库前必须校验库存** — 先查询台账表的剩余库存，不足则拦截并提示用户
2. **只使用 `lark-cli base +...` 原子命令** — 不走 `lark-cli api` 直接调用
3. **建表必须先建空表再逐个加字段** — `+table-create` 带 `--fields` 多字段时容易触发限流（错误码 800004135），改为先建空表，再用 `+field-create` 逐个加字段，每次间隔 1.5 秒
4. **关联字段必须使用双向关联** — `bidirectional: true`，否则无法在公式中做跨表聚合
5. **库存计算使用 lookup + formula 组合** — 不要用公式字段直接跨表查询（会变成全表 SUM），改用 lookup 字段做跨表聚合，再用公式字段做差值计算
6. **`+data-query` 的 table-id 放在 DSL 中** — 不支持 `--table-id` 参数，需放在 `--dsl` 的 `datasource.table.tableId` 中
7. **`+data-query` 只支持存储字段** — 不支持公式字段、lookup 字段、关联字段做聚合
8. **写记录前先拿字段结构** — `+field-list` 确认字段名和类型
9. **公式字段创建前先读 guide** — 读取 [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md)
10. **`+xxx-list` 禁止并发** — 所有 list 命令串行执行
11. **写入串行 + 间隔** — `+record-upsert` 串行，间隔 0.5-1 秒
12. **分页拉取** — `+record-list --limit 200`，用 offset 分页
13. **表名/字段名精确匹配** — 禁止凭自然语言猜测，必须来自 `+table-list` / `+field-list` 返回值

---

## Phase 1：系统初始化

### Step 1.1: 创建 Base

```bash
lark-cli base +base-create --name "进销存管理系统" --as user
```

记录返回的 `base_token`（后续所有命令需要）。

### Step 1.2: 创建数据表（先建空表，间隔 2 秒）

> **IMPORTANT**：不要使用 `--fields` 参数一次性创建多字段，容易触发限流。先创建空表，再用 `+field-create` 逐个添加字段。

```bash
# 表 1：货品库存台账
lark-cli base +table-create --base-token "<base_token>" --name "货品库存台账" --as user
sleep 2

# 表 2：供应商
lark-cli base +table-create --base-token "<base_token>" --name "供应商" --as user
sleep 2

# 表 3：入库记录
lark-cli base +table-create --base-token "<base_token>" --name "入库记录" --as user
sleep 2

# 表 4：出库记录
lark-cli base +table-create --base-token "<base_token>" --name "出库记录" --as user
```

### Step 1.3: 获取各表 table_id

```bash
lark-cli base +table-list --base-token "<base_token>" --as user
```

记录各表的 `table_id`。

### Step 1.4: 逐个添加存储字段（每创建一个间隔 1.5 秒）

#### 台账表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"text","name":"物品名称"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"text","name":"规格型号"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"select","name":"类别","options":[{"name":"服装"},{"name":"电子产品"},{"name":"食品"},{"name":"办公用品"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"select","name":"单位","options":[{"name":"件"},{"name":"个"},{"name":"箱"},{"name":"套"},{"name":"kg"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"number","name":"最低库存","style":{"type":"plain","precision":0}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"number","name":"单价","style":{"type":"currency","precision":2,"currency_code":"CNY"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" --json '{"type":"text","name":"备注"}' --as user
sleep 2
```

#### 供应商表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"text","name":"供应商名称"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"text","name":"联系人"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"text","name":"联系电话","style":{"type":"phone"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"text","name":"地址"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"select","name":"供应类别","options":[{"name":"服装"},{"name":"电子产品"},{"name":"食品"},{"name":"办公用品"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"select","name":"合作状态","options":[{"name":"合作中"},{"name":"暂停"},{"name":"终止"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<suppliers_table_id>" --json '{"type":"text","name":"备注"}' --as user
sleep 2
```

#### 入库记录表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"text","name":"供应商"}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"number","name":"入库数量","style":{"type":"plain","precision":0}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"number","name":"入库单价","style":{"type":"currency","precision":2,"currency_code":"CNY"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"select","name":"用途","options":[{"name":"采购"},{"name":"退货"},{"name":"调拨"},{"name":"盘盈"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"user","name":"入库人","multiple":false}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"datetime","name":"入库日期","style":{"format":"yyyy-MM-dd"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" --json '{"type":"text","name":"备注"}' --as user
sleep 2
```

#### 出库记录表字段

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" --json '{"type":"number","name":"出库数量","style":{"type":"plain","precision":0}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" --json '{"type":"user","name":"领用人","multiple":false}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" --json '{"type":"datetime","name":"出库日期","style":{"format":"yyyy-MM-dd"}}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" --json '{"type":"select","name":"用途","options":[{"name":"销售"},{"name":"领用"},{"name":"调拨"},{"name":"盘亏"},{"name":"其他"}]}' --as user
sleep 1.5
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" --json '{"type":"text","name":"备注"}' --as user
sleep 2
```

### Step 1.5: 创建双向关联字段（每创建一个间隔 2 秒）

> **CRITICAL**：关联字段必须使用 `bidirectional: true`，否则无法在公式/lookup 中做跨表聚合。

#### 台账 → 供应商 关联

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"type":"link","name":"供应商","link_table":"供应商","bidirectional":true,"bidirectional_link_field_name":"供应货品"}' \
  --as user
sleep 2
```

#### 入库表 → 台账 关联（双向）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" \
  --json '{"type":"link","name":"关联物品","link_table":"货品库存台账","bidirectional":true,"bidirectional_link_field_name":"入库记录"}' \
  --as user
sleep 2
```

#### 出库表 → 台账 关联（双向）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_out_table_id>" \
  --json '{"type":"link","name":"关联物品","link_table":"货品库存台账","bidirectional":true,"bidirectional_link_field_name":"出库记录"}' \
  --as user
sleep 2
```

### Step 1.6: 创建 lookup + formula 计算字段

> **IMPORTANT**：
> - 创建公式字段前 MUST 先读取 [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md)
> - 创建 lookup 字段前 MUST 先读取 [`../lark-base/references/lookup-field-guide.md`](../lark-base/references/lookup-field-guide.md)
> - 所有计算字段创建命令都需追加 `--i-have-read-guide`

**库存计算架构**：lookup 字段做跨表聚合 → formula 字段做差值计算

1. **累计入库**（lookup 字段）：从入库表汇总当前物品名称对应的入库数量
2. **累计出库**（lookup 字段）：从出库表汇总当前物品名称对应的出库数量
3. **剩余库存**（formula 字段）：累计入库 - 累计出库
4. **库存状态**（formula 字段）：IF(剩余库存 ≤ 最低库存, "低库存", "正常")
5. **入库金额**（formula 字段）：入库数量 × 入库单价

#### 台账表：累计入库（lookup）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"type":"lookup","name":"累计入库","from":"入库记录","select":"入库数量","where":{"logic":"and","conditions":[["关联物品","==",{"type":"field_ref","field":"物品名称"}]]},"aggregate":"SUM"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

> **lookup 说明**：`from` 指定源表名，`select` 指定要聚合的字段，`where` 中通过 `field_ref` 引用当前表的物品名称字段与源表关联字段匹配，`aggregate: "SUM"` 求和。

#### 台账表：累计出库（lookup）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"type":"lookup","name":"累计出库","from":"出库记录","select":"出库数量","where":{"logic":"and","conditions":[["关联物品","==",{"type":"field_ref","field":"物品名称"}]]},"aggregate":"SUM"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 台账表：剩余库存（formula）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"type":"formula","name":"剩余库存","expression":"IFBLANK([累计入库], 0) - IFBLANK([累计出库], 0)"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 台账表：库存状态（formula）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"type":"formula","name":"库存状态","expression":"IF([剩余库存] <= [最低库存], \"低库存\", \"正常\")"}' \
  --i-have-read-guide \
  --as user
sleep 2
```

#### 入库表：入库金额（formula）

```bash
lark-cli base +field-create --base-token "<base_token>" --table-id "<stock_in_table_id>" \
  --json '{"type":"formula","name":"入库金额","expression":"[入库数量] * [入库单价]"}' \
  --i-have-read-guide \
  --as user
```

---

## Phase 2：日常操作

### 2.1 新增货品

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<products_table_id>" \
  --json '{"物品名称":"米色T恤","规格型号":"S/M/L/XL","类别":"服装","单位":"件","最低库存":5,"单价":50,"备注":"夏季新款"}' \
  --as user
```

**Agent 处理逻辑**：
1. 解析用户输入：提取物品名称、规格、类别、单位、最低库存、单价、备注
2. 未提供的字段留空或使用默认值
3. 类别/单位需匹配已有选项，不在选项中的需提醒用户

### 2.2 修改货品信息

```bash
# 先查询获取 record_id
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" --as user

# 更新货品信息
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<products_table_id>" \
  --record-id "<record_id>" \
  --json '{"最低库存":10,"备注":"更新备注"}' \
  --as user
```

**Agent 处理逻辑**：
1. 根据用户提供的物品名称/编号查询 record_id
2. 只更新用户指定的字段（Delta 更新）

### 2.3 入库操作

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<stock_in_table_id>" \
  --json '{"关联物品":["<record_id>"],"入库数量":10,"入库单价":50,"用途":"采购","入库日期":<timestamp_ms>,"备注":"正常采购"}' \
  --as user
```

> **关联字段值格式**：关联字段的值为 record_id 数组，如 `["recXXXX"]`。
> **日期格式**：日期字段使用 Unix 毫秒时间戳（数字），不是字符串。

**Agent 处理逻辑**：
1. 解析用户输入：提取物品名称、入库数量、单价、用途/来源
2. 在台账表中查询物品对应的 record_id
3. 日期默认当天（`Date.now()` 毫秒时间戳）
4. 写入入库记录
5. 回复用户：入库成功，并显示更新后的剩余库存

**入库成功回复格式**：
```
入库成功！

物品：米色T恤
入库数量：10 件
入库单价：50.00 元
入库金额：500.00 元
用途：采购
日期：2024-01-15

当前剩余库存：10 件
```

### 2.4 出库操作（含库存校验）

**出库前必须执行库存校验**：

```bash
# Step 1: 查询物品当前库存
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" --as user
# 从返回结果中找到目标物品的 record_id 和剩余库存值

# Step 2: 校验库存
# 如果 剩余库存 < 出库数量 → 拦截并提示
# 如果 剩余库存 >= 出库数量 → 执行出库
```

```bash
# Step 3: 执行出库
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<stock_out_table_id>" \
  --json '{"关联物品":["<record_id>"],"出库数量":3,"用途":"领用","出库日期":<timestamp_ms>,"备注":""}' \
  --as user
```

**Agent 处理逻辑**：
1. 解析用户输入：提取物品名称、出库数量、用途、领用人
2. 查询台账表获取物品 record_id 和剩余库存
3. **校验**：剩余库存 >= 出库数量？
   - 不足 → 提示 `"库存不足！米色T恤 当前剩余库存仅 X 件，无法出库 Y 件。"` 并中止
   - 充足 → 继续出库
4. 写入出库记录
5. 回复用户

**库存不足拦截格式**：
```
出库失败 - 库存不足！

物品：米色T恤
当前剩余库存：2 件
请求出库数量：3 件
缺少：1 件
```

**出库成功回复格式**：
```
出库成功！

物品：米色T恤
出库数量：3 件
用途：领用
日期：2024-01-15

出库前库存：10 件
出库后库存：7 件
```

### 2.5 查询库存

#### 查询单个物品库存

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" --as user
# AI 从返回结果中筛选目标物品
```

**回复格式**：
```
物品库存查询

物品名称：米色T恤
规格型号：S/M/L/XL
类别：服装
累计入库：10 件
累计出库：3 件
剩余库存：7 件
最低库存：5 件
库存状态：正常
单价：50.00 元
```

#### 查询全部库存清单

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" \
  --limit 200 --as user
# 如超过 200 条，用 offset 分页
```

**回复格式**：
```
库存清单（共 N 种货品）

| 物品名称 | 规格 | 剩余库存 | 最低库存 | 状态 | 单价 |
|---------|------|---------|---------|------|------|
| 米色T恤 | S/M/L | 7 件 | 5 件 | 正常 | 50.00 |
| 黑色外套 | M/L | 2 件 | 10 件 | ⚠️低库存 | 200.00 |

⚠️ 低库存预警：1 种货品低于最低库存
```

### 2.6 查询入库/出库记录

```bash
# 入库记录
lark-cli base +record-list --base-token "<base_token>" --table-id "<stock_in_table_id>" \
  --limit 200 --as user

# 出库记录
lark-cli base +record-list --base-token "<base_token>" --table-id "<stock_out_table_id>" \
  --limit 200 --as user
```

**Agent 处理**：
- 如果用户指定了物品名称，从返回结果中筛选对应记录
- 如果用户指定了时间范围，按日期筛选
- 整理成易读的表格格式回复

### 2.7 删除货品/记录

```bash
lark-cli base +record-delete --base-token "<base_token>" --table-id "<products_table_id>" \
  --record-id "<record_id>" --yes --as user
```

> **注意**：删除货品前应提醒用户，该货品相关的入库/出库记录中的关联会受影响。

---

## Phase 3：库存预警与通知

### 3.1 低库存检测

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" \
  --limit 200 --as user
# AI 筛选 库存状态 = "低库存" 的记录
```

**回复格式**：
```
低库存预警报告

以下 N 种货品库存低于最低库存：

| 物品名称 | 当前库存 | 最低库存 | 缺少 | 单价 | 库存金额 |
|---------|---------|---------|------|------|---------|
| 黑色外套 | 2 件 | 10 件 | 8 件 | 200.00 | 400.00 |
| 蓝色牛仔裤 | 0 件 | 10 件 | 10 件 | 150.00 | 0.00 |

建议及时补货。
```

### 3.2 发送预警通知到飞书群

```bash
lark-cli im +messages-send --chat-id "<chat_id>" \
  --msg-type "interactive" \
  --content '{
    "config": {"wide_screen_mode": true},
    "header": {"title": {"tag": "plain_text", "content": "低库存预警"}, "template": "red"},
    "elements": [
      {"tag": "div", "text": {"tag": "lark_md", "content": "以下货品库存低于最低库存，请及时补货："}},
      {"tag": "div", "text": {"tag": "lark_md", "content": "**黑色外套** 当前 2 件 / 最低 10 件\n**蓝色牛仔裤** 当前 0 件 / 最低 10 件"}}
    ]
  }' \
  --as user
```

> **重要**：发送消息前需先读取 [`../lark-im/SKILL.md`](../lark-im/SKILL.md) 了解消息发送规则。

### 3.3 出库确认通知

出库完成后，如果用户要求发送通知：

```bash
lark-cli im +messages-send --chat-id "<chat_id>" \
  --msg-type "interactive" \
  --content '{
    "config": {"wide_screen_mode": true},
    "header": {"title": {"tag": "plain_text", "content": "出库确认通知"}, "template": "blue"},
    "elements": [
      {"tag": "div", "text": {"tag": "lark_md", "content": "已成功出库："}},
      {"tag": "div", "text": {"tag": "lark_md", "content": "**物品**：米色T恤\n**数量**：3 件\n**用途**：领用\n**领用人**：@用户"}}
    ]
  }' \
  --as user
```

---

## Phase 4：数据分析

### 4.1 入库汇总

> **IMPORTANT**：`+data-query` 不支持 `--table-id` 参数，table-id 必须放在 `--dsl` 的 `datasource` 中。且只支持存储字段（不支持公式字段、lookup 字段、关联字段）。

```bash
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<stock_in_table_id>"}},
    "dimensions": [{"field_name":"用途","alias":"usage_type"}],
    "measures": [
      {"field_name":"入库数量","aggregation":"sum","alias":"total_qty"},
      {"field_name":"入库单价","aggregation":"avg","alias":"avg_price"}
    ]
  }' \
  --as user
```

> **注意**：不要对公式字段（如"入库金额"）做聚合，会报错。改为对"入库数量"做 SUM，对"入库单价"做 AVG。

**回复格式**：
```
入库汇总

| 用途 | 入库数量 | 平均单价 |
|------|---------|---------|
| 采购 | 33 件 | 133.33 元 |
| **合计** | **33 件** | |
```

### 4.2 出库汇总

```bash
lark-cli base +data-query --base-token "<base_token>" \
  --dsl '{
    "datasource": {"type": "table", "table": {"tableId": "<stock_out_table_id>"}},
    "dimensions": [{"field_name":"用途","alias":"usage_type"}],
    "measures": [{"field_name":"出库数量","aggregation":"sum","alias":"total_qty"}]
  }' \
  --as user
```

### 4.3 滞销分析

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<products_table_id>" \
  --limit 200 --as user
```

**Agent 处理逻辑**：
1. 拉取台账数据，获取各物品的累计入库和剩余库存（lookup 字段值）
2. 计算出库量 = 累计入库 - 剩余库存
3. 筛选出库量为 0 或低于阈值的货品
4. 按库存金额排序

### 4.4 生成库存报表文档

```bash
lark-cli docs +create --title "库存报表 - 2024年1月" --markdown "<报表内容>" --as user
```

### 4.5 创建库存看板（仪表盘）

```bash
lark-cli base +dashboard-create --base-token "<base_token>" --name "库存看板" --as user
```

> **注意**：仪表盘配置需先读取 [`../lark-base/references/dashboard-block-data-config.md`](../lark-base/references/dashboard-block-data-config.md) 了解 data_config 结构。

---

## Phase 5：供应商管理

### 5.1 新增供应商

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<suppliers_table_id>" \
  --json '{"供应商名称":"深圳XX纺织厂","联系人":"张经理","联系电话":"13800138000","地址":"深圳市南山区XX路XX号","供应类别":"服装","合作状态":"合作中","备注":"主力供应商"}' \
  --as user
```

### 5.2 查询供应商列表

```bash
lark-cli base +record-list --base-token "<base_token>" --table-id "<suppliers_table_id>" \
  --limit 200 --as user
```

### 5.3 关联供应商到货品

```bash
lark-cli base +record-upsert --base-token "<base_token>" --table-id "<products_table_id>" \
  --record-id "<products_record_id>" \
  --json '{"供应商":["<suppliers_record_id>"]}' \
  --as user
```

> **关联字段值格式**：值为关联记录的 record_id 数组 `["recXXXX"]`。

---

## 意图 → 命令索引

| 意图 | 推荐命令 | 备注 |
|------|---------|------|
| 创建进销存系统 | `+base-create` → 4 × `+table-create` → 逐个 `+field-create` → 双向关联 → lookup + formula | 按 Step 1 顺序执行 |
| 新增货品 | `+record-upsert` → products 表 | |
| 修改货品 | `+record-list` → `+record-upsert` → products 表 | 先查 record_id |
| 删除货品 | `+record-delete` → products 表 | 带 `--yes` |
| 入库 | `+record-upsert` → stock_in 表 | 关联字段用 record_id 数组 |
| 出库 | 先校验库存 → `+record-upsert` → stock_out 表 | 必须校验 |
| 查库存 | `+record-list` → products 表 | AI 过滤/聚合 |
| 查入库记录 | `+record-list` → stock_in 表 | |
| 查出库记录 | `+record-list` → stock_out 表 | |
| 低库存预警 | `+record-list` → products 表 → AI 过滤 | |
| 发预警通知 | `im +messages-send` | 需 chat_id |
| 入库/出库汇总 | `+data-query` | 只聚合存储字段，table-id 放 DSL 中 |
| 滞销分析 | `+record-list` → products 表 → AI 计算 | 用 lookup 字段值 |
| 生成报表 | `docs +create` | Markdown 模板 |
| 创建看板 | `+dashboard-create` → `+dashboard-block-create` | |
| 新增供应商 | `+record-upsert` → suppliers 表 | |
| 查供应商 | `+record-list` → suppliers 表 | |
| 关联供应商 | `+record-upsert` → products 表 | 关联字段用 record_id 数组 |

---

## 常见错误速查

| 错误场景 | 原因 | 解决方案 |
|---------|------|---------|
| `+table-create --fields` 限流 800004135 | 一次性创建多字段触发限流 | 先建空表，再用 `+field-create` 逐个加字段，间隔 1.5 秒 |
| 公式字段跨表查询返回全表 SUM | 单向关联无法反向查询；或公式用表名引用而非关联字段 | 关联字段必须 `bidirectional: true`；库存计算改用 lookup + formula 组合 |
| 单向关联不能改为双向 | API 限制 unsafe_operation_blocked | 删除旧关联字段，重新创建双向关联 |
| `+data-query` unknown flag `--table-id` | 该命令不支持 `--table-id` | 把 table-id 放在 `--dsl` 的 `datasource.table.tableId` 中 |
| `+data-query` 不支持某字段聚合 | 公式/lookup/关联字段不在白名单 | 只对存储字段做聚合 |
| `+data-query` alias 中文乱码 | CLI 编码问题 | 用英文 alias 或 agent 自行映射 |
| 关联字段值写入失败 | 格式错误 | 关联字段值必须是 record_id 数组 `["recXXX"]` |
| 日期字段写入失败 | 格式错误 | 使用 Unix 毫秒时间戳（数字），不是字符串 |
| 出库后库存未更新 | lookup 公式计算延迟 | 等待 3-5 秒后重新查询 |
| `+record-list` 数据不全 | 分页未拉完 | 用 offset 分页，每页 200 |
| 权限不足 | scope 未开通 | 参考 lark-shared 处理权限问题 |
| 并发报错 1254291 | list 命令并发 | 所有 `+xxx-list` 串行执行 |
| 消息发送失败 | chat_id 无效 | 用 `im +chat-list` 确认 chat_id |

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
| `base +dashboard-create` | `bitable:app` |
| `im +messages-send` | `im:message` |
| `docs +create` | `docx:document:create` |

---

## 参考

- [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`../lark-base/SKILL.md`](../lark-base/SKILL.md) — 多维表格原子操作
- [`../lark-base/references/formula-field-guide.md`](../lark-base/references/formula-field-guide.md) — 公式字段写法（创建公式字段前必读）
- [`../lark-base/references/lookup-field-guide.md`](../lark-base/references/lookup-field-guide.md) — lookup 字段配置规则（创建 lookup 字段前必读）
- [`../lark-base/references/lark-base-shortcut-field-properties.md`](../lark-base/references/lark-base-shortcut-field-properties.md) — 字段属性 JSON 规范
- [`../lark-base/references/lark-base-shortcut-record-value.md`](../lark-base/references/lark-base-shortcut-record-value.md) — 记录值格式规范
- [`../lark-base/references/lark-base-data-query.md`](../lark-base/references/lark-base-data-query.md) — 聚合分析 DSL
- [`../lark-base/references/dashboard-block-data-config.md`](../lark-base/references/dashboard-block-data-config.md) — 仪表盘 Block 配置
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 飞书消息发送
- [`../lark-doc/SKILL.md`](../lark-doc/SKILL.md) — 飞书文档创建
