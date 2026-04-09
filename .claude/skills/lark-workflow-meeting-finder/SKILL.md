---
name: lark-workflow-meeting-finder
version: 1.0.0
description: "日程冲突检测器：查询多个参与者的忙闲状态，找到共同的空闲时段，可选直接创建会议日程。当用户需要'找时间开会'、'协调会议时间'、'查看空闲时段'、'日程冲突'、'约会议'、'有空什么时候'、'大家什么时候有空'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 日程冲突检测器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "查一下明天下午大家有空吗" / "查看空闲时段"
- "安排一个1小时的会，参会人：张三、李四、王五" / "约会议"
- "下周三下午我和张三、李四有没有时间开会" / "日程冲突"
- "找个大家都空的时间开会" / "协调会议时间"
- "看看今天有没有空隙可以开个会"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain calendar,contact
```

## 工作流

```
{参会人列表 + 时间范围 + 会议时长}
  │
  ├── contact +search-user ──────────► 查找参会人 open_id
  │
  ├── calendar +freebusy ────────────► 查询各参会人忙闲
  │
  ├── AI 分析空闲交集 ──────────────► 找出共同空闲时段
  │
  ├── [有明确时间] 冲突检测 ─────────► 报告冲突详情
  │
  ├── [有时间段] +suggestion ────────► 推荐多个可选时段
  │
  └── [用户确认] calendar +create ───► 创建会议日程 + 发邀请
```

---

## Step 1: 查找参会人

如果用户提供了姓名，先查找 open_id：

```bash
lark-cli contact +search-user --query "<参会人姓名>" --as user
```

记录返回的 `open_id`（`ou_xxx` 格式）。如果查不到，提示用户检查姓名是否正确。

> **注意**：只能查找同一飞书租户内的用户。跨租户用户无法查询忙闲状态。

## Step 2: 查询忙闲状态

### 逐人查询

```bash
# 查询单个用户的忙闲（默认今天）
lark-cli calendar +freebusy --user-id "ou_xxx" --as user

# 指定日期范围
lark-cli calendar +freebusy \
  --user-id "ou_xxx" \
  --start "2026-04-10T09:00:00+08:00" \
  --end "2026-04-10T18:00:00+08:00" \
  --as user
```

> **注意**：`--start` / `--end` 仅支持 ISO 8601 格式和 Unix timestamp，**不支持**自然语言。需要 AI 根据当前日期计算。

### 批量查询

对多个参会人，逐一调用 `+freebusy`（当前 lark-cli 不支持一次传多个 user-id）：

```bash
# 逐个查询，记录每个人的忙碌时段
lark-cli calendar +freebusy --user-id "ou_aaa" --start "..." --end "..." --as user
lark-cli calendar +freebusy --user-id "ou_bbb" --start "..." --end "..." --as user
lark-cli calendar +freebusy --user-id "ou_ccc" --start "..." --end "..." --as user
```

## Step 3: AI 分析空闲时段

将所有参会人的忙闲数据整合，计算**共同空闲时段**：

1. 将每个人的忙碌时段标记在时间轴上
2. 取所有忙碌时段的并集
3. 在非忙碌时段中，筛选出满足**会议时长**要求的连续空闲段
4. 按推荐优先级排序（工作时间优先，避开午休）

**输出格式：**

```
## 推荐会议时段

以下为 {日期} 所有参会人均空闲的时段（会议时长 {N} 分钟）：

| 时段 | 推荐度 | 说明 |
|------|--------|------|
| 09:00-10:00 | 推荐 | 工作时间，无冲突 |
| 14:00-15:00 | 推荐 | 工作时间，无冲突 |
| 11:00-12:00 | 一般 | 接近午休 |

### 冲突详情
| 参会人 | 忙碌时段 | 事件 |
|--------|---------|------|
| 张三 | 10:00-11:30 | 产品评审会 |
| 李四 | 13:00-14:00 | 1v1 |
```

### 智能推荐（可选）

如果用户给出了时间范围但没指定具体时间，可使用 `+suggestion` 获取系统推荐：

```bash
lark-cli calendar +suggestion \
  --start "2026-04-10T09:00:00+08:00" \
  --end "2026-04-10T18:00:00+08:00" \
  --attendee-ids "ou_aaa,ou_bbb,ou_ccc" \
  --duration-minutes 60 \
  --as user
```

## Step 4: 创建会议（可选）

用户确认时间段后，创建日程并发送邀请：

```bash
lark-cli calendar +create \
  --summary "会议主题" \
  --start "2026-04-10T09:00:00+08:00" \
  --end "2026-04-10T10:00:00+08:00" \
  --attendee-ids "ou_aaa,ou_bbb,ou_ccc" \
  --as user
```

> **注意**：创建前务必与用户确认时间、主题和参会人，避免误操作。

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 某参会人查不到 | 提示用户确认姓名，或跳过该人继续查询 |
| 无共同空闲时段 | 建议：扩大时间范围 / 减少参会人 / 拆分会议 |
| 只能找到非理想时段 | 列出"次优时段"，说明冲突详情 |
| +suggestion 不可用 | 退回到 +freebusy 手动分析 |
| 创建日程失败 | 输出推荐时段让用户手动创建 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `contact +search-user` | `contact:user.base:readonly` |
| `calendar +freebusy` | `calendar:freebusy:readonly` |
| `calendar +suggestion` | `calendar:freebusy:readonly` |
| `calendar +create` | `calendar:calendar:write`, `calendar:event:write` |

## 参考

- [`../lark-calendar/SKILL.md`](../lark-calendar/SKILL.md) — 日历原子操作
- [`../lark-contact/SKILL.md`](../lark-contact/SKILL.md) — 通讯录查询
