---
name: copera-cli
description: Drive a Copera workspace from the terminal via the `copera` CLI — boards, tables, rows, docs, drive files, channels, workspace members/teams, bookings, meeting notes, notifications, and global search. Load this skill when the user asks anything about Copera, copera CLI, their Copera boards/docs/drive, or wants to automate work in Copera.
---

# Copera CLI Skill

You are working with **Copera**, a unified workspace for boards (databases), docs (markdown), drive (files), channels (messaging), workspace members and teams, bookings and meeting notes, notifications, and global search. The `copera` CLI is the agent-friendly interface to a user's workspace.

## Before you start

1. **Verify the CLI is installed**: `copera version`. If missing, point the user at `https://developer.copera.ai/cli/overview#installation`.
2. **Verify auth**: `copera auth status --json`. If unauthenticated, ask the user to run `copera auth login`. Never paste tokens into prompts.
3. **Know who you are**: `copera auth whoami --json` returns the authenticated **user** (`name`, `email`, `user_id`) alongside the workspace (`name`, `slug`). Use it to confirm the token maps to the person and workspace the user expects before acting.
4. **Always use `--json`** when reading output programmatically. Human-formatted output is for the user, not for you.

## Global flags (available on every command)

| Flag | Effect |
|---|---|
| `--json` | Force JSON output. Always use this when parsing output yourself. |
| `--output auto\|json\|table\|plain` | Pick the output format explicitly (`auto` detects a TTY). |
| `--profile <name>` | Use a specific config profile (board/table/token defaults). |
| `--token <token>` | Override the token from config and `COPERA_CLI_AUTH_TOKEN`. |
| `--no-input` | Disable all interactive prompts. |
| `-q, --quiet` | Suppress informational messages. |
| `--verbose` | Show HTTP request/response details (debugging). |

**Non-interactive mode** is active when `--no-input` is passed **or** `CI=true` — in that mode the CLI never prompts, so pass every required value as a flag. Set **`COPERA_SANDBOX=1`** to point the CLI at Copera's dev/staging hosts instead of production.

## Routing — load the right reference file before acting

Copera is **unopinionated**: every workspace organizes boards/tables/docs/drive differently. You must discover the user's structure before acting on it. Each domain has its own reference doc with command sequences, JSON shapes, and common pitfalls. Load only what you need:

| When the user asks about… | Read first |
|---|---|
| Logging in, tokens, profiles, default IDs, `auth whoami` | `reference/auth-and-profiles.md` |
| Listing boards/tables, inspecting column schemas | `reference/boards-tables.md` |
| Creating or editing boards, tables, or columns; exporting a table view | `reference/boards-tables.md` |
| Creating, reading, updating, deleting, or commenting on rows | `reference/rows.md` |
| Row FILE attachments; rich-text / Description column cells (`column-content`) | `reference/rows.md` |
| Markdown docs (tree, search, create, update, delete) | `reference/docs.md` |
| Files and folders (tree, upload, download, search) | `reference/drive.md` |
| Channels, messaging, DMs, participants, row comments | `reference/comments-channels.md` |
| Workspace info, members, teams | `reference/workspace.md` |
| Bookings, booking types, scheduling, meeting notes | `reference/bookings-meetings.md` |
| Global search across the workspace; notifications | `reference/search-notifications.md` |
| Errors, exit codes, rate limits, retries | `reference/errors-and-retries.md` |
| Building a reusable workflow skill for the user | `workflows/build-workflow-skill.md` |

## Discovery-before-action rule

Most agent failures with Copera come from skipping discovery. Before creating or updating anything in a board you have not seen this session:

1. `copera boards list --json` → find the right `boardId`.
2. `copera tables list --board <boardId> --json` → find the right `tableId`.
3. `copera tables get --board <boardId> <tableId> --json` → **read the column schema**. Column types (TEXT, STATUS, DROPDOWN, LINK, NUMBER, DATE, etc.) determine how `--data` JSON must be shaped. Skipping this step is the #1 cause of silently-rejected row writes.

If the user has a profile configured (`COPERA_PROFILE` or `default_profile` in `~/.copera.toml`), `boardId` and `tableId` are often pre-resolved — check `reference/auth-and-profiles.md` first.

## Output and confirmation conventions

Use judgment. The user is the agent's principal — most of the time they've already told you what they want; don't make them say it again.

- **Read commands** (`boards list`, `rows get`, `docs content`, etc.): run them, summarize results in the user's preferred format.
- **Non-destructive write commands** (`rows create`, `rows update`, `docs update`, `docs create`, `drive upload`, internal `rows comment`, `channels message send` to internal channels): if the user's request is clear and unambiguous, **just run them and report what you did.** Don't restate the obvious. Confirm only when (a) you had to *infer* a value the user didn't specify, (b) the operation is reaching outside the workspace (external visibility, customer-facing channels), or (c) the user explicitly asked for confirm-each-step in this conversation.
- **Destructive commands** (`docs delete`, `rows delete`, mass uploads to populated folders, anything `--force`-required): **always confirm explicitly**, even when the user seems to have authorized in advance. State the exact resource(s), wait for an explicit "yes".

Workflow skills (see `workflows/build-workflow-skill.md`) override these defaults per-script via a `confirm:` flag set at build time. When invoking a workflow skill's bundled scripts, honor the flag; the user already decided the posture they wanted when the skill was generated.

## Building a custom workflow skill

If the user wants the agent to repeatedly perform a specific Copera workflow (triage, weekly report, customer onboarding, etc.), do not memorize their workspace structure in this conversation. Instead, load `workflows/build-workflow-skill.md` and walk them through generating a dedicated workflow skill that captures their boards, tables, columns, and procedure — and that knows how to detect schema drift later.

## What this skill does NOT do

- It does not store or transmit the user's API token.
- It does not modify the user's `~/.copera.toml` without asking.
- It does not assume column meanings — `Status`, `Priority`, `Owner` are conventions, not guarantees. Always read the schema.
