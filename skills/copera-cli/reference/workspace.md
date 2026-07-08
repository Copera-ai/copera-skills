# Workspace

Workspace identity, the authenticated user, members, and teams. Read-only except `team create`.

**Auth caveat:** these endpoints resolve against the token's workspace and user, so a PAT (`cp_pat_…`) is expected. See `auth-and-profiles.md`.

## Workspace info

```bash
copera workspace info --json
# → { "ws": { … } }
```

Human output: `ID / Name / Slug / Description / Seats / Created / Updated`. No flags beyond the globals.

## The authenticated user

```bash
copera workspace me --json
# → { "_id": "66…", "name": "…", "email": "…", "title": "…", "type": "…", "status": "…", … }
```

Human output: `ID / Name / Email / Title / Type / Status`. This is the same identity `auth whoami` surfaces — use `auth whoami` for a quick user + workspace check, `workspace me` when you want the full user object.

## Members

```bash
copera workspace members --json [--query <text>] [--limit 1-500] [--offset <n>]
```

Paginated list of workspace members. `--query` filters by name or email. `--limit` caps the page (1–500); `--offset` advances into the result set for classic offset pagination.

## Teams

```bash
copera workspace teams --json [--query <text>] [--limit 1-200] [--offset <n>]
```

Paginated list of teams. `--query` filters by team name. `--limit` is 1–200; `--offset` pages forward.

## Create a team

```bash
copera workspace team create --name "Engineering" --json
copera workspace team create --name "Design" --participant <userId> --participant <userId> --json
```

`--name` is required and must be unique in the workspace. `--participant` is repeatable — pass one flag per participant user ID (24-char hex) to seed the team's membership. Omit it to create an empty team.

Get member user IDs from `workspace members --json` first.
