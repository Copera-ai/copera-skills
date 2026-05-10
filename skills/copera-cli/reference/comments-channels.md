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

## Channel messages

```bash
copera channels message send "Deploy finished"            --channel <channelId>
echo "Build #1234 green"          | copera channels message send --channel <channelId>
copera channels message send                              --channel <channelId> < report.md
copera channels message send "Hello" --channel <channelId> --name "Release Bot"
```

Content from positional argument **or** stdin (whichever is supplied). Markdown is supported in the message body.

`--channel <id>` is required unless `channel_id` is set in the active profile.

`--name` overrides the sender display name for the message — useful when an agent posts on behalf of a named persona ("Triage Bot", "Release Bot"). Without it, the workspace user attached to the token is shown.

JSON return on success:

```json
{ "sent": true, "channel": "<channelId>" }
```

There is no internal/external distinction at the channel level — channel ACLs control reach.

## Deciding between a row comment and a channel message

- **Row comment**: tied to a specific row; the people watching that row (or the customer on it) see it. Use for context that lives with the work.
- **Channel message**: broadcast; anyone in the channel sees it. Use for status updates, alerts, scheduled summaries.

When in doubt — ask the user.
