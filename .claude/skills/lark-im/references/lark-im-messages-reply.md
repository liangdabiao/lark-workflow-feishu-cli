# im +messages-reply

> **Prerequisite:** Read [`../lark-shared/SKILL.md`](../../lark-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

Reply to a specific message. Only supports bot identity. Also supports thread replies.

This skill maps to the shortcut: `lark-cli im +messages-reply` (internally calls `POST /open-apis/im/v1/messages/:message_id/reply`).

## Safety Constraints

Replies sent by this tool are visible to other people. Before calling it, you **must** confirm with the user:

1. Which message to reply to
2. The reply content
3. Which identity to use (bot only)

**Do not** send a reply without explicit user approval.

When using `--as bot`, the reply is sent in the app's name, so make sure the app has already been added to the target chat.

## Commands

```bash
# Reply to a message (plain text, bot identity, --text is recommended)
lark-cli im +messages-reply --message-id om_xxx --text "Received"

# Equivalent manual JSON
lark-cli im +messages-reply --message-id om_xxx --content '{"text":"Received"}'

# Reply as a bot
lark-cli im +messages-reply --message-id om_xxx --text "bot reply" --as bot

# Reply inside the thread (message appears in the target thread)
lark-cli im +messages-reply --message-id om_xxx --text "Let's discuss this" --reply-in-thread

# Bot identity + thread reply
lark-cli im +messages-reply --message-id om_xxx --text "bot reply" --as bot --reply-in-thread

# Reply with a rich-text message
lark-cli im +messages-reply --message-id om_xxx --msg-type post --content '{"zh_cn":{"title":"Reply","content":[[{"tag":"text","text":"Detailed content"}]]}}'

# Reply with a local image (uploaded automatically before sending)
lark-cli im +messages-reply --message-id om_xxx --image ./photo.png

# Reply with a local file (uploaded automatically before sending)
lark-cli im +messages-reply --message-id om_xxx --file ./report.pdf

# Reply with a local video (--video-cover is required as the video cover)
lark-cli im +messages-reply --message-id om_xxx --video ./demo.mp4 --video-cover ./cover.png

# With an idempotency key
lark-cli im +messages-reply --message-id om_xxx --text "Received" --idempotency-key my-unique-id

# Preview the request without executing it
lark-cli im +messages-reply --message-id om_xxx --text "Test" --dry-run
```

## Parameters

| Parameter | Required | Description |
|------|------|------|
| `--message-id <id>` | Yes | ID of the message being replied to (`om_xxx`) |
| `--msg-type <type>` | No | Message type (default `text`): `text`, `post`, `image`, `file`, `audio`, `media`, `interactive`, `share_chat`, `share_user` |
| `--content <json>` | One of content options | Reply content as a JSON string; format depends on `msg_type` |
| `--text <string>` | One of content options | Plain text message (automatically wrapped as `{"text":"..."}` JSON) |
| `--markdown <string>` | One of content options | Markdown text (auto-wrapped as post format with style optimization; image URLs auto-resolved) |
| `--image <path\|key>` | One of content options | Local image path, `image_key` (`img_xxx`)|
| `--file <path\|key>` | One of content options | Local file path, `file_key` (`file_xxx`)|
| `--video <path\|key>` | One of content options | Local video path, `file_key`; **must be used together with `--video-cover`** |
| `--video-cover <path\|key>` | **Required with `--video`** | Video cover image path, `image_key` (`img_xxx`) |
| `--audio <path\|key>` | One of content options | Local audio path, `file_key` |
| `--reply-in-thread` | No | Reply inside the thread. The reply appears in the target message's thread instead of the main chat stream |
| `--idempotency-key <key>` | No | Idempotency key; the same key sends only one reply within 1 hour |
| `--as <identity>` | No | Identity type: `bot` only |
| `--dry-run` | No | Print the request only, do not execute it |

> **Mutual exclusivity rule:** `--text`, `--markdown`, `--content`, and `--image`/`--file`/`--video`/`--audio` cannot be used together. Media flags are also mutually exclusive with each other.
>
> **Video cover rule:** `--video` **must** be accompanied by `--video-cover`. Omitting `--video-cover` when using `--video` will fail validation. `--video-cover` cannot be used without `--video`.

## Return Value

```json
{
  "message_id": "om_xxx",
  "chat_id": "oc_xxx",
  "create_time": "1234567890"
}
```

## Usage Scenarios

### Scenario 1: Reply in the main chat stream

```bash
lark-cli im +messages-reply --message-id om_xxx --text "OK, I will handle it"
```

The reply appears in the main chat stream and references the target message.

### Scenario 2: Reply inside a thread

```bash
lark-cli im +messages-reply --message-id om_xxx --text "Let me take a look at this" --reply-in-thread
```

The reply appears in the target message's thread and does not show up in the main chat stream.

## @Mention Format (text / post)

- @specific user: `<at user_id="ou_xxx">name</at>`
- @all: `<at user_id="all"></at>`

## Notes

- `--message-id` must be a valid message ID in `om_xxx` format
- `--content` must be a valid JSON string
- `--reply-in-thread` is only meaningful in group chats
- `--image`/`--file`/`--video`/`--audio`/`--video-cover` support local file paths; use relative paths within the current working directory. The shortcut automatically uploads the file first and then sends the reply
- If the provided value starts with `img_` or `file_`, it is treated as an existing key and used directly
- When using `--video`, `--video-cover` is **required** as the video cover. Omitting `--video-cover` with `--video` will produce a validation error. `--video-cover` cannot be used without `--video`
- Failures return error codes and messages
- `--as bot` uses a tenant access token (TAT), and requires the `im:message:send_as_bot` scope
