# skills

Personal agent skills for Claude Code, OpenCode, and other harnesses.

| Skill | What it does |
| --- | --- |
| [opencode-fast](skills/opencode-fast/SKILL.md) | Route hands-on work (implementation, fixes, rebases, exploration) to an OpenCode Go worker. Claude specs, reviews, and verifies. |

## Install

```bash
git clone https://github.com/vikas-shopos/skills ~/Projects/skills
ln -s ~/Projects/skills/skills/opencode-fast ~/.claude/skills/opencode-fast   # Claude Code
ln -s ~/Projects/skills/skills/opencode-fast ~/.agents/skills/opencode-fast   # shared agents dir
```

Or with the `skills` CLI:

```bash
npx skills add vikas-shopos/skills --skill opencode-fast
```

## Requirements for opencode-fast

- `opencode2` on PATH with an OpenCode Go login (`opencode2 auth list` shows "OpenCode Go … stored").
- Optional: the Solo MCP with an OpenCode agent tool, for the preferred spawn path.

## License

MIT
