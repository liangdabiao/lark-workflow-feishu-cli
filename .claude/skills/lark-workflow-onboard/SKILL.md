---
name: lark-workflow-onboard
version: 1.0.0
description: "新人入职向导：自动发送欢迎消息、添加到相关群聊、创建入职任务清单、设置日历提醒、通知HR和直属经理。一次触发，多系统联动。当用户需要'新人入职'、'入职引导'、'新员工入职'、'入职流程'、'欢迎新人'、'入职清单执行'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 新人入职向导工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "张三今天入职，帮他安排一下" / "新人入职"
- "新人入职清单执行：张三，工程部" / "入职引导"
- "帮我走一下新员工的入职流程" / "入职流程"
- "新同事入职，帮他创建入职任务" / "欢迎新人"

## 前置条件

混合身份：**user**（拉群、创建任务、日程）和 **bot**（发消息）。

```bash
lark-cli auth login --domain im,contact,task,calendar
```

## 工作流

```
{新员工信息: 姓名, 部门, 岗位, [HR open_id], [直属经理 open_id]}
  │
  ├── contact +search-user ─────────► 查找新员工 open_id
  │
  ├── im +messages-send ────────────► ① 发送欢迎消息
  │
  ├── im chat.members create ───────► ② 添加到部门群/项目群
  │
  ├── task +create ─────────────────► ③ 创建入职任务清单（串行，间隔 1s）
  │
  ├── calendar +create ─────────────► ④ 设置日历提醒
  │     ├── 30天试用期评估
  │     └── 团队迎新会
  │
  └── im +messages-send ────────────► ⑤ 通知 HR + 直属经理
```

---

## Step 1: 查找新员工

```bash
lark-cli contact +search-user --query "<新员工姓名>" --as user
```

从返回结果中提取 `open_id`（`ou_xxx` 格式）。如果查不到，提示用户提供姓名或 open_id。

## Step 2: 发送欢迎消息

```bash
lark-cli im +messages-send \
  --user-id "<new_user_open_id>" \
  --markdown "欢迎加入{公司/团队名}！

以下是入职指南：
1. 完成账号和权限初始化
2. 加入相关群聊
3. 领取办公设备
4. 了解团队协作方式

如有任何问题，随时联系你的导师或 HR。" \
  --as bot
```

> **注意**：欢迎消息可根据部门和岗位自定义内容。可附加公司 wiki 链接、员工手册等资源。

## Step 3: 添加到群聊

```bash
# 先查看 schema 确认参数
lark-cli schema im.chat.members.create

# 添加到部门群（需要 user 身份）
lark-cli im chat.members create \
  --params '{"chat_id":"<dept_chat_id>","member_id_type":"open_id","succeed_type":1}' \
  --data '{"id_list":["<new_user_open_id>"]}' \
  --as user
```

如果需要添加到多个群，逐个调用，每次间隔 1 秒。

> **注意**：
> - `succeed_type`: 1=不需要验证直接加入（适用于内部群）
> - 如果机器人不在群中，需要先将机器人拉入群
> - 对方必须是同一租户内的用户

## Step 4: 创建入职任务清单

逐一创建入职任务（**每次间隔 1 秒**，避免频率限制）：

```bash
# 任务1：领取办公设备
lark-cli task +create \
  --summary "领取办公设备（{姓名}）" \
  --description "联系 IT 部门领取笔记本电脑、显示器等办公设备" \
  --due "<7天后日期>" \
  --as user

# 任务2：完成入职培训
lark-cli task +create \
  --summary "完成入职培训（{姓名}）" \
  --description "完成公司在线培训课程，包括安全培训、合规培训等" \
  --due "<14天后日期>" \
  --as user

# 任务3：认识团队成员
lark-cli task +create \
  --summary "认识团队成员（{姓名}）" \
  --description "与团队成员逐一交流，了解协作方式和项目背景" \
  --due "<7天后日期>" \
  --as user

# 任务4：完成试用期目标设定
lark-cli task +create \
  --summary "完成试用期目标设定（{姓名}）" \
  --description "与直属经理讨论并确定试用期工作目标和考核标准" \
  --due "<14天后日期>" \
  --as user
```

> **注意**：`--due` 使用系统 `date` 命令计算目标日期，不要心算。任务创建后记录 task_id。

## Step 5: 设置日历提醒

```bash
# 30天试用期评估提醒
lark-cli calendar +create \
  --summary "30天试用期评估 - {姓名}" \
  --start "<30天后 10:00>" \
  --end "<30天后 11:00>" \
  --as user

# 团队迎新会（如适用）
lark-cli calendar +create \
  --summary "团队迎新会 — 欢迎{姓名}" \
  --start "<本周五 15:00>" \
  --end "<本周五 16:00>" \
  --attendee-ids "<new_user_open_id>,<manager_open_id>" \
  --as user
```

> **注意**：创建日程前与用户确认时间和内容，避免误操作。

## Step 6: 通知 HR 和直属经理

```bash
# 通知 HR
lark-cli im +messages-send \
  --user-id "<hr_open_id>" \
  --markdown "新员工入职通知

姓名：{姓名}
部门：{部门}
岗位：{岗位}
入职日期：{日期}

入职引导已完成以下操作：
- 已发送欢迎消息
- 已添加到部门群
- 已创建入职任务清单
- 已设置试用期评估提醒" \
  --as bot

# 通知直属经理
lark-cli im +messages-send \
  --user-id "<manager_open_id>" \
  --markdown "您团队的新成员 {姓名} 今日入职！

已为 {姓名} 创建了入职任务清单和试用期评估提醒。
请在本周内与 {姓名} 讨论试用期目标。" \
  --as bot
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 查不到新员工 | 提示用户提供 open_id |
| 群聊添加失败 | 列出失败群聊，让用户手动添加 |
| 任务创建频率限制 | 降低创建速度，间隔 2 秒 |
| 日程创建失败 | 仅创建任务，日程让用户手动添加 |
| HR/经理 open_id 未知 | 询问用户提供，或跳过通知步骤 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `contact +search-user` | `contact:user.base:readonly` |
| `im +messages-send` | `im:message` |
| `im chat.members create` | `im:chat.member:write` |
| `task +create` | `task:task:write` |
| `calendar +create` | `calendar:calendar:write`, `calendar:event:write` |

## 参考

- [`../lark-contact/SKILL.md`](../lark-contact/SKILL.md) — 通讯录查询
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — IM 操作
- [`../lark-task/SKILL.md`](../lark-task/SKILL.md) — 任务管理
- [`../lark-calendar/SKILL.md`](../lark-calendar/SKILL.md) — 日历操作
