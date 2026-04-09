---
name: lark-workflow-bitable-sync
version: 1.0.0
description: "跨表数据同步器：从源多维表格读取数据，与目标表进行比对去重，将差异数据写入目标表。支持字段映射、增量同步、变更报告。当用户需要'数据同步'、'跨表同步'、'合并表格'、'数据迁移'、'表间数据同步'、'把A表的数据同步到B表'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 跨表数据同步器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "把表格A的新数据同步到表格B" / "数据同步"
- "跨表同步：收集表 → 跟进表" / "跨表同步"
- "合并这两个表格的数据" / "合并表格"
- "把A表的数据迁移到B表" / "数据迁移"
- "监控表格XXX，有新记录就通知我" / "表间数据同步"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain base
```

## 工作流

```
{源表 base_token + table_id} → {目标表 base_token + table_id}
  │
  ├── base +field-list (源表) ──────► 源表字段结构
  │
  ├── base +field-list (目标表) ────► 目标表字段结构
  │
  ├── AI 字段映射 ──────────────────► 源字段 ↔ 目标字段（用户确认）
  │
  ├── base +record-list (源表) ────► 拉取源数据（分页，max 200/页）
  │
  ├── base +record-list (目标表) ──► 拉取目标现有数据（用于去重）
  │
  ├── AI 数据比对去重 ──────────────► 差异记录集（新增 + 更新）
  │
  └── base +record-upsert (目标表) ─► 写入差异数据（串行，间隔 1s）
```

---

## Step 1: 获取表结构

分别获取源表和目标表的字段列表：

```bash
# 源表字段
lark-cli base +field-list \
  --base-token "<src_base_token>" \
  --table-id "<src_table_id>" \
  --as user

# 目标表字段
lark-cli base +field-list \
  --base-token "<tgt_base_token>" \
  --table-id "<tgt_table_id>" \
  --as user
```

> **注意**：两个 `+field-list` 调用不能并发，必须串行。

## Step 2: AI 生成字段映射

根据两张表的字段名和类型，AI 自动生成字段映射方案：

```markdown
## 字段映射方案

| 源字段 | 源类型 | → | 目标字段 | 目标类型 |
|--------|--------|---|---------|---------|
| 客户名称 | text | → | 公司名称 | text |
| 联系电话 | text | → | 电话 | text |
| 状态 | select | → | 跟进状态 | select |
| 创建时间 | datetime | → | 录入日期 | datetime |

**去重字段**：客户名称（以此字段判断记录是否已存在）

请确认此映射方案。
```

> **重要**：
> - 必须与用户确认字段映射方案后再执行同步
> - 让用户指定去重字段（用于判断记录是否已存在）
> - 只映射**可写入字段**（忽略公式、查找引用、系统字段）

## Step 3: 拉取源数据

```bash
# 拉取源表全部记录（分页，每页最多 200 条）
lark-cli base +record-list \
  --base-token "<src_base_token>" \
  --table-id "<src_table_id>" \
  --as user

# 如果数据量大，使用分页
lark-cli base +record-list \
  --base-token "<src_base_token>" \
  --table-id "<src_table_id>" \
  --limit 200 \
  --as user
```

> **注意**：每页最多 200 条记录。如果返回数据中有分页信息，继续使用 offset 拉取下一页。

## Step 4: 拉取目标数据（去重用）

```bash
# 拉取目标表现有记录（获取去重字段值）
lark-cli base +record-list \
  --base-token "<tgt_base_token>" \
  --table-id "<tgt_table_id>" \
  --as user
```

## Step 5: AI 比对去重

对比源数据和目标数据，基于去重字段判断：

- **新增**：源表中有但目标表中没有的记录
- **更新**：源表和目标表都有的记录，但字段值有差异

生成差异报告：

```markdown
## 同步预览

| 类型 | 数量 |
|------|------|
| 新增记录 | N |
| 需更新记录 | N |
| 无变化记录 | N |
| **总计** | **N** |

### 新增记录预览（前 5 条）
| 客户名称 | 联系电话 | 状态 |
|---------|---------|------|

### 需更新记录预览（前 5 条）
| 客户名称 | 字段 | 旧值 | 新值 |
|---------|------|------|------|

确认执行同步？
```

## Step 6: 写入差异数据

对每条差异记录调用 `+record-upsert`（**串行，间隔 1 秒**）：

### 新增记录

```bash
lark-cli base +record-upsert \
  --base-token "<tgt_base_token>" \
  --table-id "<tgt_table_id>" \
  --json '{"公司名称":"XX科技","电话":"13800138000","跟进状态":"新线索"}' \
  --as user
sleep 1
```

### 更新记录

先获取目标记录的 record_id，再更新：

```bash
lark-cli base +record-upsert \
  --base-token "<tgt_base_token>" \
  --table-id "<tgt_table_id>" \
  --record-id "<existing_record_id>" \
  --json '{"跟进状态":"已联系"}' \
  --as user
sleep 1
```

> **注意**：
> - 每次调用间隔至少 1 秒
> - 日期字段需要 Unix 毫秒时间戳（不是字符串）
> - 人员字段格式为 `[{"id":"ou_xxx"}]`
> - 单选字段传选项名称字符串

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 字段类型不匹配 | 跳过不兼容字段，只同步兼容字段 |
| 目标表无匹配字段 | 提示用户在目标表创建对应字段 |
| 数据量过大 | 分批同步（每批 50 条） |
| 写入频率限制 | 降低到每 2 秒一条 |
| 字段映射不明确 | 询问用户确认映射关系 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `base +field-list` | `bitable:app:readonly` |
| `base +record-list` | `bitable:app:readonly` |
| `base +record-upsert` | `bitable:app` |

## 参考

- [`../lark-base/SKILL.md`](../lark-base/SKILL.md) — 多维表格原子操作
