# im +messages-send

> **Prerequisite:** Read [`../lark-shared/SKILL.md`](../../lark-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

Send a message to a group chat or a direct message conversation. Only supports bot identity.

This skill maps to the shortcut: `lark-cli im +messages-send` (internally calls `POST /open-apis/im/v1/messages`).

## Safety Constraints

Messages sent by this tool are visible to other people. Before calling it, you **must** confirm with the user:

1. The recipient (which person or which group)
2. The message content
3. The sending identity (bot only)

**Do not** send messages without explicit user approval.

When using `--as bot`, the message is sent in the app's name, so make sure the app has already been added to the target chat.

## Commands

```bash
# Send plain text (--text is recommended; it is wrapped into JSON automatically)
lark-cli im +messages-send --chat-id oc_xxx --text "Hello"

# Equivalent manual JSON
lark-cli im +messages-send --chat-id oc_xxx --content '{"text":"Hello"}'

# Send to a direct message (pass open_id)
lark-cli im +messages-send --user-id ou_xxx --text "Hello"

# Send a rich-text message
lark-cli im +messages-send --chat-id oc_xxx --msg-type post --content '{"zh_cn":{"title":"Title","content":[[{"tag":"text","text":"Body"}]]}}'

# Send a local image (uploaded automatically before sending)
lark-cli im +messages-send --chat-id oc_xxx --image ./photo.png

# Or send directly with an existing image_key
lark-cli im +messages-send --chat-id oc_xxx --image img_xxx

# Send a local file (uploaded automatically before sending)
lark-cli im +messages-send --chat-id oc_xxx --file ./report.pdf

# Send a video (--video-cover is required as the cover)
lark-cli im +messages-send --chat-id oc_xxx --video ./demo.mp4 --video-cover ./cover.png
lark-cli im +messages-send --chat-id oc_xxx --video ./demo.mp4 --video-cover img_xxx

# Send audio
lark-cli im +messages-send --chat-id oc_xxx --audio ./voice.opus

# Use an idempotency key (same key sends only once within 1 hour)
lark-cli im +messages-send --chat-id oc_xxx --text "Hello" --idempotency-key my-unique-id

# Preview the request without executing it
lark-cli im +messages-send --chat-id oc_xxx --text "Test" --dry-run
```

## Parameters

| Parameter | Required | Description |
|------|------|------|
| `--chat-id <id>` | One of two | Group chat ID (`oc_xxx`) |
| `--user-id <id>` | One of two | User open_id (`ou_xxx`) for direct messages |
| `--text <string>` | One of seven content options | Plain text message (automatically wrapped as `{"text":"..."}` JSON) |
| `--markdown <string>` | One of seven content options | Markdown text (auto-wrapped as post format with style optimization; image URLs auto-resolved) |
| `--content <json>` | One of seven content options | Message content JSON string; format depends on `msg_type` |
| `--image <path\|key>` | One of seven content options | Local image path or `image_key` (`img_xxx`). Local paths are uploaded automatically |
| `--file <path\|key>` | One of seven content options | Local file path or `file_key` (`file_xxx`). Local paths are uploaded automatically |
| `--video <path\|key>` | One of seven content options | Local video path or `file_key`. Local paths are uploaded automatically. **Must be paired with `--video-cover`** |
| `--video-cover <path\|key>` | **Required with `--video`** | Video cover image path or `image_key` (`img_xxx`). Local paths are uploaded automatically |
| `--audio <path\|key>` | One of seven content options | Local audio path or `file_key`. Local paths are uploaded automatically |
| `--msg-type <type>` | No | Message type (default `text`): `text`, `post`, `image`, `file`, `audio`, `media`, `interactive`, `share_chat`, `share_user`. Automatically set when using `--text`/`--image`/`--file`/`--video`/`--audio` |
| `--idempotency-key <key>` | No | Idempotency key; the same key sends only one message within 1 hour |
| `--as <identity>` | No | Identity type: `bot` only |
| `--dry-run` | No | Print the request only, do not execute it |

> **Mutual exclusivity rule:** `--text`, `--markdown`, `--content`, and `--image`/`--file`/`--video`/`--audio` cannot be used together. Media flags are also mutually exclusive with each other.
>
> **Video cover rule:** `--video` **must** be accompanied by `--video-cover`. Omitting `--video-cover` when using `--video` will fail validation. `--video-cover` cannot be used without `--video`.

## `content` Format Reference

| `msg_type` | Example `content` |
|----------|-------------|
| `text` | `{"text":"Hello <at user_id=\"ou_xxx\">name</at>"}` |
| `post` | `{"zh_cn":{"title":"Title","content":[[{"tag":"text","text":"Body"}]]}}` |
| `image` | `{"image_key":"img_xxx"}` |
| `file` | `{"file_key":"file_xxx"}` |
| `audio` | `{"file_key":"file_xxx"}` |
| `media` | `{"file_key":"file_xxx","image_key":"img_xxx"}` (video; `image_key` is the cover from `--video-cover` — **required**) |
| `share_chat` | `{"chat_id":"oc_xxx"}` |
| `share_user` | `{"user_id":"ou_xxx"}` |
| `interactive` | Card JSON (see Feishu interactive card documentation) |

## Return Value

```json
{
  "message_id": "om_xxx",
  "chat_id": "oc_xxx",
  "create_time": "1234567890"
}
```

## @Mention Format (text / post)

- @specific user: `<at user_id="ou_xxx">name</at>`
- @all: `<at user_id="all"></at>`

## Notes

- `--chat-id` and `--user-id` are mutually exclusive; you must provide exactly one
- `--content` must be a valid JSON string
- `--image`/`--file`/`--video`/`--audio` support local file paths; use relative paths within the current working directory. The shortcut automatically uploads the file first and then sends the message. You do not need to call a separate upload command manually
- If the provided value starts with `img_` or `file_`, it is treated as an existing key and used directly
- When using `--video`, `--video-cover` is **required** as the video cover (`image_key`). Omitting `--video-cover` with `--video` will produce a validation error. `--video-cover` cannot be used without `--video`
- Failures return an error code and message
- `--as bot` uses a tenant access token (TAT) and requires the `im:message:send_as_bot` scope
- When sending as a bot, the app must already be in the target group or already have a direct-message relationship with the target user
