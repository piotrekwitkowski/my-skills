# my-skills

Custom AI coding skills.

## Skills

| Skill | Description |
|-------|-------------|
| [gh-push](./gh-push/SKILL.md) | Push code to GitHub without `git push` — uses `gh` CLI and GitHub API exclusively |

## Installation (OpenCode)

To install a skill globally (available in all projects), copy it to `~/.opencode/skills/`:

```bash
mkdir -p ~/.opencode/skills/gh-push
cp gh-push/SKILL.md ~/.opencode/skills/gh-push/SKILL.md
```

To install for a specific project only, copy it into the project's `.opencode/skills/` directory:

```bash
mkdir -p <project-root>/.opencode/skills/gh-push
cp gh-push/SKILL.md <project-root>/.opencode/skills/gh-push/SKILL.md
```
