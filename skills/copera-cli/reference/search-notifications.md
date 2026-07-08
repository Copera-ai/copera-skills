# Search & Notifications

Cross-workspace global search plus the notification inbox.

## Global search

```bash
copera search "<query>" --json \
  [--type <entity>]... \
  [--sort createdAt|updatedAt] [--order asc|desc] \
  [--limit 1-100]
```

Searches **across entity types** in one call: documents, channels, channel messages, voice transcriptions, drive content, todos, todo items, AI chats, and AI chat messages.

- `--type` is repeatable — pass it once per type to narrow the search. Valid values: `document`, `channel`, `channelMessage`, `voiceTranscription`, `driveContent`, `todo`, `todoItem`, `aiChat`, `aiChatMessage`. Omit to search all types.
- `--sort` — `createdAt` or `updatedAt`.
- `--order` — `asc` or `desc`.
- `--limit` — 1–100.

Human output prints one line per hit (`ID / Type / Title / Updated`) followed by `Showing N of M`, so you can tell when results were truncated by `--limit`.

**When to use which search:**

- `copera search` — you don't know where the thing lives, or it spans entity types (a message *and* a doc *and* a drive file).
- `docs search` (see `docs.md`) — you know it's a document and want doc-specific fields (`highlight.mdBody`, `parents`).
- `drive search` (see `drive.md`) — you know it's a drive file/folder.

The scoped searches return richer, type-specific shapes; global search returns a uniform hit shape. Prefer the scoped command when you already know the entity type.

## Notifications

Notifications belong to the user behind the current PAT.

### List

```bash
copera notifications list --json [--after <id>] [--before <id>]
# → { "notifications": [...], "unreadCount": <n>, "count": <n> }
```

Pagination cursors are **notification IDs** (24-char ObjectId), not offsets:

- `--after <id>` — fetch notifications **older** than that ID.
- `--before <id>` — fetch notifications **newer** than that ID.

`unreadCount` is the inbox-wide unread total; `count` is the number in the returned page.

### Mark read / unread

```bash
copera notifications read   <notification-id> --json
copera notifications unread <notification-id> --json
```

Both take the notification ID as a required positional and flip its status.

### Delete

```bash
copera notifications delete <notification-id> [--force]
```

Destructive. Interactive terminals prompt for confirmation; non-interactive mode (`CI=true`, piped stdout, `--no-input`) refuses without `--force`. Agents must pass `--force` explicitly and only after user confirmation. See `errors-and-retries.md` for exit-code semantics.
