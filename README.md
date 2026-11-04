# Crucible for coding agents

[Crucible](https://crucibl.sh) runs a repo's tests in a fresh Firecracker microVM per run, on a
warm base of the repo, and answers with one JSON result an agent can act on. This repo holds the
`crucible` agent skill and the hosted MCP server's client config.

Fastest start: tell your agent `Set up Crucible for this repo: https://crucibl.sh/agents.md`.

## Install

| Client | Command |
|---|---|
| Claude Code (skill and server) | `claude plugin marketplace add HelixDB/crucible-agent`, then `claude plugin install crucible@crucible-agent` |
| Any agent with skills (Codex, Cursor, Gemini CLI, VS Code, Claude Code) | `npx skills add HelixDB/crucible-agent` |
| Cursor (server) | [Add to Cursor](cursor://anysphere.cursor-deeplink/mcp/install?name=crucible&config=eyJ1cmwiOiJodHRwczovL21jcC5jcnVjaWJsLnNoL21jcCJ9) |
| Cursor (rules, without skills) | copy `.cursor/rules/crucible.mdc` into the repo's `.cursor/rules/` |
| Agents that read only `AGENTS.md` | append `AGENTS.md` to the repo's `AGENTS.md` |

Every other client's server line: https://crucibl.sh/docs/clients.md.

## Contents

- `skills/crucible/`: the skill (`SKILL.md` and `references/`).
- `.claude-plugin/`: the Claude Code marketplace and plugin manifests.
- `.mcp.json`: the hosted server, `https://mcp.crucibl.sh/mcp`.
- `AGENTS.md`, `.cursor/rules/crucible.mdc`: the skill as one file, generated from `SKILL.md`.

Docs: https://crucibl.sh/llms.txt. Support: access@crucibl.sh.

This repo is published from Crucible's own repository; changes made here are overwritten.
