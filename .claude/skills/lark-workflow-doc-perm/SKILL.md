---
name: lark-workflow-doc-perm
version: 1.0.0
description: "文档权限管家：批量设置飞书文档权限，按角色或人员批量授权（可编辑/只读/完整权限），通知被授权用户。当用户需要'设置文档权限'、'批量授权'、'文档权限管理'、'分享文档给团队'、'文档权限配置'、'把文档分享给XX'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 文档权限管家工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "把文档XXX分享给张三、李四，可编辑" / "设置文档权限"
- "项目文档权限批量设置：工程组只读，产品组可编辑" / "批量授权"
- "把这篇文档的编辑权限给整个团队" / "文档权限管理"
- "分享文档给团队" / "文档权限配置"

## 前置条件

需要 **user 身份**（操作自己文档的权限）和 **bot 身份**（发消息通知）。执行前确保已授权：

```bash
lark-cli auth login --domain drive,wiki,contact,im
```

## 工作流

```
{文档 URL 或 file_token + 授权人列表 + 权限级别}
  │
  ├── [wiki 链接] wiki spaces get_node ──► 解析真实 token + obj_type
  │
  ├── drive permission.members create ──► 批量授权（逐个，间隔 1s）
  │
  └── im +messages-send ────────────────► 通知被授权用户
```

---

## Step 1: 解析文档 token

### wiki 链接处理

如果用户提供的是知识库链接（`https://xxx.feishu.cn/wiki/xxx`），先解析获取真实文档 token：

```bash
lark-cli wiki spaces get_node \
  --params '{"token":"<wiki_token>"}' \
  --as user
```

从返回结果中提取 `obj_token`（真实文档 token）和 `obj_type`（文档类型）。

### 直接 token

如果用户直接提供了文档 token（`doccnXXX`、`sheetXXX`、`bascnXXX` 等），可直接使用。

### 确定文档类型

| obj_type | 说明 | type 参数 |
|----------|------|----------|
| `docx` | 新版文档 | `docx` |
| `sheet` | 电子表格 | `sheet` |
| `bitable` | 多维表格 | `bitable` |
| `doc` | 旧版文档 | `doc` |
| `mindnote` | 思维导图 | `mindnote` |
| `file` | 文件 | `file` |

## Step 2: 查找授权目标用户

```bash
lark-cli contact +search-user --query "<用户姓名>" --as user
```

记录每个用户的 `open_id`。支持批量查找多个用户。

## Step 3: 设置权限

### 查看参数结构

```bash
lark-cli schema drive.permission.members.create
```

### 逐个授权

对每个用户调用一次（**间隔 1 秒**，避免频率限制）：

```bash
lark-cli drive permission.members create \
  --params '{"token":"<file_token>","type":"<doc_type>"}' \
  --data '{
    "member_type": "openid",
    "member_id": "ou_xxx",
    "perm": "edit"
  }' \
  --as user
```

**权限级别：**

| perm 值 | 说明 | 适用场景 |
|---------|------|---------|
| `full_access` | 完整权限 | 文档管理员、核心协作者 |
| `edit` | 可编辑 | 需要修改文档的协作者 |
| `view` | 只读 | 仅需查看的成员 |

### 按角色批量设置示例

```bash
# 工程组：只读
for user_id in "ou_aaa" "ou_bbb" "ou_ccc"; do
  lark-cli drive permission.members create \
    --params '{"token":"<file_token>","type":"docx"}' \
    --data "{\"member_type\":\"openid\",\"member_id\":\"$user_id\",\"perm\":\"view\"}" \
    --as user
  sleep 1
done

# 产品组：可编辑
for user_id in "ou_ddd" "ou_eee"; do
  lark-cli drive permission.members create \
    --params '{"token":"<file_token>","type":"docx"}' \
    --data "{\"member_type\":\"openid\",\"member_id\":\"$user_id\",\"perm\":\"edit\"}" \
    --as user
  sleep 1
done
```

> **重要**：
> - 设置 `full_access` 前必须与用户确认（安全规则）
> - 每次调用间隔至少 1 秒
> - 如果提示"无权限操作"，需要确认当前用户是文档所有者或管理员

## Step 4: 通知被授权用户

```bash
lark-cli im +messages-send \
  --user-id "<user_open_id>" \
  --markdown "已为您开通文档「{文档标题}」的{权限级别}权限。

文档链接：{文档URL}

如有问题，请联系文档管理员。" \
  --as bot
```

> **注意**：通知消息间隔 1 秒发送。如果授权人数超过 20 人，分批发送。

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| wiki 链接解析失败 | 询问用户直接提供文档 token |
| 用户查不到 | 提示确认姓名，或直接使用 open_id |
| 权限设置失败 | 检查当前用户是否有文档管理权限 |
| 通知发送失败 | 列出已授权用户，让用户手动通知 |
| 频率限制 | 降低速度到每 2 秒一次 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `wiki spaces get_node` | `wiki:node:read` |
| `contact +search-user` | `contact:user.base:readonly` |
| `drive permission.members create` | `drive:drive:write` |
| `im +messages-send` | `im:message` |

## 参考

- [`../lark-drive/SKILL.md`](../lark-drive/SKILL.md) — 云空间原子操作
- [`../lark-wiki/SKILL.md`](../lark-wiki/SKILL.md) — 知识库操作
- [`../lark-contact/SKILL.md`](../lark-contact/SKILL.md) — 通讯录查询
- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — 消息发送
