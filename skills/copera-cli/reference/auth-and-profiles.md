# Auth & Profiles

How the `copera` CLI resolves credentials and default IDs. Read this before any command — most "missing auth" or "wrong workspace" issues are config-resolution problems.

## Login modes

`copera auth login` has three sub-flows. Pick the one that matches the user's situation:

| Command | Behavior | When to use |
|---|---|---|
| `copera auth login` | Prints `<web>/oauth/cli`, opens it in the browser, then prompts for a masked paste-back. Requires a TTY. | Default for humans on a desktop. |
| `copera auth login --token` *(bare flag, no value)* | Skips the browser, drops straight into the masked paste prompt. | WSL/SSH/headless where `xdg-open` is unreliable but the user has a token in hand. |
| `copera auth login --token=cp_pat_…` | Saves the token verbatim. No browser, no paste prompt. | **Scripts, CI, agents.** This is the only mode that works in non-interactive terminals. |

In interactive terminals all three modes still ask for **profile name** and **save location**:

```
Profile name [default]: <enter>
Where should credentials be saved?
  1) ~/.copera.toml           (personal, all projects)
  2) .copera.local.toml       (this project only, add to .gitignore)
  3) .copera.toml             (this project, committed — no tokens!)
Choice [1]: <enter>
```

Choice 2 also offers to append `.copera.local.toml` to `.gitignore`.

## Status, whoami, logout

```
copera auth status  --json   # {profile, token_source, token, configured}
copera auth whoami  --json   # {profile, token, workspace, slug, name, email, user_id}  — verifies against the API
copera auth logout [--force] # removes token from the resolved file
```

`auth status` reads only local config; `auth whoami` actually calls the API and tells you which user + workspace the token resolves to. Use `whoami` when you need to confirm the user is on the workspace they think.

`whoami` human output leads with the user, then workspace/profile/token:

```
User:      Ada Lovelace
Email:     ada@example.com
User ID:   66…
Workspace: Acme (acme)
Profile:   default
Token:     cp_pat_…abcd
```

`auth logout` only works against tokens stored in a file. If the token came from `COPERA_CLI_AUTH_TOKEN` or `--token`, `logout` returns `no_stored_token`.

## Token resolution order

Highest precedence wins:

1. `COPERA_CLI_AUTH_TOKEN` env var
2. `--token` flag (root-level, applies to any subcommand)
3. `.copera.local.toml` in cwd
4. `.copera.toml` in cwd
5. `~/.copera.toml`

Config files 3–5 are **merged** (not first-wins): `~/.copera.toml` < `<cwd>/.copera.toml` < `<cwd>/.copera.local.toml`, so `.copera.local.toml` overrides the others key by key.

`auth status --json` includes a `token_source` field — use it to debug "wrong token" mysteries.

## Environment / base URLs

By default the CLI talks to production: API `https://api.copera.ai/public/v1`, web `https://app.copera.ai`.

Set `COPERA_SANDBOX=1` to swap both to dev hosts — API `https://api-dev.copera.ai/public/v1`, web `https://dev.copera.ai`. Use this to point a session at the dev workspace without editing config. (The swap only applies to base URLs left at their defaults.)

## Token types

| Prefix | Type | What it can do |
|---|---|---|
| `cp_pat_…` | Personal Access Token | Everything: boards, tables, rows, **docs**, drive, channels. |
| `cp_key_…` | Integration Key | Boards, tables, rows, drive, channels. **NOT docs** — docs endpoints return 401. |

If a `docs` command fails with auth on a `cp_key_` token, switch profiles or re-login with a PAT.

## Profiles

A profile bundles a token plus default IDs. Every profile entry under `[profiles.<name>]` in any of the config files looks like:

```toml
default_profile = "work"

[profiles.work]
token       = "cp_pat_…"
board_id    = "66abc123def456789012abcd"
table_id    = "66pqr012stu345678901vwxy"
row_id      = "…"
doc_id      = "…"
channel_id  = "…"
```

**Note on `row_id`:** it is a recognized profile key (loaded from config), but **no command currently consumes it** — `rows` commands take a positional `<row-id>` or `--row <n>` and ignore any profile `row_id`. Don't rely on it as a default.

**Active profile** resolution (highest wins):

1. `--profile <name>` flag
2. `COPERA_PROFILE` env var
3. `default_profile` config key
4. `"default"`

When a profile sets `board_id` / `table_id` / `doc_id` / `channel_id`, every command that needs those falls back to the profile value if no `--board` / `--table` / `--channel` flag (or positional arg, for docs) is supplied. Designing per-workflow profiles (`triage`, `releases`, `crm`) is the recommended way to make Copera workflows reproducible — see `workflows/build-workflow-skill.md`.

## Shared workflows: align profile names across teammates

Bundled workflow skills (see `workflows/build-workflow-skill.md`) embed `COPERA_PROFILE="${COPERA_PROFILE:-<name>}"` in their scripts. That default name is hardcoded at generation time. For a workflow committed and shared with a team, every teammate must have a `[profiles.<name>]` entry with the **same name** in their own `~/.copera.toml` for the script to resolve correctly without per-call overrides.

When generating a shared workflow skill:

- Pick a profile name everyone on the team will use (the workflow's kebab-case name is a good default — `triage-bug` workflow → `[profiles.triage-bug]`).
- The generated `SKILL.md` includes a **Setup** section telling teammates exactly what to add to their `~/.copera.toml`.
- Tokens stay per-teammate (each person uses their own PAT); only the profile *name* and the *board/table IDs* are shared.

Teammates who'd rather not add a second profile entry can override per call:

```bash
COPERA_PROFILE=their-existing-profile bash scripts/triage.sh --severity P1 --title "…"
```

— but the convention scales better. Document the expected name in your team's onboarding.

## Common mistakes

- Token in env var **and** in config file — env always wins. Users get confused when an old `COPERA_CLI_AUTH_TOKEN` shadows a fresh `auth login`.
- Running `auth login` over SSH or in a Codespace — the browser launch is silently dropped. Use `--token` (bare) and paste manually.
- Using a `cp_key_…` for docs commands — fails with 401. Switch to a PAT.
- Forgetting that `default_profile` lives at the top level of `~/.copera.toml`, not inside a `[profiles.*]` block.
- A teammate's profile name not matching what a shared workflow skill expects — see the **Shared workflows** section above.
