# The `crucible` CLI

> Install, log in, run the working tree's tests, read jobs, exit codes, and the CLI's local MCP server.

The CLI tests the working tree as it is, uncommitted changes and binary files included, against
the repo's base: it uploads only what differs (`.gitignore`d files are left out, at most 64 MiB of
changes). Use it when the hosted server cannot take the change (binary files, a large diff) or
for a shell-only agent.

## Install and log in

```sh
curl -fsSL https://get.crucibl.sh/install.sh | sh
crucible --version
```

The installer puts `crucible` in `~/.local/bin` (macOS or Linux, x86_64 or arm64) and writes its
config file without a key. The **human** then creates an **agent** key at https://crucibl.sh/keys
and runs `crucible login` in their own terminal: it reads the key without echo, checks it, and
stores it at mode 600. An agent never sees, asks for or stores the key. In CI, the key comes from
a secret in `CRUCIBLE_KEY`.

Config file: `~/.config/crucible/config.toml` (Linux) or
`~/Library/Application Support/crucible/config.toml` (macOS). The environment overrides it:
`CRUCIBLE_KEY`, `CRUCIBLE_ENDPOINT`, `CRUCIBLE_CA_FILE`, `CRUCIBLE_CONFIG`.

## Commands

Run from inside a checkout of a connected repo (or name one with `-C DIR`).

```sh
crucible test                       # the detected suite (affected tests only, when selection applies)
crucible test -- tests::adds        # only tests matching a filter (several allowed)
crucible test --failed-first        # the last run's failures first, then the rest
crucible test --fail-fast           # stop at the first failure
crucible test --no-skip             # the full suite, doc tests included: no affected-test selection
crucible test --stream-output       # the test output on stderr as it runs
crucible test --timings             # add per-step timings and usage to the result
crucible test -C ../other-repo      # as if started in that directory
crucible test --wait 45             # detached: at most 45 s, then {"status":"running",…} (exit 75)
crucible test --ref main            # a pushed branch, tag, SHA or #<pr> on GitHub, no local files
crucible status                     # who you are, the workspace, its repos' bases, the next step
crucible status --repo OWNER/NAME
crucible checks get OWNER/NAME --pr 123   # a PR's Crucible check (or --sha <head>)
crucible actions setup OWNER/NAME   # runner labels, the App and group state, a snippet
crucible ci report OWNER/NAME       # the repo's Actions runs measured, findings, the suggested mode
crucible ci report OWNER/NAME --days 30
crucible jobs list                  # your recent jobs
crucible jobs list --repo OWNER/NAME --limit 50
crucible jobs get 0192f0c4-0000-7000-8000-000000000000
crucible jobs get 0192f0c4-0000-7000-8000-000000000000 --timings
crucible jobs get 0192f0c4-0000-7000-8000-000000000000 --wait 45
crucible jobs cancel 0192f0c4-0000-7000-8000-000000000000
crucible jobs logs 0192f0c4-0000-7000-8000-000000000000 --grep 'FAIL|panicked'   # a page of its log
crucible jobs logs 0192f0c4-0000-7000-8000-000000000000 --from 900 --limit 100
crucible feedback --kind wrong_result --job 0192f0c4-0000-7000-8000-000000000000 MESSAGE   # what you expected, what happened
crucible login                      # the human, in their own terminal
crucible daemon status              # the background daemon the CLI starts
crucible daemon stop
crucible mcp                        # the local MCP server on stdio (below)
```

`crucible jobs list --page <next_page>` continues a listing. Maintainer keys only, never for
agents: `crucible run -- make check` (any command), `crucible account`, `crucible topup 20`,
`crucible repo register`, `crucible ci set OWNER/NAME --mode actions` (a repo's CI mode:
`actions` or `crucible_checks`). `crucible daemon run` is what the CLI starts by itself.

## Output and exit codes

Every command prints **one JSON line** on stdout ([results.md](results.md)); progress and
waits go to stderr (`crucible: queued, position 3, ETA ~40 s`). Branch on the exit code:

| Exit | Status |
|---|---|
| 0 | `passed` |
| 1 | `failed`, `build_failed`, `no_tests` |
| 2 | `error`, `timed_out`, `cancelled` |
| 3 | `infra`: Crucible's side failed, retry |
| 64 | Usage, configuration, or a refusal retrying cannot fix: `reason` says what to do ([reasons.md](reasons.md)) |

A busy cell queues the job (at most 15 min); the CLI waits through a base build of the repo and
reconnects after an API restart, keeping its place. Ctrl-C cancels the job: free while it waits
in the queue; once its VM started it is billed for the time it ran (5 s at least).

## The local MCP server

`crucible mcp` serves the same jobs over MCP on stdio, from the directory it starts in (`-C DIR`
names another), using the CLI's config. It tests the local working tree, so it needs no diff. Add
it under its own name so it does not clash with the hosted server:

```sh
claude mcp add --scope user crucible-local -- ~/.local/bin/crucible mcp
```

| Tool | Arguments |
|---|---|
| `run_tests` | `filters`, `fail_fast`, `failed_first`, `no_skip`, `wait_seconds` |
| `get_job` | `id`, `wait_seconds`, `failures_page` |
| `list_jobs` | `page` |
| `cancel_job` | `id` |
| `get_job_logs` | `id`, `from`, `limit`, `grep`, `ignore_case` |
| `send_feedback` | `kind`, `message`, `job_id`, `tool` |
| `status` | `repo` |
| `get_pr_check` | `repo`, `pr`, `sha` |
| `actions_setup` | `repo` |
| `ci_report` | `repo`, `days` |
| `account` | none |

`run_tests` waits at most `wait_seconds` (≤ 45), then answers `{"status":"running","job_id":…}`;
the job runs on (a client's cancel or disconnect does not stop it): `get_job` follows it,
`cancel_job` stops it. `account` needs a maintainer key.
