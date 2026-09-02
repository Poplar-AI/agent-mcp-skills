# Poplar agent skills

Skills that teach a coding agent or assistant how to operate [Poplar](https://usepoplar.com) — the AI social-media platform — through its MCP server: draft posts in a customer's own voice, add media and polls, run approvals and comments, and schedule, with every guardrail explained.

## Install

```bash
npx skills add Poplar-AI/agent-mcp-skills
```

Works with Claude Code, Cursor, and any agent that reads the [Agent Skills](https://agentskills.io) format. To see what is in the repo first:

```bash
npx skills add Poplar-AI/agent-mcp-skills --list
```

## Connect the agent to Poplar

Poplar is a remote MCP server at `https://usepoplar.com/api/mcp`.

- **Claude.ai / Claude Desktop** — add the URL as a connector and sign in. Nothing to paste.
- **Claude Code / Cursor / anything with headers** — create an API key in Poplar → Settings → API keys, then:

```json
{ "mcpServers": { "poplar": { "url": "https://usepoplar.com/api/mcp", "headers": { "Authorization": "Bearer pop_live_..." } } } }
```

## What's here

| Skill | What it teaches |
|---|---|
| [`skills/poplar`](skills/poplar/SKILL.md) | The full Poplar MCP surface — writing with Poplar's writer, media, composition, approvals, comments, scheduling — and the rules that keep an agent from doing damage. |

The skill contains integration knowledge only. Poplar's writer, prompts and learned rules run inside Poplar and are not in this repository.

## License

Proprietary — see [`skills/poplar/LICENSE.md`](skills/poplar/LICENSE.md).
