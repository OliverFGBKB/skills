# skills

Personal agent skills. Written first for [Claude Code](https://claude.com/claude-code),
but they degrade gracefully in any agent that reads `SKILL.md` — no `Artifact` tool,
no companion skills, still works.

A skill is a directory containing a `SKILL.md`: YAML frontmatter with a name and
description, and a markdown body the agent follows when the skill is invoked. These
are mine — opinionated, small, and written to be read rather than configured.

## Skills

| Skill | What it does |
| --- | --- |
| [`brief`](skills/brief) | Turns the current conversation into a shareable project brief for **human** teammates — goal, chosen route, routes rejected, key decisions, progress — published as a styled web page. The human counterpart of `handoff`. |

## Install

Copy the skill directory into your user-level skills folder:

```bash
git clone https://github.com/OliverFGBKB/skills.git
cp -r skills/skills/brief ~/.claude/skills/
```

Restart your session — Claude Code enumerates skills at startup, so a
freshly copied skill is not callable until then. Then invoke it by name:

```bash
/brief for the backend team
```

For a project-scoped install, copy into `.claude/skills/` inside the repo
instead, and commit it so teammates get it on their next pull.

If you run several agents, symlink instead of copying so there is one source of
truth and an edit lands everywhere at once:

```bash
for dir in ~/.claude/skills ~/.agents/skills ~/.config/opencode/skills; do
  [ -d "$dir" ] && ln -sfn "$PWD/skills/brief" "$dir/brief"
done
```

If you use the [`skills` CLI](https://www.npmjs.com/package/skills), this repo
follows the conventional `skills/<name>/` layout:

```bash
npx skills add OliverFGBKB/skills --skill brief
```

## Conventions

Every skill here follows a few rules, because they are the difference between a
skill that gets used and a prompt that gets ignored:

- **One file if possible.** `SKILL.md` carries the instructions. Extra files
  only when they hold something that is genuinely data — a stylesheet, a
  template — not more prose.
- **No scripts, no dependencies.** If a skill needs to install something to
  work, it is doing too much.
- **`disable-model-invocation: true`** on anything that produces an artifact
  the user should have asked for. These fire on `/name`, not on a hunch.
- **`argument-hint`** wherever one argument changes the whole output. Ask
  rather than guess.

## License

MIT — see [LICENSE](LICENSE).
