# Custom Skills & Agents

This repository contains custom Claude Code skills and agents.

## Repository Structure

- `.claude/skills/` — Project-level skills auto-loaded by Claude Code
- `skills/` — Custom skills (copy to `.claude/skills/` to activate)
- `agents/` — Custom agent definitions

## Available Skills

- **skill-creator** — Create, test, and iterate on new skills. Invoke with `/skill-creator`.

## Adding a New Skill

1. Create a directory under `skills/<skill-name>/`
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) and markdown instructions
3. Optionally add `scripts/`, `references/`, and `assets/` subdirectories
4. To activate, copy or symlink into `.claude/skills/`

## Adding a New Agent

1. Create a markdown file under `agents/<agent-name>.md`
2. Define the agent's purpose, tools, and instructions
