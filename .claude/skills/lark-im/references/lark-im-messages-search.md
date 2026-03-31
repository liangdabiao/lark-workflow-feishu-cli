# im +messages-search

> **Prerequisite:** Read [`../lark-shared/SKILL.md`](../../lark-shared/SKILL.md) first to understand authentication, global parameters, and safety rules.

Search Feishu messages across conversations. This shortcut automatically performs a multi-step workflow: search for message IDs, batch fetch message details, then enrich the results with chat context.

> **User identity only** (`--as user`). Bot identity is not supported.

This skill maps to the shortcut: `lark-cli im +messages-search` (internally calls `POST /open-apis/im/v1/messages/search` + `GET /open-apis/im/v1/messages/mget`, then batch-fetches chat context).

## Commands

```bash
# Search by keyword
lark-cli im +messages-search --query "project progress"

# Restrict search to a specific group chat
lark-cli im +messages-search --query "weekly report" --chat-id oc_xxx

# Filter by sender (comma-separated)
lark-cli im +messages-search --query "requirement" --sender ou_xxx,ou_yyy

# Filter by attachment type
lark-cli im +messages-search --query "report" --include-attachment-type file

# Filter by chat type (group / p2p)
lark-cli im +messages-search --query "progress" --chat-type group

# Filter by sender type (user / bot)
lark-cli im +messages-search --query "reminder" --sender-type bot

# Exclude bot senders
lark-cli im +messages-search --query "reminder" --exclude-sender-type bot

# Only messages that @me
lark-cli im +messages-search --query "announcement" --is-at-me

# Combined filters + time range
lark-cli im +messages-search --query "meeting" --sender ou_xxx --chat-type group --start "2026-03-13T00:00:00+08:00" --end "2026-03-20T23:59:59+08:00"

# Specific time range (ISO 8601)
lark-cli im +messages-search --query "release" --start "2026-03-01T00:00:00+08:00" --end "2026-03-10T00:00:00+08:00"

# Output format options
lark-cli im +messages-search --query "test" --format pretty
lark-cli im +messages-search --query "test" --format table
lark-cli im +messages-search --query "test" --format csv

# Pagination
lark-cli im +messages-search --query "test" --page-token <PAGE_TOKEN>

# Preview the request without executing it
lark-cli im +messages-search --query "test" --dry-run
```

## Parameters

| Parameter | Required | Description |
|------|------|------|
| `--query <text>` | No | Search keyword (may be empty when used with other filters) |
| `--chat-id <id>` | No | Restrict to chat IDs, comma-separated (`oc_xxx,oc_yyy`) |
| `--sender <ids>` | No | Sender open_ids, comma-separated (`ou_xxx`) |
| `--include-attachment-type <type>` | No | Attachment filter: `file` / `image` / `video` / `link` |
| `--chat-type <type>` | No | Chat type: `group` / `p2p` |
| `--sender-type <type>` | No | Sender type: `user` / `bot` |
| `--exclude-sender-type <type>` | No | Exclude messages from `user` or `bot` senders |
| `--is-at-me` | No | Only return messages that mention `@me` |
| `--start <time>` | No | Start time with local timezone offset required (e.g. `2026-03-24T00:00:00+08:00`) |
| `--end <time>` | No | End time with local timezone offset required (e.g. `2026-03-25T23:59:59+08:00`) |
| `--page-size <n>` | No | Page size (default 20, range 1-50) |
| `--page-token <token>` | No | Pagination token for the next page |
| `--format <fmt>` | No | Output format: `json` (default) / `pretty` / `table` / `ndjson` / `csv` |
| `--as <identity>` | No | Identity type (defaults to and only supports `user`) |
| `--dry-run` | No | Print the request only, do not execute it |

## Core Constraints

### 1. Provide at least one filter whenever possible

All parameters are optional, but you should usually provide at least one filter (`--query`, `--sender`, `--chat-id`, etc.). Otherwise the search scope may be too broad and return low-signal results.

### 2. Two-step orchestration is automatic

The shortcut automatically performs:

1. The **search API** returns matching `message_id` values
2. The **mget API** fetches full message content for those message IDs in batch

The user does not need to manage the orchestration manually.

### 3. Conversation context is enriched automatically

In JSON output, each message automatically includes conversation context:

| Field | Description |
|------|------|
| `chat_type` | Conversation type: `p2p` / `group` |
| `chat_name` | Group name (for groups) or the other participant's name (for p2p chats) |
| `chat_partner` | For p2p only: the other participant's `open_id` and `name` |

In pretty output, the `chat` column shows the chat name for groups, or `"p2p"` for direct messages.

Each message in JSON output contains:

| Field | Description |
|------|------|
| `message_id` | Message ID |
| `msg_type` | Message type: `text`, `image`, `file`, `interactive`, `post`, `audio`, `video`, `system`, etc. |
| `create_time` | Creation time |
| `sender` | Sender information (includes `name` for user senders) |
| `content` | Message content |
| `chat_id` | ID of the conversation the message belongs to |
| `deleted` | Whether the message has been recalled (`true` = recalled) |
| `updated` | Whether the message has been edited after sending |
| `mentions` | Array of @mentions in the message; each item contains `{id, key, name}`. Present only when the message contains @mentions |
| `thread_id` | Thread ID (`omt_xxx`) if the message has replies in a thread. Present only when replies exist |

### 4. Search results contain follow-up clues

In JSON output, each message includes `chat_id` and `thread_id` (when present). Use them with other shortcuts for deeper inspection:

```bash
# View the full message stream for the conversation that contains the search result
lark-cli im +chat-messages-list --chat-id <chat_id>

# View replies in the thread that contains the search result
lark-cli im +threads-messages-list --thread <thread_id>
```

## Resource Rendering

Search results reuse the same content formatter as other read commands. Image messages are rendered as placeholders such as `[Image: img_xxx]`; resource binaries are **not** downloaded automatically.

Use `im +messages-resources-download` if you need to fetch the underlying image or file bytes from a specific message.

## AI Usage Guidance

### Resolving chat_id from a chat name

When the user refers to a chat by name and you need its `chat_id` for the `--chat-id` filter, use [`+chat-search`](lark-im-chat-search.md) first:

```bash
# Step 1: Find the chat_id by name
lark-cli im +chat-search --query "<chat name keyword>" --format json

# Step 2: Use the chat_id to narrow down message search
lark-cli im +messages-search --query "keyword" --chat-id <chat_id>
```

**Do not use `im chats search` or `im chats list` — always use the `+chat-search` shortcut.**

## Work Summary / Report Generation

When the user asks you to summarize work, generate a weekly report, or compile activity from chat messages, you should **paginate through all available results** to get a complete picture. A single page is rarely enough for thorough summarization.

### Strategy

1. **Start with targeted filters** — use `--chat-id`, `--sender`, `--start`, `--end` to narrow the scope as much as possible before paginating.
2. **Fetch all pages** — after the first call, check the output for `has_more` and `page_token`. If `has_more` is true, immediately issue the next call with `--page-token <token>`. Repeat until `has_more` is false or the results are clearly sufficient.
3. **Accumulate before summarizing** — collect all pages of messages first, then analyze and summarize. Do not summarize after the first page alone — you will miss important context.
4. **Use `--format json`** — JSON output includes `has_more` and `page_token` fields needed for pagination. `pretty` and `table` formats omit these fields and are not suitable for pagination. Note: `pretty` is human-readable (per-message rows); `table` is a flat key-value dump of the response envelope and is not human-readable for message lists.

### Example: Weekly work summary from a project chat

```bash
# Page 1
lark-cli im +messages-search --query "" --chat-id oc_xxx --sender ou_me --start "2026-03-18T00:00:00+08:00" --end "2026-03-25T23:59:59+08:00" --page-size 50 --format json

# Page 2 (if has_more is true)
lark-cli im +messages-search --query "" --chat-id oc_xxx --sender ou_me --start "2026-03-18T00:00:00+08:00" --end "2026-03-25T23:59:59+08:00" --page-size 50 --page-token <token_from_page_1> --format json

# Continue until has_more is false, then summarize all collected messages.
```

### Key points

- **Always paginate exhaustively** for summary tasks. A single page of 20-50 messages is usually insufficient for a meaningful work summary.
- If the user does not specify a time range, default to the current week (Monday to today) for weekly reports, or ask for clarification.
- When summarizing, group messages by topic/thread rather than by chronological order for better readability.

## Common Errors and Troubleshooting

| Symptom | Root Cause | Solution |
|---------|---------|---------|
| Too few results | The time range is too narrow or the keyword is too specific | Expand the time range and try broader keywords |
| No results | Missing permission or no match | Confirm `search:message` is authorized and relax the filters |
| Permission denied | Search scope not authorized | Run `auth login --scope "search:message"` |

## References

- [lark-im](../SKILL.md) - all message-related commands
- [lark-im-threads-messages-list](lark-im-threads-messages-list.md) - inspect thread replies
- [lark-shared](../../lark-shared/SKILL.md) - authentication and global parameters
