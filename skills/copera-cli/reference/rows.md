# Rows

CRUD on rows. Every write needs the column schema first — see `boards-tables.md`.

All `rows` subcommands require `--board <id>` and `--table <id>` (or those IDs set on the active profile). The row ID is a positional argument to `get`, `update`, `update-description`, `delete`, `comment`, and `comments`; `list` and `create` take no row positional.

## Read

```bash
copera rows list --board <B> --table <T> --json
# → array of row objects

copera rows get  --board <B> --table <T> <rowId> --json
# → single row with columns array
```

`rows get` (human output only) auto-resolves option IDs and LINK display strings against the cached table schema. JSON output is raw — agents work with raw IDs.

### Filter & sort

`rows list` accepts `--filter` (JSON) and `--sort` (`colA:asc,colB:desc`). Both are optional. The CLI passes them through as query params; the API validates.

```bash
copera rows list --board <B> --table <T> --json \
  --filter '{"match":"and","conditions":[
    {"column_id":"col_status","operator":"equals","value":["opt_done"]},
    {"column_id":"col_assignee","operator":"is_empty"}
  ]}' \
  --sort col_due:asc

# Load the filter from a file with @path
copera rows list --board <B> --table <T> --filter @./filter.json --json
```

Filter shape:

```json
{
  "match": "and",
  "conditions": [
    { "column_id": "<id>", "operator": "<op>", "value": <v> }
  ]
}
```

`match` is `"and"` (default) or `"or"`. Up to 20 conditions. **No nested groups**.

Operators by column type:

| Type   | Operators |
|--------|-----------|
| string | `equals`, `not_equals`, `contains`, `not_contains`, `starts_with`, `ends_with`, `is_empty`, `is_not_empty` |
| number | `equals`, `not_equals`, `gt`, `gte`, `lt`, `lte`, `includes`, `not_includes`, `is_empty`, `is_not_empty` |
| select | `equals`, `not_equals`, `includes`, `not_includes`, `is_empty`, `is_not_empty` |
| bool   | `equals`, `not_equals`, `is_empty`, `is_not_empty` |
| date   | `equals`, `before`, `after`, `between`, `today`, `yesterday`, `tomorrow`, `next_7_days`, `last_7_days`, `current_week`, `last_week`, `next_week`, `current_month`, `last_month`, `next_month`, `is_empty`, `is_not_empty` |

Value shape:

- String → `string`
- Number `gt`/`gte`/`lt`/`lte`/`equals`/`not_equals` → `number`; `includes`/`not_includes` → `number[]`
- Select → `string` or `string[]` of option IDs
- Date `equals`/`before`/`after` → ISO date string; `between` → `[startISO, endISO]`; relative operators (`today`, `current_week`, …) → omit `value`
- `is_empty` / `is_not_empty` → omit `value`

Sorting by a `PASSWORD` column returns `400`.

Row JSON shape:

```json
{
  "id": "66…",
  "rowId": 42,
  "owner": "user_…",
  "board": "66…",
  "table": "66…",
  "description": "markdown body",
  "columns": [
    { "columnId": "col_a4f2", "value": "opt_2",            "linkValue": null },
    { "columnId": "col_link", "value": ["66xy…", "66yz…"], "linkValue": ["Acme Inc.", "Globex"] }
  ],
  "updatedAt": "2026-05-09T12:34:56Z"
}
```

`rowId` is the user-visible sequential number (e.g. "Row #42"); `id` is the canonical 24-char ObjectId.

## Create

```bash
copera rows create --board <B> --table <T> --data '{
  "columns": [
    { "columnId": "col_a4f2",  "value": "opt_2" },
    { "columnId": "col_link",  "value": ["66abc…"] },
    { "columnId": "col_text",  "value": "Sample" }
  ]
}' --json
```

`--data` accepts the JSON inline. Alternatively, pipe via stdin:

```bash
echo '{"columns":[…]}' | copera rows create --board <B> --table <T> --json
```

The payload field is **`columnId`**, not `id` or `column_id`. Unsupported / unknown column IDs are silently dropped server-side; verify by re-reading the row after create.

## Update column values

```bash
copera rows update <rowId> --board <B> --table <T> --data '{
  "columns": [
    { "columnId": "col_a4f2", "value": "opt_3" }
  ]
}' --json
```

Same payload shape as `create`. Stdin works too.

Only the columns you include are touched — omitted columns retain their values.

## Update description (markdown body)

```bash
copera rows update-description <rowId> --board <B> --table <T> \
    [--operation replace|append|prepend] [--content "…"]
```

Content from `--content` or stdin. Default operation is `replace`.

This is **async**: the server returns 202, the CLI prints `Description update queued (operation: …). Processing is async.`. Re-read with `rows get` after a few seconds.

## Delete

```bash
copera rows delete <rowId> --board <B> --table <T> [--force]
```

Interactive terminals prompt `Delete row <id>? [y/N]:` unless `--force`. In non-interactive mode (`CI=true`, piped stdout, `--no-input`) the command refuses to run **without** `--force` — agents must pass `--force` explicitly *and only after* user confirmation.

## Authenticate (form-row login)

For tables that store user credentials (an identifier column + a password column):

```bash
copera rows authenticate --board <B> --table <T> \
  --identifier-column <colId> --identifier-value <value> \
  --password-column   <colId> --password-value   <value>
```

All four flags are required; missing any returns a usage error. Returns the matching row on success.

## Comments

See `comments-channels.md` for `rows comment` and `rows comments`.
