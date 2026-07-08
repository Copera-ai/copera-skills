# Boards & Tables

Board and table discovery — the prerequisite for any row write. **Always run `tables get` and read the column schema before calling `rows create` or `rows update`.** Skipping that step is the #1 cause of silently-rejected writes.

## Discovery sequence

```bash
copera boards list --json
# → [{ "id": "66…", "name": "…", "description": "…", "createdAt": "…", "updatedAt": "…" }, …]

copera boards get <boardId> --json
# → single board object

copera tables list --board <boardId> --json
# → [{ "id": "66…", "name": "…", "columns": [...], "updatedAt": "…" }, …]

copera tables get --board <boardId> <tableId> --json
# → table object with full column definitions
```

`boards` has `bases` as an alias — use whichever the user prefers. Both subcommands accept `--json`.

`tables list` and `tables get` require `--board <id>` unless `board_id` is set in the active profile. `tables get` takes the table ID as a positional arg (no `--table` flag here).

Both list commands accept `--query <text>` for server-side search: `boards list --query <text>` matches board name or description; `tables list --query <text>` matches table name.

## Reading column schemas

`tables get --json` returns each column with this shape:

```json
{
  "columnId": "col_a4f2",
  "label": "Status",
  "type": "STATUS",
  "options": [
    { "optionId": "opt_1", "label": "Triage" },
    { "optionId": "opt_2", "label": "In Progress" },
    { "optionId": "opt_3", "label": "Done" }
  ]
}
```

The `columnId` is what you put in row `--data` payloads. The `label` is what humans see in the UI — never use it as an identifier.

## Column types

The types the CLI can **create** (`tables columns add --type`, UPPERCASE, validated) are exactly:

```
TEXT PARAGRAPH NUMBER MONEY DATE EMAIL PHONE WEBSITE CHECKBOX
STATUS DROPDOWN LABELS USERS LOCATION DESCRIPTION FILE
```

| Type | Stored value shape | Notes |
|---|---|---|
| `TEXT`, `PARAGRAPH` | string | |
| `NUMBER`, `MONEY` | number | |
| `DATE` | ISO 8601 string | |
| `EMAIL`, `PHONE`, `WEBSITE` | string | |
| `CHECKBOX` | boolean | |
| `STATUS`, `DROPDOWN` | option ID (string) | Resolve via `options[].optionId`. Provide options at create time with `--option`. |
| `LABELS` | array of option IDs | Multi-select. Provide options with `--option`. |
| `USERS` | array of user IDs | |
| `LOCATION` | string | |
| `DESCRIPTION` | rich-text markdown | Read/write via `rows column-content` / `rows update-column-content` (see `rows.md`). |
| `FILE` | array of file metadata objects | Attach via `rows attachments` (see `rows.md`). |

Only `STATUS`, `DROPDOWN`, and `LABELS` take `--option` / `--add-option`.

**Read schemas may surface additional/legacy types that are not creatable via the CLI** (e.g. `LINK`, see below). If `tables get --json` shows a type not in the list above, treat it as read-only from the CLI's create path and inspect the schema directly — don't guess.

## LINK columns

LINK columns reference rows in another table. **The schema does not carry a back-pointer to the linked table** — you must already know what's on the other end (or ask the user).

When reading rows, LINK columns return both:

- `value` — array of foreign row IDs (canonical).
- `linkValue` — array of display strings (index-matched with `value`, populated by the server).

When writing rows, set `value` to an array of foreign row IDs. The server resolves display strings on read; you do not send `linkValue`.

## Creating & editing boards

```bash
copera boards create --name "CRM" [--description "Sales pipeline"] [--table-name "Leads"]
# → board object; a board is always created with one initial table (name from --table-name, default "Table").
#   Grab the initial table id from the output before creating rows.

copera boards update <board-id> [--name "…"] [--description "…"] [--icon "🚀"]
# Only the flags you pass are sent; at least one is required.

copera boards delete <board-id> [--force]
# Moves the board to trash (destructive). Interactive terminals prompt unless --force;
# non-interactive mode (CI/piped/--no-input) refuses without --force.
```

`--name` is required on `create`. `update` sends only changed flags.

### Board participants

```bash
copera boards participants add    --board <id> --participant <id> [--participant <id> …]
copera boards participants remove <participant-id> --board <id> [--force]
```

`--participant` is repeatable (at least one required) and accepts workspace user, team, or AI agent IDs. `--board` resolves from the arg/flag or the active profile. `remove` takes the participant ID as a positional and is destructive-gated (`--force` in non-interactive mode).

## Creating & editing tables

```bash
copera tables create --board <id> --name "Tasks"
# → table object.

copera tables update <table-id> --board <id> --name "New name"
# Rename only. --name is required.

copera tables delete <table-id> --board <id> [--force]
# Destructive; --force to skip the confirmation prompt (required in non-interactive mode).
```

`--name` is required on both `create` and `update`.

## Managing columns

```bash
copera tables columns add --board <id> --table <id> --label "Priority" --type STATUS \
    --option "Low:green" --option "High:red"

copera tables columns update <column-id> --board <id> --table <id> [--label "Stage"] [--add-option "Blocked:red" …]
```

- `add`: `--label` and `--type` are required (see the type list above). `--option` is repeatable; format `"Label"` or `"Label:color"`; only meaningful for STATUS/DROPDOWN/LABELS.
- `update`: rename with `--label` and/or append choices with `--add-option` (repeatable). **`--add-option` is append-only** — it cannot rename or remove existing options. At least one of `--label` / `--add-option` is required.

## Exporting a table view

```bash
copera tables export <table-id> --board <id> --view <view-id> --format CSV --output-file report.csv
copera tables export <table-id> --board <id> --view <view-id> --format JSON -o -   > rows.json
copera tables export <table-id> --board <id> --view <view-id> --format PDF --save-to-drive
```

`--view <view-id>` is **required** (boards can have multiple views). `--format` is one of `CSV|XLSX|JSON|MARKDOWN|HTML|PDF|ZIP|ICS`.

| Flag | Effect |
|---|---|
| `-o`, `--output-file <path>` | Write the payload here. Default is stdout; `-` also means stdout. |
| `--column <id>` | Restrict to these column IDs (repeatable). |
| `--row <id>` | Restrict to these row IDs (repeatable). |
| `--include-hidden` | Include hidden columns. |
| `--include-system` | Include system columns. |
| `--save-to-drive` | Save the rendered file to drive instead of returning it inline. |
| `--force-async` | Force the server to use the async pipeline. |

Text/spreadsheet formats (CSV, XLSX, JSON, MARKDOWN, HTML, ICS) are returned inline and written to `--output-file`/stdout. **PDF, ZIP, and large renders** are queued through the async pipeline; instead of a payload the command prints a job descriptor (Job / Status / URL) — poll or fetch the URL when ready.

## Per-board ID requirement

Every `tables` and `rows` subcommand needs `--board <id>` (or `board_id` in the active profile). Tables are namespaced by board; the same table ID under a different board does not exist.

## Caching

The CLI caches table schemas locally (see `errors-and-retries.md` for cache invalidation). After a workspace admin changes a column or option, users may need `copera cache clean` to see fresh labels. For agent code, prefer `--no-cache` or `cache clean` between schema-aware operations if the user has just edited the schema.
