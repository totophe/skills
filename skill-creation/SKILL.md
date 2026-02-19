---
name: skill-creation
description: "Guide for creating and structuring Agent Skills following the open standard specification (agentskills.io). Use when creating a new skill, updating an existing skill, packaging skills, writing SKILL.md files, or structuring skill directories with references, scripts, and assets."
metadata:
  version: "1.0.0"
  author: totophe
---

# Skill Creation Guide

## Skill Directory Structure

```
skill-name/
├── SKILL.md          # Required: frontmatter + instructions
├── scripts/          # Optional: executable code (Python, Bash, JS)
├── references/       # Optional: documentation loaded on demand
└── assets/           # Optional: templates, images, data files for output
```

## SKILL.md Format

Two parts: YAML frontmatter (metadata) + Markdown body (instructions).

```yaml
---
name: my-skill
description: "What it does and when to use it. Include trigger keywords."
license: MIT
metadata:
  version: "1.0.0"
  author: your-name
---

# My Skill

Step-by-step instructions here...
```

### Frontmatter Fields

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Max 64 chars. Lowercase, numbers, hyphens only. Must match folder name. |
| `description` | Yes | Max 1024 chars. What it does AND when to use it. |
| `license` | No | License name or reference to bundled LICENSE file. |
| `compatibility` | No | Max 500 chars. Environment requirements (tools, packages, network). |
| `metadata` | No | Key-value pairs (author, version, etc.). |
| `allowed-tools` | No | Space-delimited pre-approved tools. Experimental. |

### Name Rules

- Lowercase letters, numbers, hyphens only
- No leading/trailing/consecutive hyphens
- Must match the parent directory name

### Description Is the Trigger

The `description` is the **primary mechanism** Claude uses to decide when to activate the skill. All "when to use" information must be here — not in the body (which only loads after triggering).

Good:
```yaml
description: "Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction."
```

Bad:
```yaml
description: "Helps with PDFs."
```

## Core Principles

### 1. Claude Is Already Smart

Only add context Claude doesn't already have. Challenge each line: "Does Claude need this?" and "Does this justify its token cost?" Prefer concise examples over verbose explanations.

### 2. Progressive Disclosure

Skills load in three stages:

1. **Metadata** (~100 tokens) — `name` + `description`, always in context for all skills
2. **SKILL.md body** (<5,000 tokens recommended) — loaded when the skill triggers
3. **Bundled resources** (unlimited) — loaded only when Claude needs them

Keep SKILL.md body **under 500 lines**. Move detailed docs to `references/`.

### 3. Match Freedom to Fragility

| Level | When | Example |
|-------|------|---------|
| High freedom (text instructions) | Multiple valid approaches | "Generate a REST API endpoint" |
| Medium freedom (pseudocode/parameterized) | Preferred pattern exists | "Follow this template but adapt to context" |
| Low freedom (specific scripts) | Fragile, consistency critical | "Run this exact script with these params" |

## Bundled Resources

### `scripts/` — Executable Code

Use when deterministic reliability is needed or the same code gets rewritten repeatedly. Token efficient — may execute without loading into context.

### `references/` — Documentation

Keeps SKILL.md lean. Claude loads these on demand. For files >10k words, include grep patterns in SKILL.md. Structure longer files with a table of contents. Keep references one level deep from SKILL.md.

### `assets/` — Output Files

Templates, images, boilerplate code, fonts. Not loaded into context — used directly in output.

### What NOT to Include

- README.md, INSTALLATION_GUIDE.md, QUICK_REFERENCE.md, CHANGELOG.md
- User-facing documentation
- Setup/testing procedures
- Process documentation about how the skill was created

The skill should only contain information an AI agent needs to do the job.

## Creation Process

1. **Clarify scope** — Understand what the skill should do with concrete examples
2. **Plan resources** — Identify what scripts, references, and assets would help
3. **Create the directory** — `mkdir -p skill-name/references`
4. **Write resources first** — Implement `scripts/`, `references/`, `assets/` files
5. **Write SKILL.md** — Frontmatter with strong description, then concise body instructions
6. **Iterate** — Use the skill on real tasks, notice struggles, refine

## Detailed Reference

See [specification.md](references/specification.md) for the full Agent Skills specification and validation rules.
