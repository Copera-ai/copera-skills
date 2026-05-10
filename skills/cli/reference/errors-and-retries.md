# Errors, Exit Codes, Retries, Caching

How to read CLI failures and handle async / rate-limited operations correctly.

## Structured errors

Every error is JSON on **stderr** (regardless of `--json` setting on stdout):

```json
{
  "error":      "auth_required",
  "message":    "no API token configured",
  "suggestion": "Run 'copera auth login' or set COPERA_CLI_AUTH_TOKEN",
  "transient":  false
}
```

| Field | Meaning |
|---|---|
| `error` | Machine-readable type code (e.g. `auth_required`, `missing_id`, `api_error`, `input_error`, `unknown_profile`, `invalid_flag`, `unsafe_url`, `confirmation_required`). |
| `message` | Human-readable detail. |
| `suggestion` | Often actionable — surface it to the user. |
| `transient` | If `true`, retry after a backoff. If `false`, the same call will fail again. |

## Exit codes

| Code | Meaning | Typical causes |
|---|---|---|
| 0 | success | |
| 1 | generic error | network failure, server 500, write errors |
| 2 | usage error | missing/invalid flags, missing IDs, bad JSON in `--data` |
| 3 | not found | unknown board/table/row/doc/file ID |
| 4 | auth failure | no token, expired token, wrong token type for endpoint |
| 5 | conflict | concurrent edit, version mismatch |
| 6 | rate limit | request quota exceeded — see below |

Agents should branch on exit code first, then on the `error` field for finer routing.

## Rate limits

Published per-endpoint quotas:

| Surface | Limit |
|---|---|
| Board reads (boards/tables/rows GET) | 50 req/min |
| Row create | 30 req/min |
| Channel messages | 100 req/min |
| Docs | not published — back off on 429 |

Headers exposed on every response (visible with `--verbose`):

- `x-ratelimit-limit`
- `x-ratelimit-remaining`
- `x-ratelimit-reset` — unix epoch seconds when the bucket refills

On exit code 6 (`error: "rate_limit"`, `transient: true`), wait until `x-ratelimit-reset` before retrying. For burst-heavy workflows (creating many rows), pace requests below 30/min instead of relying on retries.

## Async operations

These return 202 Accepted and process in the background:

- `copera docs update <docId>`
- `copera rows update-description <rowId>`

The CLI prints `… update queued (operation: …). Processing is async.` and exits 0. **There is no dedicated status endpoint.** To verify completion, re-read with `docs content` / `rows get` after a few seconds and compare. Don't claim the change landed until you've read it back.

## Retries

Retry on:

- Exit 1 with `transient: true` (network blips, 5xx).
- Exit 6 (rate limit) — wait for `x-ratelimit-reset`.

Do **not** retry on:

- Exit 2 (usage) — fix the input first.
- Exit 3 (not found) — the ID is wrong.
- Exit 4 (auth) — re-authenticate.
- Exit 5 (conflict) — re-read state, reconcile, then retry.

## Caching

The CLI caches:

- **Doc markdown content** (`docs content` reads). Invalidated automatically by `docs update` and `docs delete`. Bypass with `--no-cache`.
- **Table schemas** (used by `rows get` to resolve option IDs and LINK display strings). No automatic invalidation — workspace admins editing the schema requires `copera cache clean` for human-mode rendering. JSON output is unaffected (it returns raw IDs).

Manage:

```bash
copera cache status   # path + size
copera cache clean    # nuke all cached data
```

If a user just edited a column or option label and the agent is still showing the old name, run `cache clean`.
