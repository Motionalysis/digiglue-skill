# DigiGlue Agent Skill

An Agent Skill for the [DigiGlue](https://digiglue.io) MCP connector. Helps Claude design, size, and quote packaging die-lines without re-discovering the routing rules, vocabulary, and per-tool conventions on every conversation.

## What this Skill teaches Claude

- **Tool routing** — which of DigiGlue's 12 tools to call for which kind of user request.
- **Packaging vocabulary** — flutes (A/B/C/E), ECT board grades, common shape aliases (mailer / EComm / HSC / RSC), brand-name → template-ID mapping.
- **Hard rules** — Y-before-X dimensions, single-call batching for multi-item requests, customer-required quoting, default material/sheet fallback behavior.
- **Worked example workflows** — six scenarios mirroring the public examples at [digiglue.io/docs/connector](https://digiglue.io/docs/connector).

## How to use it

### claude.ai
1. Pro/Max/Team/Enterprise plan with code execution enabled.
2. Settings → Features → Skills → Upload.
3. Upload this repository as a zip.

### Claude Code
1. Clone this repository into `~/.claude/skills/digiglue/` or `<project>/.claude/skills/digiglue/`.
2. Claude Code auto-discovers Skills in those locations.

### Claude API
Upload via the `/v1/skills` endpoints. See [the API guide](https://platform.claude.com/docs/en/build-with-claude/skills-guide).

## The DigiGlue MCP connector

This Skill works alongside the [DigiGlue MCP](https://digiglue.io/api/mcp). It assumes the user has the connector enabled — without it, the routing instructions in `SKILL.md` won't have anything to route to.

- **Server URL**: `https://digiglue.io/api/mcp`
- **Auth**: OAuth 2.0 via claude.ai
- **Public docs**: https://digiglue.io/docs/connector

## License

MIT — see [LICENSE](LICENSE).
