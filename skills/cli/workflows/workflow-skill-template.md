# Workflow Skill Template

The boilerplate `build-workflow-skill.md` fills in. Each generated skill is a **directory bundle**:

```
<workflow-name>/
├── SKILL.md           # — the slim entry point (procedure + scripts)
├── fingerprint.md     # — schema snapshot + drift handling (loaded only on error)
└── scripts/
    ├── <verb-1>.sh    # — semantic-flag bash, hardcoded Copera IDs
    ├── <verb-2>.sh
    └── …
```

Why two markdown files instead of one: schema metadata, frozen column lists, and the full drift-handling procedure are verbose **and rarely needed** — only when `copera` returns an error or the user asks for a manual refresh. Splitting them into `fingerprint.md` keeps the agent's working context small on every successful run.

---

## SKILL.md — copy and fill

````markdown
---
name: <kebab-case-name>
description: <one sentence — what the workflow does and when to invoke it. Include trigger phrases so agents auto-detect.>
---

# <Workflow Title>

<One-paragraph summary in plain English. End with the trigger phrase the user agreed on.>

## Setup (first-run / new teammate)

This workflow expects a Copera profile named **`<profile-name>`** in your `~/.copera.toml`. If you don't have it yet, add:

```toml
[profiles.<profile-name>]
token    = "cp_pat_…"     # your Personal Access Token
board_id = "<boardId>"
table_id = "<tableId>"
# add doc_id / channel_id here if the workflow uses them
```

If you already have a profile under a different name with the same token, either rename it to `<profile-name>` or set `COPERA_PROFILE=<your-name>` before invoking. Profile names must align across teammates for the bundled scripts to work without per-call overrides.

## Scripts

The procedure below invokes scripts in `./scripts/` rather than calling `copera` directly. Column IDs, option IDs, and payload shapes are hardcoded inside the scripts to eliminate LLM ID-hallucination.

| Script | Purpose | Required flags | Optional flags | Confirm |
|---|---|---|---|---|
| `scripts/<verb-1>.sh` | <one-line purpose> | `--<a>`, `--<b>` | `--<c>` | no |
| `scripts/<verb-2>.sh` | <…> | `--<x>` | | yes |
| `scripts/<verb-3>.sh` | <destructive op> | `--<id>` | | **yes (locked)** |

The **Confirm** column is the agent's runtime contract:

- `no` — agent invokes silently, reports the result. No restatement of the call.
- `yes` — agent restates the call in plain English and waits for an explicit "yes" before running.
- `yes (locked)` — destructive operation. Always confirms regardless of any session-level overrides.

The user picked these values at workflow-build time. The agent honors them as-is — do not re-ask whether to confirm; that decision is already made. The user can still override per-session by saying "ask me before each step" (escalate everything) or "skip confirmations for this run" (relax everything except locked).

All scripts pass through `copera --json` output on stdout, pass through copera exit codes and stderr unchanged, read auth via the standard CLI rules, and are executable.

## Inputs

What the user provides when invoking this workflow:

- `<placeholder name>` — `<what it is, where it comes from>`. Required / optional.

## Procedure

> **Confirmation contract:** each script in the **Scripts** table above carries a `Confirm` value (`no`, `yes`, or `yes (locked)`). Honor it without re-asking. If the user typed "ask me before each step" in this conversation, escalate every script to `yes` for the session. Locked entries (destructive ops) always confirm regardless.

1. <Plain-English step>
   ```bash
   bash scripts/<verb>.sh --<flag> "<value>" --<flag> "<value>"
   ```
   Capture: `<what you extract from the JSON output>`.

2. <Plain-English step — possibly conditional>
   If `<condition>`, run:
   ```bash
   bash scripts/<verb-2>.sh --<flag> "<value>"
   ```

<Add steps as needed. The procedure should NEVER include raw `copera ...` invocations — that's what scripts are for.>

## Output

What you tell the user when done — the format they agreed on. Often a one-line summary plus the URLs/IDs of created resources.

## On copera error or schema-refresh request

If a `bash scripts/<verb>.sh` call exits non-zero, **before retrying or surfacing the error**, load and follow [`./fingerprint.md`](./fingerprint.md). That file owns the schema snapshot, the saved fingerprint, the error-classification rules (drift vs auth vs rate-limit vs network), and the full update procedure.

If the user explicitly asks "refresh the schema for `<workflow-name>`", also load `fingerprint.md` and follow its manual-refresh path.

Do **not** load `fingerprint.md` on every invocation. It's only needed when something has gone wrong or the user asks.
````

---

## fingerprint.md — copy and fill

````markdown
# Schema fingerprint & drift handling — `<workflow-name>`

> **You only read this file when**:
> 1. A script in `./scripts/` exited non-zero and you need to decide whether the error is schema drift, OR
> 2. The user explicitly asked: *"refresh the schema for `<workflow-name>`"*.
>
> Do **not** load this file on successful runs. The user's PAT has a low rate-limit budget; pre-flight checks would burn it for nothing.

## Metadata

```yaml
profile: <copera-profile-name-or-omit>
schema_snapshot_date: <YYYY-MM-DD>
schema_fingerprints:
  <tableId-1>: <16-hex>
  <tableId-2>: <16-hex>
```

## Schema (frozen — diagnostic reference)

### Board: `<boardId>` — `<board name>`

#### Table: `<tableId>` — `<table name>`

| Column ID | Label | Type | Options / linked table |
|---|---|---|---|
| `<colId>` | `<label>` | `<TYPE>` | `<opt_1: Triage, opt_2: In Progress, opt_3: Done>` |
| `<colId>` | `<label>` | `LINK` | → table `<linked tableId>` (`<linked table name>`, captured from interview — not in schema) |
| … | | | |

<Repeat for each table the workflow touches.>

### Docs

- Parent doc: `<docId>` — `<doc title>`

### Drive

- Target folder: `<folderId>` — `<folder path>`

### Channels

- Notification channel: `<channelId>` — `<channel name>`

## Step 1 — Classify the error

Read the structured JSON copera printed to stderr (`{"error":…, "message":…, "suggestion":…, "transient":…}`; see `reference/errors-and-retries.md`).

**Schema-flavored** — proceed to step 2:

- `api_error` whose `message` mentions *invalid column*, *unknown columnId*, *invalid option*, *unknown optionId*.
- An unexpected HTTP 400 / 422 from a write call.
- `missing_id` from a script whose IDs are hardcoded (suggests the table or column was renamed/deleted).

**Not drift** — surface to the user and stop. Do NOT probe the schema:

- `auth_required` (exit 4) → ask the user to re-auth.
- `rate_limit` (exit 6, `transient: true`) → wait until `x-ratelimit-reset`, retry once.
- Network / 5xx with `transient: true` → retry once with backoff, then surface.
- Anything else without a schema-flavored signal — surface as-is.

## Step 2 — Ask the user

Stop the workflow. Tell the user, in plain English:

> The script `<scriptname>.sh` failed with `<error.message>`. This looks like the schema for `<table name>` may have changed. Want me to fetch the current schema and update this skill?

## Step 3 — Update on confirmation

If the user says yes:

1. Run `copera tables get --board <boardId> <tableId> --json` for the affected table.
2. Recompute the fingerprint:
   - Sort `columns[]` by `columnId`.
   - For each column, build `<columnId>:<type>` plus, if option-based, `:<sorted optionIds joined by ','>`.
   - Join with `\n`. Take `sha256` of the bytes; first 16 hex chars.
3. Compare to the value under **Metadata** → `schema_fingerprints.<tableId>` above.
4. Show the user a concrete diff (added/removed columns, renamed options, type changes).
5. **Update both files**:
   - This file: rewrite the relevant table in **Schema (frozen)** above and bump the entry in `schema_fingerprints` plus `schema_snapshot_date`.
   - Affected script(s) in `./scripts/`: update the `case` mappings that translate semantic flags to Copera option IDs.
6. Re-run the originally failing script. If it still fails, the issue is something else — surface that error.

If the user says no: exit cleanly. Do not retry against a schema you suspect is stale.

## Manual refresh

When the user asks "refresh the schema for `<workflow-name>`", run **Step 3** directly without an error trigger.

## Known limitations

- LINK column retargeting (a LINK column repointed to a different linked table) is **undetectable** by this procedure — the Copera schema does not carry the linked-table back-pointer. The linked table for each LINK column is captured during the original interview and documented in **Schema (frozen)** above; if the user changes the linked target, regenerate the workflow skill from scratch.
- New optional columns added by an admin are not flagged as drift (they don't break existing scripts). Capture them on the next manual refresh if the workflow needs them.
````

---

## Bash script — copy and fill

Per script, drop this into `scripts/<verb>.sh`. Replace placeholders with collected data. Make executable.

```bash
#!/usr/bin/env bash
# <verb>.sh — <one-line purpose>
#
# Usage:    <verb>.sh --<flag> <value> [--<optional> <value>]
# Required: --<flag-1>, --<flag-2>
# Optional: --<flag-3>
# Requires: `copera` CLI authenticated. COPERA_PROFILE optional.
# Output:   raw copera --json on stdout. Errors to stderr.
# confirm:  no                    # agent runtime contract — see SKILL.md Scripts table
                                  # values: no | yes | yes (locked)

set -euo pipefail

PROFILE="${COPERA_PROFILE:-<profile-name-or-default>}"
BOARD_ID="<boardId>"
TABLE_ID="<tableId>"

# --- argument parsing ---
arg_a=""
arg_b=""
arg_c=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --<flag-1>) arg_a="$2"; shift 2 ;;
    --<flag-2>) arg_b="$2"; shift 2 ;;
    --<flag-3>) arg_c="$2"; shift 2 ;;
    -h|--help)
      sed -n 's/^# \{0,1\}//p' "$0" | sed -n '2,8p'
      exit 0 ;;
    *) echo "unknown flag: $1" >&2; exit 2 ;;
  esac
done

[[ -z "$arg_a" ]] && { echo "--<flag-1> is required" >&2; exit 2; }
[[ -z "$arg_b" ]] && { echo "--<flag-2> is required" >&2; exit 2; }

# --- semantic → Copera ID translation ---
case "$arg_b" in
  <semantic-1>) opt_b="<opt_id_1>" ;;
  <semantic-2>) opt_b="<opt_id_2>" ;;
  *)  echo "invalid <flag-2>: $arg_b (allowed: <semantic-1>|<semantic-2>)" >&2; exit 2 ;;
esac

# --- escape user-provided strings for JSON ---
escape_json() { printf '%s' "$1" | sed -e 's/\\/\\\\/g' -e 's/"/\\"/g'; }
arg_a_esc="$(escape_json "$arg_a")"

# --- payload + call ---
cols='[
  {"columnId":"<col-1>","value":"'"$arg_a_esc"'"},
  {"columnId":"<col-2>","value":"'"$opt_b"'"}
]'
if [[ -n "$arg_c" ]]; then
  arg_c_esc="$(escape_json "$arg_c")"
  cols="${cols%]}, {\"columnId\":\"<col-3>\",\"value\":\"$arg_c_esc\"}]"
fi

COPERA_PROFILE="$PROFILE" copera rows create \
  --board "$BOARD_ID" --table "$TABLE_ID" \
  --data "{\"columns\":$cols}" \
  --json
```

---

## Filling tips

- **Hardcode IDs in scripts.** Column IDs and option IDs go into the scripts, not into agent prose. The whole point is to remove LLM choice from the ID layer.
- **Keep one script per verb.** Don't make `do-everything.sh` — make `triage.sh`, `notify.sh`, `archive.sh`. Easier to test, easier to edit on schema drift.
- **`profile:` is optional** in `fingerprint.md`. If set, scripts default `COPERA_PROFILE=<name>`. Use this when the workflow operates on a single board+table.
- **Fingerprints are per-table.** Use a map (`schema_fingerprints`) when the workflow touches multiple tables.
- **Don't use `jq`** unless you confirm it's available. Plain bash with parameter expansion is the default. For payloads beyond ~5 columns or nested arrays, fall back to a `python3` or `node` heredoc only after confirming availability.
- **LINK columns:** the linked table is captured during the interview and documented in `fingerprint.md`. Drift-check cannot detect LINK retargeting — flag this in the **Known limitations** section.
- **Confirmation is the agent's job, not the script's.** Scripts are pure CLI translation. The agent shows the planned invocation and waits for OK before running.
