# Building a Workflow Skill

This is a **meta-procedure**. Read it when the user asks the agent to "remember", "automate", or "always do" a Copera workflow — anything they expect to repeat. The output is a small bundle (SKILL.md + bash scripts) that captures their workspace structure and step-by-step procedure, dropped wherever their agent expects to find skills.

Do **not** invoke this for one-shot tasks. If the user says "create one bug for me", just do it. If they say "every time I get a Sentry alert, file a triage row" — that's a workflow.

## What you produce

A directory bundle with **two markdown files** and a scripts folder:

```
<name>/
├── SKILL.md            # slim entry point — procedure + scripts table
├── fingerprint.md      # schema snapshot + full drift-handling procedure
└── scripts/
    ├── <verb-1>.sh     # e.g. triage.sh — takes semantic flags, runs copera
    ├── <verb-2>.sh     # e.g. notify.sh
    └── …
```

`SKILL.md` is what the agent reads on **every** invocation — keep it small. It contains the procedure (calling the bundled scripts), the inputs, and a one-paragraph "on error" pointer that defers to `fingerprint.md`.

`fingerprint.md` contains the schema metadata, the frozen column lists, and the full drift-detection / classification / update procedure. The agent loads it **only** when a script call fails or the user asks for a manual schema refresh. This split keeps the agent's working context small on the 99% of runs where nothing has gone wrong.

`scripts/<verb>.sh` files instruct the agent to **invoke the bundled scripts**, not to type `copera` commands directly. Column IDs, option IDs, and JSON payload shapes are the #1 source of LLM mistakes — hardcoding them inside bash eliminates that whole class of error.

The bundle is portable, regenerable, committable, and readable by every agent that supports the Agent Skills standard.

## Why scripts

A workflow skill *could* embed raw `copera` commands inline. Three reasons not to:

1. **LLM ID hallucination.** Even with the schema in context, agents occasionally substitute a remembered-similar column/option ID. A bash script that hardcodes `col_a4f2` and accepts `--severity P1` (mapping to `opt_p1` internally) makes that impossible.
2. **Single source of truth on schema drift.** When the schema changes, you update the script once — not every reference scattered through the procedure prose.
3. **Testability.** The user can `bash scripts/triage.sh --severity P1 --title "test"` from their terminal to verify the workflow without going through the agent.

Scripts must:
- Use `#!/usr/bin/env bash` with `set -euo pipefail`.
- Accept semantic flags (`--severity`, `--title`, `--customer-name`), not Copera column IDs.
- Map semantic values to Copera IDs internally (a small case statement).
- Pass through `copera` exit codes and stderr so the agent can react to errors.
- Default to `--json` on read calls so the agent gets parseable output.
- Never embed the user's token. Token resolution happens in the CLI via env or config.

## Procedure (you, the agent, follow this)

### 1. Confirm intent

Ask:

> Should I save this as a reusable workflow skill, or do you want me to do it once now? A reusable skill means I generate a `SKILL.md` plus small bash scripts that any future agent session can load and execute identically.

If they want one-shot, exit this procedure and just do the task.

### 2. Interview

Capture in writing:

- **Name (kebab-case)** — `triage-bug`, `weekly-report`. Becomes the bundle's directory name.
- **Trigger** — phrases the user will say to invoke this.
- **Inputs** — what does the user provide? Free text? URL? File path? Row ID?
- **Steps in plain English** — let them describe the procedure. Don't interrupt to ask about IDs yet.
- **Output / confirmation** — what the agent shows when done.

Echo it back. Get explicit agreement before moving on.

### 3. Probe the workspace

For each Copera object the user named (board, table, doc, drive folder, channel), resolve the name to an ID. Run discovery from `reference/boards-tables.md`, `reference/docs.md`, `reference/drive.md`. Show the user:

> I see two tables matching "Bugs": `Bugs (Engineering)` (`66ab…`) and `Customer Bugs (CRM)` (`66cd…`). Which one do you mean?

For every table involved, run `copera tables get --board <B> <T> --json` once. Capture the **full column array** — IDs, types, option IDs, option labels — verbatim. You will need every value from this in the scripts. Do not abbreviate.

For LINK columns, **ask the user** which table is on the other end. The schema does not carry that.

### 4. Snapshot the schema (for `fingerprint.md`, not for pre-flight checks)

For each table touched, save a schema snapshot — you'll write it into `fingerprint.md` (see template), **not** into SKILL.md. Compute a fingerprint with this recipe, used only later when an error suggests the schema may have drifted, never on every run:

1. Sort `columns[]` by `columnId`.
2. For each column, build `<columnId>:<type>` plus, if option-based, `:<sorted optionIds joined by ',' >`.
3. Join with `\n`, sha256, take first 16 hex chars.

Per-table fingerprints go in `fingerprint.md`'s `schema_fingerprints` map. SKILL.md does not see this data — it's deferred so the agent only loads it when something goes wrong.

### 5. Decide where the bundle lives, pick the shared profile name, agree on the OS baseline

The recommendation is simple: **the Copera CLI skill (Tier A) belongs installed globally**, once per developer machine. **Workflow skills (the ones this procedure generates) belong project-local**, committed with the codebase so the team shares them. Default to project-local for any skill the user describes as something more than one person will run.

Ask:

> Where should I save this skill?
> 1. **Project-local** — committed with the codebase, shared with your team. **Recommended for any team-facing workflow.**
> 2. **User-global** — follows you across projects, not shared. Reserve this for personal-only workflows.
>
> Which agent are you using? I'll write to the project-local path:
>
> - Claude Code: `.claude/skills/<name>/`
> - Cursor:      `.cursor/skills/<name>/`
> - OpenCode:    `.opencode/skills/<name>/`
> - Codex:       `.agents/skills/<name>/`
> - Generic:     `.agents/skills/<name>/`

For user-global, swap to the `~/.<agent>/skills/<name>/` equivalent.

Then ask about the team's **operating-system baseline**:

> Will every teammate who runs this workflow be on the same OS family?
>
> - **All macOS / Linux / WSL / Git Bash on Windows** → bash scripts (default, recommended).
> - **All Windows-native (no WSL, no Git Bash)** → PowerShell scripts. I'll generate `.ps1` files instead of `.sh`.
> - **Mixed** → I'll generate **both** `.sh` and `.ps1` variants and the procedure will pick at runtime. (More files to maintain — the convention is to require Git Bash or WSL on Windows so we only ship one flavor.)

Default to bash-only with the convention that Windows teammates use Git Bash or WSL. Generate dual variants only if the user explicitly opts in.

Then ask about the **profile name**:

> If you're project-local-sharing this with teammates, every teammate needs a `[profiles.<name>]` entry with the same name in their `~/.copera.toml`. What name should we use? (Default: the workflow's kebab-case name, e.g. `triage-bug`.)

Why: the bundled scripts hardcode `COPERA_PROFILE="${COPERA_PROFILE:-<this-name>}"`. If your teammate's profile is called `work` instead of `triage-bug`, the CLI will look for the wrong profile and fail. There are two ways teams keep this aligned:

- **Convention** (recommended) — agree on a profile name (the workflow name is a good default), and every teammate adds `[profiles.<name>]` to their own `~/.copera.toml` with their own token + the same board/table IDs. The bundled SKILL.md will spell this out in a "Setup" section so new teammates know what to configure.
- **Override per call** — teammates set `COPERA_PROFILE=<their-name>` before invoking, or pass `--profile <name>` if the script supports it. Useful for users who don't want a second profile entry, but every invocation has to remember.

For most cases, default to convention. Capture the chosen profile name; you'll need it for the script's `${COPERA_PROFILE:-<name>}` default and for the **Setup** section in the emitted SKILL.md.

### 6. Write the scripts and decide confirmation per script

For every distinct `copera` write or schema-aware read in the procedure, write a script. Conventions:

- One script per **verb** the workflow performs (`triage.sh`, `notify.sh`, `archive.sh`).
- Required and optional flags are documented in a header comment block.
- The header comment includes a **`# confirm: yes | no`** line — this is the runtime contract for whether the agent restates the call before running.
- Map semantic values to Copera IDs in a `case` statement near the top of the file.
- Build the `--data` JSON inline; never let semantic input flow into JSON without translation.

**Default confirmation per script** — set during the build interview using these heuristics:

| Script touches | Default | User can override? |
|---|---|---|
| `rows delete`, `docs delete`, force-delete, mass-overwrite uploads | `confirm: yes` | **No** — destructive ops always confirm. |
| External visibility (`--visibility external` comments, customer-facing channels) | `confirm: yes` | Yes |
| Bulk operations (creating > 5 rows in one invocation, recursive directory upload to a populated folder) | `confirm: yes` | Yes |
| Internal `rows create` / `rows update` / `docs update` / `docs create` / internal `channels message send` | `confirm: no` | Yes (can escalate to confirm) |
| Read-only operations | `confirm: no` | n/a (reads don't confirm) |

After classifying, present the table to the user and ask: *"Confirm-default for these looks right? Anything you want to escalate or relax?"* Then bake the chosen `confirm:` value into each script's header AND surface it in the SKILL.md **Scripts** table (a column titled `Confirm`).

The agent that loads this workflow skill at runtime reads each script's `confirm:` value and either restates+waits or runs+reports accordingly. **Never re-asks per call when the build-time decision said `no`.** The user opted into that already.

Skeleton:

```bash
#!/usr/bin/env bash
# triage.sh — file a bug-triage row in Engineering > Bugs.
# Usage: triage.sh --title <text> --severity <P0|P1|P2|P3> [--customer <name>]
# Requires: copera CLI authenticated; COPERA_PROFILE optional (defaults to "triage").
set -euo pipefail

PROFILE="${COPERA_PROFILE:-triage}"
BOARD_ID="66ab…"
TABLE_ID="66bg…"

title=""
severity=""
customer=""

while [[ $# -gt 0 ]]; do
  case "$1" in
    --title)    title="$2"; shift 2 ;;
    --severity) severity="$2"; shift 2 ;;
    --customer) customer="$2"; shift 2 ;;
    *) echo "unknown flag: $1" >&2; exit 2 ;;
  esac
done

[[ -z "$title" ]]    && { echo "--title is required" >&2; exit 2; }
[[ -z "$severity" ]] && { echo "--severity is required" >&2; exit 2; }

case "$severity" in
  P0) sev_opt="opt_p0_id" ;;
  P1) sev_opt="opt_p1_id" ;;
  P2) sev_opt="opt_p2_id" ;;
  P3) sev_opt="opt_p3_id" ;;
  *)  echo "invalid severity: $severity (P0|P1|P2|P3)" >&2; exit 2 ;;
esac

# Build columns; conditionally include optional columns
cols='[
  {"columnId":"col_title","value":"'"${title//\"/\\\"}"'"},
  {"columnId":"col_severity","value":"'"$sev_opt"'"},
  {"columnId":"col_status","value":"opt_triage_id"}
]'

COPERA_PROFILE="$PROFILE" copera rows create \
  --board "$BOARD_ID" --table "$TABLE_ID" \
  --data "{\"columns\":$cols}" \
  --json
```

Make every script executable in the bundle:

```bash
chmod +x scripts/*.sh
```

**Avoid non-native tooling.** Don't use `jq`, `python`, or `node` inside generated scripts — these are not present on every developer machine, and a missing dependency is a worse failure mode than a slightly more verbose script. Use only the shell (bash or PowerShell) and the `copera` CLI itself. Build small JSON inline with parameter expansion (escape double quotes). For payloads with more than ~5 columns or nested arrays, write the payload across multiple variables in plain shell — still ugly is better than a `command not found: jq` for a teammate.

If the workflow genuinely requires structured JSON manipulation that bash can't handle cleanly, pause and reconsider: can you split it into two simpler scripts? Can you let `copera` handle the manipulation (e.g. read-then-update via separate calls)? Adding a tooling dependency should be a last resort and explicitly negotiated with the user.

Notes for the agent reading this:

- For inputs that may contain shell metacharacters or quotes, escape with `${var//\"/\\\"}` and pass via `--data` (not stdin) to keep arguments contained.
- Scripts should print the raw `copera --json` output to stdout. The agent parses it.
- Don't add `--quiet` inside scripts — let the agent decide.

### 7. Emit `SKILL.md` and `fingerprint.md`

Use `workflow-skill-template.md` as the boilerplate. The bundle contains **two** markdown files; do not collapse them into one.

`SKILL.md` (slim — read on every invocation):

- Frontmatter: minimal (`name` + `description` only), with trigger phrases in the description.
- **Scripts** section: list each script with its flag contract, mirroring the header comments.
- **Inputs** section: what the user provides at invocation time.
- **Procedure** section: numbered steps that invoke the bundled scripts. Include confirmation rules. **No raw `copera` commands in the procedure — only `bash scripts/<verb>.sh …`.**
- **Output** section: what to report back when done.
- **On copera error or schema-refresh request** section: a one-paragraph pointer that says "load and follow `./fingerprint.md`". Do not duplicate drift-handling logic here.

`fingerprint.md` (verbose — read only on script error or manual refresh):

- **Metadata** YAML block: `profile`, `schema_snapshot_date`, `schema_fingerprints` map.
- **Schema (frozen)** section: human-readable per-table column list with IDs, types, options, and LINK target tables (captured from the interview).
- **Step 1 — Classify the error**: schema-flavored vs auth/rate-limit/network, with concrete copera error signals for each.
- **Step 2 — Ask the user**: the exact prompt to show.
- **Step 3 — Update on confirmation**: the fetch + recompute fingerprint + diff + update both files + retry sequence. Critical: updates must edit **both** `fingerprint.md` AND the affected script's `case` mappings.
- **Manual refresh** section: how to handle "refresh the schema for `<name>`" without an error trigger.
- **Known limitations** section: LINK retargeting undetectable; new optional columns not flagged as drift.

If files at the target path already exist, **stop and ask** — don't overwrite without consent.

### 8. Dry-run

Run the procedure end-to-end against the user's actual workspace, with two rules:

- **Stop before every script call** that performs a write. Show the exact `bash scripts/<verb>.sh …` invocation. Get a yes before executing.
- **Halt on any unexpected output**. If a script returns a copera error, treat it as a real error — go through the drift-handling branch (below) before deciding whether the script itself is buggy.

After the dry-run, summarize what worked.

### 9. Iterate

If the dry-run revealed bugs in the scripts, ambiguous flag handling, or missing steps, edit the bundle and re-run. Stop when the user confirms the workflow does what they want.

## Reactive drift handling — owned by `fingerprint.md`

Workflow skills do **not** pre-check the schema before every run. Pre-checks burn rate-limit budget; the user's PAT has a low ceiling and most calls would never hit drift. Drift handling is reactive — it runs only when copera rejects something.

The full procedure (error classification, user prompt, fetch + diff + update, manual refresh path) lives in the generated `fingerprint.md` file, not in this builder document. The slim SKILL.md only contains a pointer telling the agent to load `fingerprint.md` on script failure or on a manual-refresh request.

You — the agent running this builder procedure — do not need to spell out drift handling in the generated SKILL.md. Just emit a brief "on error" section that says **load and follow `./fingerprint.md`**. The template (`workflow-skill-template.md`) shows the exact prose to use.

## Anti-patterns

- **Don't** put column or option labels in scripts. `Status` is a label; `col_a4f2` is the ID. Labels move; IDs are stable.
- **Don't** generate scripts that paper over schema gaps. If `tables get` shows a column as `LONG_TEXT` but the user says "it's a number", surface the conflict — don't coerce.
- **Don't** skip the **Drift handling** section, even if the user says "the schema never changes". It's how the skill stays correct over time.
- **Don't** embed the user's PAT, sensitive channel IDs, or other secrets in a script that lives in a public/committable repo. Flag this during step 5.
- **Don't** chain multiple unrelated workflows into one skill. One skill per workflow. Composability beats megaliths.
- **Don't** assume `jq`, `python`, or `node` are available. Default to plain shell. If a workflow seems to need them, restructure (split into two scripts, let `copera` do the manipulation) before adding a dependency.
- **Don't** make scripts call other scripts in the same bundle. Keep each script self-contained so it's individually testable.

## Platform notes

The default script form is bash. Works out of the box on macOS, Linux, WSL, and Git Bash on Windows. The OS-baseline question in step 5 decides whether you generate bash, PowerShell, or both:

- **bash only** (default, recommended): teammates on Windows install Git Bash or WSL. One file per verb; one set of patterns to maintain.
- **PowerShell only**: when the entire team is Windows-native. Adapt the bash patterns: `case` becomes `switch`, `set -euo pipefail` becomes `$ErrorActionPreference = 'Stop'`, JSON-string escaping uses `ConvertTo-Json -Compress` *only if* `pwsh` is the agreed-on baseline (it ships with PowerShell so it's safe).
- **Both**: emit `scripts/<verb>.sh` AND `scripts/<verb>.ps1`. The procedure section in SKILL.md should pick the right one based on detection (`if command -v bash` or check `$IsWindows`). This doubles maintenance — only do it when the user explicitly opts in.
