# dotodo-io/skills

Source of truth for the [dotodo](https://dotodo.io) agent skill (`SKILL.md`, icon, agents).

```bash
npx skills add dotodo-io/skills --skill dotodo
```

Catalog: https://skills.sh/dotodo-io/skills/dotodo

## Downloads

| Artifact | URL |
| -------- | --- |
| `SKILL.md` | https://raw.githubusercontent.com/dotodo-io/skills/main/skills/dotodo/SKILL.md |
| `skill.zip` (ChatGPT upload) | https://github.com/dotodo-io/skills/releases/latest/download/skill.zip |

The zip has a top-level `dotodo/` folder (`dotodo/SKILL.md`). It is a **GitHub Release asset**, not a file in git. A new release is created on every merge that changes `skills/dotodo/**` (or via Actions → Release skill.zip → Run workflow).

## Layout

```
skills/dotodo/SKILL.md
skills/dotodo/icon.svg
skills/dotodo/agents/
```

## License

MIT
