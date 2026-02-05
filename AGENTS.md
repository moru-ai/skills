# Contribution Guidelines

Guidelines for contributing to Moru agent skills.

## Code Style

- No emojis in markdown or code comments (use `Yes/No` in tables, not checkmarks)
- Keep examples concise and focused
- Use `# Bad:` and `# Good:` comment patterns for Python
- Use `// Bad:` and `// Good:` comment patterns for JavaScript/TypeScript

## File Structure

Each skill directory under `skills/` must contain:

1. **SKILL.md** (required) - Main skill file with YAML frontmatter
2. **LICENSE.txt** (required) - License terms
3. **references/** (optional) - On-demand documentation
4. **scripts/** (optional) - Executable helper scripts

## SKILL.md Format

```yaml
---
name: skill-name
description: "Comprehensive trigger description (100-300 words). Include: what it does, when to use it, specific keywords/triggers, file types if applicable."
---
```

### Description Field

The description is the primary trigger mechanism. It must:

- Explain WHAT the skill does
- List WHEN to use it (specific triggers, keywords, file types)
- Be comprehensive enough for Claude to match user intent
- Include synonyms and variations users might use

### Body Content

- Start with installation command if applicable
- Include Quick Start with minimal working example
- Add Quick Reference table for common tasks
- Use code examples over explanations
- End with Common Pitfalls section

## Naming Conventions

- Skill directories: `moru-*` prefix, kebab-case
- Scripts: snake_case (`helper_script.py`)
- References: kebab-case (`api-reference.md`)

## Adding New Skills

1. Create directory: `skills/moru-[name]/`
2. Add `SKILL.md` with proper frontmatter
3. Add `LICENSE.txt` (Apache 2.0 for open source)
4. Update `.claude-plugin/marketplace.json` if applicable
5. Test the skill before submitting
