---
name: lark-workflow-announce
version: 1.0.0
description: "群公告管理器：获取、设置、追加群公告内容，并通知群成员。当用户需要'群公告'、'发布公告'、'更新公告'、'设置群公告'、'追加公告'、'群通知'、'更新群公告'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 群公告管理器工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "更新XX群的公告为..." / "设置群公告"
- "在群公告末尾追加一条通知" / "追加公告"
- "看看XX群现在的公告是什么" / "群公告"
- "发布一条群公告，@所有人" / "发布公告"
- "把项目里程碑更新到群公告" / "更新公告"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain im
```

## 工作流

```
{chat_id + 操作类型（查看/设置/追加）+ 公告内容}
  │
  ├── im +chat-search ──────────────► 查找群聊（获取 chat_id）
  │
  ├── [查看] lark-cli api GET ──────► 读取当前群公告
  │
  ├── [追加] 先读取再合并 ─────────► 保留旧公告 + 追加新内容
  │
  ├── [设置] lark-cli api PATCH ────► 设置新公告（注意 revision）
  │
  └── [可选] im +messages-send ────► @全体成员通知公告更新
```

---

## Step 1: 查找群聊

如果用户提供了群名称，先获取 chat_id：

```bash
lark-cli im +chat-search --query "<群名称>" --as user
```

从返回结果中提取 `chat_id`（`oc_xxx` 格式）。

## Step 2: 查看当前群公告

群公告 API 未被 lark-cli 直接封装，使用 `lark-cli api` 调用：

```bash
# 读取当前群公告
MSYS_NO_PATHCONV=1 lark-cli api GET "/open-apis/im/v1/chats/<chat_id>/announcement" \
  --as user
```

返回结果包含：
- `content` — 当前公告 HTML 内容
- `revision` — 版本号（设置公告时必须传递，用于乐观并发控制）

如果返回 `code: 0` 但 `data` 为空，说明当前没有群公告。

## Step 3: 设置群公告

### 覆盖模式

用新内容完全替换旧公告：

```bash
MSYS_NO_PATHCONV=1 lark-cli api PATCH "/open-apis/im/v1/chats/<chat_id>/announcement" \
  --data '{
    "revision": "<current_revision>",
    "requests": ["<HTML 公告内容>"]
  }' \
  --as user
```

### 追加模式

先读取当前公告，在末尾追加新内容：

```bash
MSYS_NO_PATHCONV=1 lark-cli api PATCH "/open-apis/im/v1/chats/<chat_id>/announcement" \
  --data '{
    "revision": "<current_revision>",
    "requests": ["<旧公告内容><br><br>---<br><br><新追加内容>"]
  }' \
  --as user
```

> **重要**：
> - `revision` 字段**必须**传递当前的版本号，否则会返回错误
> - 公告内容格式为 **HTML**（不是 Markdown），支持 `<br>`, `<b>`, `<a href="">`, `<font color="">` 等标签
> - `requests` 是一个字符串数组，每个元素是一段 HTML 内容
> - 设置前**必须先读取**获取最新 revision

### Markdown 转 HTML 参考

AI 需要将用户提供的 Markdown 内容转换为 HTML：

| Markdown | HTML |
|----------|------|
| `**加粗**` | `<b>加粗</b>` |
| `# 标题` | `<font size="5"><b>标题</b></font>` |
| `- 列表项` | `列表项<br>` |
| `[链接](url)` | `<a href="url">链接</a>` |

## Step 4: 通知群成员（可选）

```bash
lark-cli im +messages-send \
  --chat-id "<chat_id>" \
  --text '<at user_id="all"></at> 群公告已更新，请查看群公告。' \
  --as bot
```

## 降级策略

| 场景 | 降级方案 |
|------|---------|
| 群公告 API 失败 | 用普通消息发送公告内容到群里 |
| revision 冲突 | 重新读取获取最新 revision，再设置 |
| 无群公告权限 | 提示用户需要群管理员权限 |
| 群聊找不到 | 提示用户确认群名称或提供 chat_id |
| @所有人受限 | 改为发送普通消息通知 |

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `im +chat-search` | `im:chat:readonly` |
| `im/v1/chats/:id/announcement (GET)` | `im:chat:readonly` |
| `im/v1/chats/:id/announcement (PATCH)` | `im:chat` |
| `im +messages-send` | `im:message` |

## 参考

- [`../lark-im/SKILL.md`](../lark-im/SKILL.md) — IM 原子操作
- [`../lark-openapi-explorer/SKILL.md`](../lark-openapi-explorer/SKILL.md) — 原生 API 调用模式
