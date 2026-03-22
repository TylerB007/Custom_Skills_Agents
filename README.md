# Custom Skills & Agents

A collection of custom [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills and agents.

## What's Inside

### Skills

Skills extend Claude Code with specialized capabilities. They live in `.claude/skills/` for auto-discovery or in `skills/` for sharing and browsing.

| Skill | Description |
|-------|-------------|
| [skill-creator](.claude/skills/skill-creator/) | Create, evaluate, and iterate on new skills. From [anthropics/skills](https://github.com/anthropics/skills). |
| [lp-strategy-dev](.claude/skills/lp-strategy-dev/) | Co-develop LP tools, delta-neutral strategies, Uniswap V3 analytics, and DeFi software. Includes 5 deep reference files. |

### Agents

Custom agent definitions live in `agents/`. *(None yet — add your own!)*

## Quick Start

1. Clone this repo
2. Open the directory in Claude Code — skills in `.claude/skills/` are auto-loaded
3. Use `/skill-creator` to start building your own skills

## Adding Skills

```
skills/my-skill/
├── SKILL.md          # Required: name, description, instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: docs loaded into context
└── assets/           # Optional: templates, icons, fonts
```

To activate a skill from `skills/`, copy or symlink it into `.claude/skills/`.

## Resources

- [Claude Code Skills Documentation](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Official Skills Repository](https://github.com/anthropics/skills)
- [Skills Guide (PDF)](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
