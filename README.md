# Zalmox for AI agents

A Claude Code plugin marketplace with one plugin, `zalmox`: the Zalmox MCP server (`zalmox mcp`, from the `zalmox` npm package) and a skill
that tells the agent how to use it well (quote the cost first, draft before final, save files into the project).

Install it in Claude Code:

```sh
npx zalmox login                              # once: approve on zalmox.io
claude plugin marketplace add absolutx13/zalmox-agent
claude plugin install zalmox@zalmox
```

Other agents can use the MCP server directly (see the `zalmox` npm package README) and the skill file in
`plugins/zalmox/skills/zalmox/SKILL.md`.
