# dotodo-io/skills

Agent skill for [dotodo](https://dotodo.io) — install via [skills.sh](https://skills.sh).

```bash
npx skills add dotodo-io/skills --skill dotodo
```

This copies **skill files only**. It does **not** configure the MCP server. For skill + MCP URL:

```bash
npx dotodo install
```

Then sign in with OAuth in your AI client: https://dotodo.io/setup/mcp

## Layout

```
skills/dotodo/SKILL.md
```

Source of truth lives in the product monorepo (`skill/dotodo`). This repo is the public skills.sh package.

## License

MIT
