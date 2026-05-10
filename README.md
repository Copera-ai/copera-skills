# Copera Skills

Official agent skills for the [Copera](https://copera.ai) platform. Teaches AI coding agents — Claude Code, Cursor, OpenCode, Codex, Windsurf, Cline, and others — how to drive a Copera workspace via the [`copera` CLI](https://cli.copera.ai).

This repository is a **Claude Code marketplace** and a **Cursor plugin** at the same time, and is also installable through the universal [`npx skills`](https://skills.sh) tool for every other agent.

## What's inside

```
skills/
└── cli/                    # the Copera CLI skill — works on any agent
    ├── SKILL.md            # router — "load reference/X.md when user asks Y"
    ├── reference/          # domain references, loaded on demand
    │   ├── auth-and-profiles.md
    │   ├── boards-tables.md
    │   ├── rows.md
    │   ├── docs.md
    │   ├── drive.md
    │   ├── comments-channels.md
    │   └── errors-and-retries.md
    └── workflows/          # how to generate per-user workflow skills
        ├── build-workflow-skill.md
        └── workflow-skill-template.md
```

Future skills (SDKs, API, integrations) will be added under `skills/<name>/`.

## Install

### Claude Code

```
/plugin marketplace add copera-ai/copera-skills
/plugin install copera@copera-ai
```

Invoke with `/copera:cli` or let it auto-trigger from your prompt.

### Cursor

```
/add-plugin copera-ai/copera-skills
```

Or browse the [Cursor marketplace](https://cursor.com/marketplace).

### Everyone else (OpenCode, Codex, Windsurf, Cline, Aider, …)

```
npx skills add copera-ai/copera-skills
```

`npx skills` auto-detects which agents are installed and writes the skill to each one's expected location.

## Authoring

The repository follows the [Agent Skills](https://skills.sh) convention: each skill is a directory under `skills/` containing `SKILL.md` (universal frontmatter: `name` + `description`) plus optional `reference/`, `scripts/`, and `assets/` subdirectories.

Frontmatter is intentionally minimal so the same `SKILL.md` works in every agent. Platform-specific manifests live in `.claude-plugin/` and `.cursor-plugin/`.

## License

MIT — see `LICENSE`.
