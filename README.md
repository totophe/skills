# Agent Skills

A collection of [Agent Skills](https://agentskills.io) for AI coding agents. Each skill provides specialized knowledge and patterns that agents can load on demand.

## Available Skills

| Skill | Description |
|-------|-------------|
| [sequelize-7](sequelize-7/) | Sequelize v7 (alpha) — TypeScript-first ORM with decorator-based models, queries, associations, transactions |
| [umzug](umzug/) | Umzug v3 — database migration runner adapted for Sequelize v7 |
| [excel-reader](excel-reader/) | Excel file reader — list sheets, preview headers, extract rows and columns with streaming support |
| [astro-static-site](astro-static-site/) | Astro static site — scaffold and deploy sites with React, TypeScript, TailwindCSS on GitHub Pages |
| [skill-creation](skill-creation/) | Guide for creating and structuring Agent Skills following the open standard |

## Installation

### Using skills.sh (recommended)

Install all skills at once:
```bash
npx skills add totophe/skills
```

Or install individual skills:
```bash
npx skills add totophe/skills/sequelize-7
npx skills add totophe/skills/umzug
npx skills add totophe/skills/excel-reader
npx skills add totophe/skills/astro-static-site
npx skills add totophe/skills/skill-creation
```

### Using git clone

**Personal scope** (available in all your projects):

```bash
# Clone individual skills into your personal skills directory
git clone https://github.com/totophe/skills.git /tmp/totophe-skills
cp -r /tmp/totophe-skills/sequelize-7 ~/.claude/skills/sequelize-7
cp -r /tmp/totophe-skills/umzug ~/.claude/skills/umzug
```

**Project scope** (available only in a specific project):

```bash
# From your project root
mkdir -p .claude/skills
git clone https://github.com/totophe/skills.git /tmp/totophe-skills
cp -r /tmp/totophe-skills/sequelize-7 .claude/skills/sequelize-7
cp -r /tmp/totophe-skills/umzug .claude/skills/umzug
```

### Skill discovery

Skills are automatically discovered by compatible agents (Claude Code, Cursor, Copilot, Gemini CLI, and others). The agent reads each skill's `name` and `description` at startup, then loads the full instructions on demand when the task matches.

| Scope | Location | Applies to |
|-------|----------|------------|
| Personal | `~/.claude/skills/<skill-name>/` | All your projects |
| Project | `.claude/skills/<skill-name>/` | Current project only |

## How skills work

Each skill follows the [Agent Skills specification](https://agentskills.io/specification):

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
└── references/       # Optional: detailed docs loaded on demand
```

1. **Discovery** — The agent scans skill directories and loads only `name` + `description` from each `SKILL.md`
2. **Activation** — When a task matches a skill's description, the full `SKILL.md` body is loaded into context
3. **Execution** — The agent follows the instructions, loading reference files as needed

## License

MIT
