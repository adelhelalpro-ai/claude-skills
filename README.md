# Claude Code Skills

Custom skills for [Claude Code](https://claude.com/claude-code), ready to install.

## Skills included

| Skill | What it does |
|---|---|
| [`/army`](skills/army/SKILL.md) | Orchestrator mode: Claude delegates maximally to an army of subagents (right model, right effort per task) and spends its own tokens only on decomposition, specs, arbitration, and integration. |

## Installation

Copy the skill folder into your personal skills directory:

```bash
git clone https://github.com/adelhelalpro-ai/claude-skills.git
cp -R claude-skills/skills/army ~/.claude/skills/
```

That's it. In any Claude Code session, type `/army`.

> Skills placed in `~/.claude/skills/` are available in **all** your projects.
> To make a skill available only in one project, put it in `<project>/.claude/skills/` instead.

## Usage

- `/army` — arms orchestrator mode for the whole session; `/army <task>` starts on a task immediately. Say "stop army" to deactivate.

## Writing your own skills

A skill is just a folder with a `SKILL.md` file containing YAML frontmatter (`name`, `description`) followed by instructions. See `/army` here as an example, or run `/skill-creator` in Claude Code.

## License

MIT
