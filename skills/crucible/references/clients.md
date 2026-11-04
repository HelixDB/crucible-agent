# Clients

> How each agent client adds the hosted Crucible MCP server, signs in, and finds the skill; plus permissions and keys for headless agents.

The server is `https://mcp.crucibl.sh/mcp` (streamable HTTP). Sign-in is OAuth in the browser
with GitHub or Google, started by the client the first time a tool needs it. Tool calls return
within `wait_seconds` (at most 45), so no client needs a longer tool timeout.

## Claude Code

```sh
claude plugin marketplace add HelixDB/crucible-agent
claude plugin install crucible@crucible-agent
```

The plugin holds this skill and the server. Without it:

```sh
claude mcp add --transport http crucible https://mcp.crucibl.sh/mcp
```

`--scope user` adds it for every project; `--scope project` writes `.mcp.json` (no secret in it,
safe to commit):

```json
{ "mcpServers": { "crucible": { "type": "http", "url": "https://mcp.crucibl.sh/mcp" } } }
```

Sign in: `/mcp`, pick `crucible`, Authenticate. Skill folders: `~/.claude/skills/crucible` or the
repo's `.claude/skills/crucible`. Permissions, in `.claude/settings.json`:

```json
{ "permissions": { "allow": ["mcp__crucible__status", "mcp__crucible__run_tests", "mcp__crucible__get_job", "mcp__crucible__list_jobs", "mcp__crucible__get_pr_check", "mcp__crucible__actions_setup", "Bash(crucible test *)", "Bash(crucible jobs *)"] } }
```

## Codex

```sh
codex mcp add crucible --url https://mcp.crucibl.sh/mcp
codex mcp login crucible
```

Or in `~/.codex/config.toml`:

```toml
[mcp_servers.crucible]
url = "https://mcp.crucibl.sh/mcp"
```

`codex mcp login crucible --no-browser` prints the sign-in link instead of opening it. Skill
folders: `~/.agents/skills/crucible` or the repo's `.agents/skills/crucible`.

## Cursor

`~/.cursor/mcp.json` (every project) or the repo's `.cursor/mcp.json`:

```json
{ "mcpServers": { "crucible": { "url": "https://mcp.crucibl.sh/mcp" } } }
```

Or open `cursor://anysphere.cursor-deeplink/mcp/install?name=crucible&config=eyJ1cmwiOiJodHRwczovL21jcC5jcnVjaWJsLnNoL21jcCJ9`.
Sign in: Cursor Settings, MCP, `crucible`, Connect. Skill folders: `~/.cursor/skills/crucible`
or `.agents/skills/crucible`. Without skills, copy https://crucibl.sh/skills/crucible.mdc to the
repo's `.cursor/rules/crucible.mdc`.

## VS Code (GitHub Copilot)

```sh
code --add-mcp '{"name":"crucible","type":"http","url":"https://mcp.crucibl.sh/mcp"}'
```

Or the repo's `.vscode/mcp.json`:

```json
{ "servers": { "crucible": { "type": "http", "url": "https://mcp.crucibl.sh/mcp" } } }
```

The browser opens for sign-in on the first connection. Skill folders: `~/.copilot/skills/crucible`,
or the repo's `.github/skills/crucible`.

## Gemini CLI

```sh
gemini mcp add --transport http crucible https://mcp.crucibl.sh/mcp
```

`--scope user` adds it for every project. In `settings.json` the entry is
`{"mcpServers": {"crucible": {"httpUrl": "https://mcp.crucibl.sh/mcp"}}}`. Sign in:
`/mcp auth crucible`. Skill folders: `~/.gemini/skills/crucible` or `.agents/skills/crucible`.

## Windsurf

`~/.codeium/windsurf/mcp_config.json`:

```json
{ "mcpServers": { "crucible": { "serverUrl": "https://mcp.crucibl.sh/mcp" } } }
```

Windsurf asks to sign in when the server first needs it. Without skills, append
https://crucibl.sh/skills/AGENTS.md to the repo's `AGENTS.md`.

## claude.ai and Claude Desktop

Settings, Connectors, Add custom connector, URL `https://mcp.crucibl.sh/mcp`, then Connect. A
connector added there also shows in Claude Code signed in to the same account. Here
`run_tests` takes a pushed `ref`, a `pr`, or a `diff` the chat produced with its `base`.

## Any other agent

- The skill: `npx skills add HelixDB/crucible-agent`, or download it:

  ```sh
  mkdir -p ~/.agents/skills/crucible/references && cd ~/.agents/skills/crucible
  curl -fsSLO https://crucibl.sh/skills/crucible/SKILL.md
  for f in getting-started clients mcp results reasons config actions cli; do curl -fsSL -o "references/$f.md" "https://crucibl.sh/skills/crucible/references/$f.md"; done
  ```

- No skills at all: `AGENTS.md` (https://crucibl.sh/skills/AGENTS.md) is the same text.

## Headless agents and CI: agent keys

Where no browser can sign in, an **agent** key works as a bearer token. An admin creates it at
https://crucibl.sh/keys and stores it in the environment as `CRUCIBLE_KEY`; it never goes into a
prompt or a committed file. Maintainer keys are refused.

- Claude Code `.mcp.json`: `"headers": {"Authorization": "Bearer ${CRUCIBLE_KEY}"}` beside `"url"`.
- Codex `config.toml`: `bearer_token_env_var = "CRUCIBLE_KEY"` under `[mcp_servers.crucible]`.
- Cursor `mcp.json`: `"headers": {"Authorization": "Bearer ${env:CRUCIBLE_KEY}"}`.
- The CLI reads `CRUCIBLE_KEY` itself ([cli.md](cli.md)).

## Sandboxes

An agent sandbox that limits the network needs these hosts: `mcp.crucibl.sh` (the server, used
by the client), `get.crucibl.sh` (CLI install), `grpc.crucibl.sh` (CLI runs), and `github.com`
for `git fetch`.
