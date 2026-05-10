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

## Column types you will encounter

| Type | Stored value shape | Notes |
|---|---|---|
| `TEXT`, `LONG_TEXT` | string | |
| `NUMBER` | number | |
| `DATE` | ISO 8601 string | |
| `STATUS`, `DROPDOWN` | option ID (string) | Resolve via `options[].optionId`. |
| `MULTI_DROPDOWN` | array of option IDs | |
| `CHECKBOX` | boolean | |
| `LINK` | array of foreign row IDs | See below. |
| `PERSON` | array of user IDs | |
| `URL`, `EMAIL`, `PHONE` | string | |
| `FILE` | array of file metadata objects | |

If the user is on an exotic column type not listed here, dump the schema with `tables get --json` and inspect it directly — don't guess.

## LINK columns

LINK columns reference rows in another table. **The schema does not carry a back-pointer to the linked table** — you must already know what's on the other end (or ask the user).

When reading rows, LINK columns return both:

- `value` — array of foreign row IDs (canonical).
- `linkValue` — array of display strings (index-matched with `value`, populated by the server).

When writing rows, set `value` to an array of foreign row IDs. The server resolves display strings on read; you do not send `linkValue`.

## Per-board ID requirement

Every `tables` and `rows` subcommand needs `--board <id>` (or `board_id` in the active profile). Tables are namespaced by board; the same table ID under a different board does not exist.

## Caching

The CLI caches table schemas locally (see `errors-and-retries.md` for cache invalidation). After a workspace admin changes a column or option, users may need `copera cache clean` to see fresh labels. For agent code, prefer `--no-cache` or `cache clean` between schema-aware operations if the user has just edited the schema.
