# Comments & Channels

Two messaging surfaces in Copera: row comments (scoped to a single row) and channel messages (broadcast to a channel).

## Row comments

### Post

```bash
copera rows comment <rowId> --board <B> --table <T> \
    [--visibility internal|external] [--content "…"]
echo "Triaged. Assigning to backend." | copera rows comment <rowId> --board <B> --table <T>
```

Content from `--content` or stdin. **HTML is supported** in the comment body — useful for `<a href>` links, but escape user input you didn't author.

`--visibility` defaults to `internal`:

- `internal` — visible only to workspace members.
- `external` — visible to external collaborators on the row (e.g. customers in a CRM table). Use sparingly; ask the user before posting external.

JSON return shape:

```json
{
  "id": "66…",
  "author": { "name": "…", "email": "…" },
  "visibility": "internal",
  "createdAt": "2026-05-09T12:34:56Z",
  "content": "<p>Triaged.</p>"
}
```

### List (paginated)

```bash
copera rows comments <rowId> --board <B> --table <T> --json \
    [--visibility all|internal|external] \
    [--after <cursor> | --before <cursor>]
```

`--visibility` omitted = all. JSON return:

```json
{
  "items": [ … ],
  "pageInfo": {
    "hasNextPage": true,
    "endCursor":   "opaque-string",
    "startCursor": "opaque-string"
  }
}
```

Use `pageInfo.endCursor` with `--after` to fetch the next page. Don't synthesize cursors.

### Download a comment attachment

```bash
copera rows comments attachments download <rowId> --board <B> --table <T> \
    --comment <commentId> --file <fileId> [-o ./out.pdf]
```

`--comment` and `--file` are both required. `-o`/`--dest` sets the output path (default: current dir + original filename). The download is authorized in the row, table, board, comment, and file context.

## Channels

`channels` covers discovery, lifecycle, participants, and messaging. Most subcommands resolve the channel from a positional arg, `--channel <id>`, or the active profile's `channel_id`.

### List

```bash
copera channels list --json \
    [--query <text>] [--type <type>] [--kind group|dm] \
    [--participant <userOrTeamId>] [--limit <1-200>] [--offset <n>]
```

`--query` matches channel name, description, or participant. `--kind` filters group vs. dm channels. `--limit`/`--offset` paginate.

### Create

```bash
copera channels create "launch-room"
copera channels create --name "launch-room" --description "Q3 launch coordination"
copera channels create "voice-room" --type voice --participant user1 --participant user2
```

Name comes from the positional arg or `--name` (1–200 chars). `--type` is `text` or `voice` (default `text`). `--participant` is repeatable. The token user becomes the creator and a participant.

### Archive / unarchive

```bash
copera channels archive   [channel-id] [--channel <id>] [--force]
copera channels unarchive [channel-id] [--channel <id>]
```

Both resolve the channel from the arg, `--channel`, or profile `channel_id`, and require channel owner/admin. `archive` is destructive-gated (`--force` in non-interactive mode). Archived channels can be restored with `unarchive`.

### Participants

```bash
copera channels participants add    --channel <id> --participant user1 [--participant user2 …]
copera channels participants remove <participant-user-id> --channel <id> [--force]
```

`add` takes a repeatable `--participant` (at least one; workspace users including hired AI agents). `remove` takes the participant user ID as a positional and is destructive-gated. Both require channel admin; `--channel` falls back to profile `channel_id`.

## Sending messages

### To a channel

```bash
copera channels message send "Deploy finished"            --channel <channelId>
echo "Build #1234 green"          | copera channels message send --channel <channelId>
copera channels message send                              --channel <channelId> < report.md
copera channels message send "Hello" --channel <channelId> --name "Release Bot"
```

Content from positional argument **or** stdin (whichever is supplied), max 10000 characters. Markdown is supported in the message body.

`--channel <id>` is required unless `channel_id` is set in the active profile.

`--name` overrides the sender display name for the message — useful when an agent posts on behalf of a named persona ("Triage Bot", "Release Bot"). Without it, the workspace user attached to the token is shown. `--name` applies to channel messages only.

JSON return on success:

```json
{ "sent": true, "channel": "<channelId>" }
```

There is no internal/external distinction at the channel level — channel ACLs control reach.

### As a direct message

`channels message send` can also DM a workspace user with `--user <userId>` instead of `--channel` — the two are mutually exclusive:

```bash
copera channels message send "Hello there" --user <userId>
```

The dedicated `channels dm send` does the same thing (DM by user ID, creating or reusing the DM channel):

```bash
copera channels dm send "Deploy done" --user <userId>
echo "Build green"  | copera channels dm send --user <userId>
```

`--user` is required; text comes from the positional arg or stdin, max 10000 characters. `--name` does not apply to DMs.

## Deciding between a row comment and a channel message

- **Row comment**: tied to a specific row; the people watching that row (or the customer on it) see it. Use for context that lives with the work.
- **Channel message**: broadcast; anyone in the channel sees it. Use for status updates, alerts, scheduled summaries.

When in doubt — ask the user.
