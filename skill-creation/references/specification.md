# Agent Skills Specification Reference

> Full specification from [agentskills.io/specification](https://agentskills.io/specification)

## SKILL.md Frontmatter Validation

### `name` Field

- 1–64 characters
- Unicode lowercase alphanumeric and hyphens only (`a-z`, `0-9`, `-`)
- Must not start or end with `-`
- Must not contain consecutive hyphens (`--`)
- Must match the parent directory name

```yaml
# Valid
name: pdf-processing
name: data-analysis
name: code-review

# Invalid
name: PDF-Processing    # uppercase
name: -pdf              # leading hyphen
name: pdf--processing   # consecutive hyphens
```

### `description` Field

- 1–1024 characters
- Must describe both what the skill does and when to use it
- Include specific keywords that help agents identify relevant tasks
- Write in third person (description is injected into system prompt)

### `license` Field

- Optional
- Keep short: license name or filename reference
- Example: `license: Proprietary. LICENSE.txt has complete terms`

### `compatibility` Field

- Optional, 1–500 characters if provided
- Only include if the skill has specific environment requirements

```yaml
compatibility: Designed for Claude Code (or similar products)
compatibility: Requires git, docker, jq, and access to the internet
```

### `metadata` Field

- Optional key-value map (string → string)
- Use reasonably unique key names to avoid conflicts
- Common keys: `author`, `version`

```yaml
metadata:
  author: example-org
  version: "1.0"
```

### `allowed-tools` Field

- Optional, experimental
- Space-delimited list of pre-approved tools

```yaml
allowed-tools: Bash(git:*) Bash(jq:*) Read
```

## Body Content Guidelines

No format restrictions. Recommended sections:

- Step-by-step instructions
- Examples of inputs and outputs
- Common edge cases

The agent loads the entire body once the skill triggers. Split longer content into referenced files.

## File References

Use relative paths from the skill root:

```markdown
See [the reference guide](references/REFERENCE.md) for details.
Run the extraction script: scripts/extract.py
```

Keep references one level deep from SKILL.md. Avoid deeply nested chains.

## Progressive Disclosure Patterns

### Pattern 1: High-Level Guide with References

```markdown
# PDF Processing

## Quick Start
Extract text with pdfplumber:
[code example]

## Advanced Features
- **Form filling**: See [FORMS.md](references/forms.md)
- **API reference**: See [REFERENCE.md](references/reference.md)
```

### Pattern 2: Domain-Specific Organization

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── references/
    ├── finance.md
    ├── sales.md
    ├── product.md
    └── marketing.md
```

### Pattern 3: Conditional Details

```markdown
# DOCX Processing

## Creating Documents
Use docx-js for new documents. See [DOCX-JS.md](references/docx-js.md).

## Editing Documents
For simple edits, modify the XML directly.
**For tracked changes**: See [REDLINING.md](references/redlining.md)
```

## Description Writing Best Practices

The description field has the highest impact on skill activation. Tested patterns:

| Technique | Activation Rate |
|-----------|----------------|
| Minimal description | ~20% |
| Optimized description with keywords | ~50% |
| Description with trigger examples | ~72–90% |

Include:
- What the skill does (capabilities)
- When to use it (trigger scenarios, keywords users might say)
- What packages/tools it relates to (e.g., `@sequelize/core`)

## Validation

Use the `skills-ref` library to validate:

```bash
npx skills-ref validate ./my-skill
```

Checks: frontmatter format, required fields, naming conventions, file organization.

## Skill Distribution

### Claude Code

Place skills in the `.claude/skills/` directory of your project or home directory. Claude Code discovers skills from nested `.claude/skills/` directories in monorepos.

### Claude.ai

Package as a `.skill` file (zip with `.skill` extension) and upload via Settings > Capabilities > Skills.

### Workspace Deployment

Admins can deploy skills workspace-wide with automatic updates and centralized management.
