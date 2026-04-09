---
name: lark-workflow-approval-reminder
version: 1.0.0
description: "审批催办机器人：自动查询待审批实例，定位当前审批人并发送催办提醒，支持逐级升级催办策略。当用户需要'催审批'、'催办'、'审批提醒'、'提醒审批人'、'跟进审批流程'、'审批到谁了'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 审批催办机器人工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我催一下那个报销审批" / "催审批"
- "审批单12345到谁手里了？" / "审批到谁了"
- "帮我催办一下审批流程" / "催办"
- "提醒一下审批人处理审批" / "审批提醒"
- "跟进一下XXX的审批进度" / "跟进审批流程"

## 前置条件

需要 **user 身份**（查询审批）和 **bot 身份**（发消息催办）。

```bash
lark-cli auth login --domain im,contact --scope "approval:instance:readonly"
```

> **注意**：审批 API 未被 lark-cli 直接封装，需要使用 `lark-cli api` 调用。首次使用前，参考 [`../lark-openapi-explorer/SKILL.md`](../lark-openapi-explorer/SKILL.md) 确认 API 路径。

## 工作流

```
{审批实例 ID 或 approval_code}
  │
  ├── lark-cli api GET ────────────► 查询审批实例状态
  │     /approval/v4/instances/{id}
  │
  ├── AI 提取当前审批人 ───────────► approver_open_id
  │
  ├── contact +search-user ─────────► 获取审批人信息（姓名等）
  │
  ├── im +messages-send ────────────► 发送催办消息
  │
  └── [超24h未处理] ────────────────► 升级催办策略
        ├── 群内 @催办
        └── 通知上级/HR
```

---

## Step 1: 查询审批实例状态

### 方式一：按实例 ID 查询

如果用户提供了审批实例 ID：

```bash
MSYS_NO_PATHCONV=1 lark-cli api GET "/open-apis/approval/v4/instances/<instance_id>" \
  --as user
```

### 方式二：按审批定义 code 查询

如果用户提供了审批定义的 code（或名称）：

```bash
# 先通过 openapi-explorer 确认 API 路径
# 查询待审批实例列表
MSYS_NO_PATHCONV=1 lark-cli api GET "/open-apis/approval/v4/instances" \
  --params '{
    "approval_code": "<approval_code>",
    "status": "1",
    "page_size": 50
  }' \
  --as user
```

**status 参数说明：**
- `1` — 审批中（待处理）
- `2` — 已通过
- `3` — 已拒绝
- `4` — 已撤销
- `6` — 已转交

### API 文档确认

> 审批 API 未被 lark-cli 直接封装。调用前，建议使用 openapi-explorer 模式确认最新 API 路径和参数：
>
> 1. 获取顶层索引：`https://open.feishu.cn/llms.txt`
> 2. 找到审批模块文档链接
> 3. 确认具体 API 的参数和响应格式

## Step 2: 提取当前审批人

从审批实例详情中提取：

- `instance_code` — 实例编号
- `title` — 审批标题
- `status` — 当前状态
- 当前审批节点和审批人信息

AI 从返回结果中分析出**当前待处理的审批节点**和**对应的审批人**。

审批人可能是个人（`open_id`）也可能是群组，需要分别处理。

## Step 3: 查找审批人信息

```bash
lark-cli contact +search-user --query "<审批人姓名>" --as user
```

获取审批人的 `open_id`。

## Step 4: 发送催办消息

```bash
lark-cli im +messages-send \
  --user-id "<approver_open_id>" \
  --markdown "审批催办提醒

审批单：{审批标题}
审批编号：{instance_code}
提交时间：{submit_time}
当前状态：待您审批

请您尽快处理，如有问题请联系发起人。" \
  --as bot
```

## Step 5: 升级催办（可选）

如果审批已超过 24 小时未处理，启动升级策略：

### 第二级：群内 @催办

```bash
lark-cli im +messages-send \
  --chat-id "<相关群chat_id>" \
  --markdown '<at user_id="ou_xxx"></at> 您有一笔审批单待处理已超过 24 小时，请尽快处理：

审批单：{审批标题}
审批编号：{instance_code}' \
  --as bot
```

### 第三级：通知上级

如果超过 48 小时仍未处理，通知审批人的直属上级或发起人：

```bash
lark-cli im +messages-send \
  --user-id "<上级或发起人_open_id>" \
  --markdown "审批超时通知

审批单：{审批标题}
审批编号：{instance_code}
当前审批人：{审批人姓名}
等待时间：{N} 小时

请协助推动审批流程。" \
  --as bot
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 审批 API 不可用 | 提示用户检查 approval 权限是否开通 |
| 查不到审批实例 | 提示用户确认实例 ID 或审批名称 |
| 审批人查不到 | 使用返回的 open_id 直接发送消息 |
| 消息发送失败 | 输出催办信息让用户手动通知 |
| 无群聊可 @ | 跳过群内 @催办，直接通知个人 |
| 审批已完成 | 告知用户"该审批已处理完毕"，无需催办 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `approval/v4/instances (GET)` | `approval:instance:readonly` |
| `contact +search-user` | `contact:user.base:readonly` |
| `im +messages-send` | `im:message` |

## 参考

- [`../lark-openapi-explorer/SKILL.md`](../lark-openapi-explorer/SKILL.md) — 原生 API 调用模式（审批 API 未封装，需用此模式）
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 消息发送
- [`../lark-contact/SKILL.md`](../lark-contact/SKILL.md) — 通讯录查询
