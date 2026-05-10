# Docs

Markdown documents (workspace-wide, not per-board). Tree, search, read, write, delete.

**Auth caveat:** docs endpoints require a PAT (`cp_pat_…`). Integration keys (`cp_key_…`) return 401. See `auth-and-profiles.md`.

## Discover

```bash
copera docs tree --json                       # workspace root
copera docs tree --parent <docId> --json      # subtree under a doc
```

The tree is **fully hydrated** — children come back as complete `DocNode` objects (not stubs), so you don't need to recurse with separate calls. There is no `--depth` flag.

```bash
copera docs search "query terms" --json [--sort-by createdAt|updatedAt] [--sort-order asc|desc] [--limit 1-50]
```

Search returns hits with this shape:

```json
{
  "id": "66…",
  "title": "…",
  "highlight": { "title": "…", "mdBody": "matched snippet" },
  "parents": [
    { "_id": "66…", "title": "Engineering" },
    { "_id": "66…", "title": "Runbooks" }
  ],
  "createdAt": "1714838400000",
  "updatedAt": "1715184000000"
}
```

**Quirks to remember:**

- `parents` is `[]{ _id, title }` — objects, not bare ID strings.
- `createdAt` / `updatedAt` are **millisecond epoch strings**, not RFC 3339. Convert with `new Date(parseInt(value))`.
- The hit's most useful field is `highlight.mdBody`; fall back to `highlight.title` if it's empty.

## Read

```bash
copera docs get     [docId] --json   # metadata only
copera docs content [docId]          # markdown body, plain text on stdout
copera docs content [docId] --no-cache   # skip the local content cache
```

Both commands accept the doc ID as a positional arg, or fall back to `doc_id` from the active profile.

`docs content` caches the markdown body locally (see `errors-and-retries.md`). Use `--no-cache` after the user reports that an external edit landed.

## Update content

```bash
copera docs update <docId> [--operation replace|append|prepend] [--content "…"]
echo "## New section" | copera docs update <docId> --operation append
```

Default operation is `replace`. Content from `--content` or stdin (one of the two is required).

This is **async**: server returns 202, CLI prints `Content update queued (operation: …). Processing is async.`. The local cache is invalidated on success, so the next `docs content` will fetch fresh.

## Update metadata

```bash
copera docs metadata <docId> [--title "…"] [--icon "…"] [--cover "…"]
```

At least one of the three flags is required. `icon` and `cover` are workspace-specific values — ask the user for valid choices rather than guessing.

## Create

```bash
copera docs create --title "Required" [--parent <docId>] [--content "…"]
echo "# Body" | copera docs create --title "Required"
```

`--title` is mandatory. `--parent` places the doc under another doc; without it, the doc lands at the workspace root. Stdin is read for content **only when stdout is non-interactive** — interactive shells need `--content` explicitly.

## Delete

```bash
copera docs delete <docId> [--force]
```

Interactive terminals prompt for confirmation; non-interactive mode requires `--force` or the command refuses. Soft-delete only (recoverable workspace-side, but the CLI cannot un-delete). Only the document owner can delete.
