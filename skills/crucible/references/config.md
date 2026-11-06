# Detection and `.crucible.toml`

> How Crucible finds a repo's tests with no config, what the test VM can reach, and every `.crucible.toml` key.

## Zero config

Crucible inspects the repo root and takes the first match, Rust, Go, Node, Python:

| Marker | Tests run as | Warm-up in the base |
|---|---|---|
| `Cargo.toml` | `cargo nextest run --workspace --no-fail-fast`, then `cargo test --doc` | `cargo test --no-run` |
| `go.mod` | `go test -json -fullpath ./...` | `go test -count=1 -run '^$' ./...` |
| `package.json` with `vitest`, `jest` or a `node --test` script | that runner, with a report | install from the lockfile (`pnpm-lock.yaml`, `bun.lock`, `yarn.lock`, `package-lock.json`) |
| `package.json` with another `test` script | `npm test` (or pnpm, yarn, `bun run test`), exit code only | as above |
| `pyproject.toml`, `setup.py`, `pytest.ini` or `requirements.txt` | `pytest` (`uv run` with `uv.lock`, else the project's `.venv`) | `uv sync`, `poetry install`, or a venv with `pip install` |

- Node needs a lockfile at the root; a pip project must list pytest in its requirements.
- Toolchains: Rust 1.97.1, Go 1.27.3, Node 24 (pnpm, yarn, bun), Python 3.12 (uv, poetry).
- The test VM reaches only the public package registries (crates.io, npm, PyPI, the Go proxy,
  the Buf Schema Registry, Debian) and, for git dependencies, **public** GitHub repos (fetched
  anonymously, never pushed): private git dependencies and private registries fail.
  No Docker and no services. Tests that need them belong on Actions runners
  ([actions.md](actions.md)).
- Commands run as the user `runner` with passwordless `sudo`.

## The base

When the Helix Crucible App is installed on a repo, Crucible builds a warm **base** from the
default branch's head: it detects the tests and runs the warm-ups (installs, test builds). Every
push to the default branch rebuilds it (after 30 s without another push, at most 5 min later);
jobs use the previous base meanwhile. A run tests the base plus your change. Bases unused for 14
days are deleted.

## `.crucible.toml`

Optional, at the repo root, read **from the default branch only**: a change takes effect once it
is merged and the base rebuilt, never from a PR or a diff. Every key is optional; unknown keys
are errors.

```toml
# .crucible.toml (every key optional)
[test]
command = ["cargo", "test", "--workspace", "--all-features", "--no-fail-fast", "--message-format=json"]
runner = "cargo"     # cargo | nextest | go | vitest | jest | node_test | pytest | script
workdir = "backend"  # run from a subdirectory (monorepos)
env = { RUST_LOG = "warn" }
timeout_secs = 1200  # 15..=3600, default 900

[warm]               # replaces the detected warm-ups (at most 16)
commands = [["cargo", "test", "--workspace", "--all-features", "--no-run"]]

[shard]
max = 4              # split a long suite across up to 4 microVMs; 1 = never

[flaky]              # the defaults
retries = 2          # retries of a failing test, 0..=3
quarantine = true    # tests that keep flaking stop failing runs

[checks]
pull_requests = true   # Crucible's own `Crucible` PR check (default: the dashboard's choice, off)

[resources]
size = "large"            # every job and base build of the repo on 8 vCPU / 16 GiB
build_timeout_secs = 2700 # base builds, 15..=3600, default 1800
```

- `runner = "cargo"` alone switches a Rust repo from nextest to plain `cargo test` (no selection,
  no shards).
- A custom `command` replaces the detected one. Each test is reported only if it writes what
  `runner` reads: cargo's `--message-format=json`, `go test -json`, jest's
  `--json --outputFile=/run/crucible/jest.json`, or for vitest, node_test and pytest a JUnit file
  at `/run/crucible/junit.xml`. Otherwise only its exit code counts.
- `[checks] pull_requests` wins over the dashboard's choice of the repo's PR CI when set: `true`
  is **Crucible check**, `false` **GitHub Actions** (the default); base builds and agent runs go on
  either way.
- `repo = "OWNER/NAME"` at the top names the repo when the checkout has no github.com `origin`
  (CLI only).
- `[resources] size`: `"standard"` (the default: 2 vCPU / 4 GiB, $0.0025 a minute) or `"large"`
  (8 vCPU / 16 GiB, $0.01 a minute; it counts as 4 of the workspace's concurrent jobs). Both are
  Arm at $0.00125 per vCPU-minute, billed per second.

## Shards

A run whose tests are predicted to take long (over about 20 s for an agent run, 2 min for a PR
check) is split across microVMs by test, then merged: `shards` in the result says how many. The
count is at most `[shard] max`, the workspace's limit (4) and its free job slots. Filtered runs,
plain `cargo test`, `node --test` and script runners are never split.

## Flaky tests

A failing test is retried up to `[flaky] retries` times; one that then passes is `flaky` and
does not fail the job. With `quarantine = true`, a test flaky in 2 of its last 20 PR-check or
base-build runs is quarantined: its failures are reported under `quarantined` and fail nothing,
until 30 clean runs release it. Agent and fork runs never change the list.
