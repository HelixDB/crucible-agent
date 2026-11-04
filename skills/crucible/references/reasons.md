# Refusals and what to do

> Every `rejected: <code>` reason Crucible answers, its CLI exit code, and the fix; plus sign-in, key and setup failures.

A refused run answers `{"status":"error"|"infra","source":"client","reason":"rejected: <code>: <detail>"}`.
Match on `<code>`; the detail says the specifics (a wait, a reset time, a job id). Exit 64 means
retrying cannot help, 3 means retry with backoff, 2 means fix something first.

## Codes

| Code | Exit | Do |
|---|---|---|
| `unknown_repo` | 64 | The repo is not connected, has no base yet, or `origin` is spelled differently from GitHub (case matters). Call `status()`; the human installs the Helix Crucible App on the repo at https://crucibl.sh/github, then the first base builds |
| `base_building` | 3 | The repo's base is building (minutes). Retry in a minute; the CLI waits by itself |
| `base_build_failed` | 2 | The default branch's base build failed (the detail names the build's job). The human reads it on https://crucibl.sh/jobs; the default branch needs a fix (an install or warm-up step, [config.md](config.md)), and the next push to it rebuilds |
| `insufficient_balance` | 64 | No credit for the job. Tell the human: an admin adds credit at https://crucibl.sh/billing. Do not retry until they have |
| `tenant_frozen` | 64 | The workspace is frozen by Crucible (the detail says why). Do not retry; the human writes to access@crucibl.sh |
| `daily_egress_cap` | 64 | The workspace's outbound traffic for today is used up. Wait until the reset time in the detail (next UTC midnight) |
| `key_spend_cap` | 64 | This API key's jobs reached the key's daily spend cap. Wait until the reset time in the detail (next UTC midnight), or the human issues a key with a higher cap at https://crucibl.sh/keys |
| `starter_spend_cap` | 64 | The workspace reached its daily spend cap while on its starter credit. Wait until the reset time in the detail (next UTC midnight), or an admin adds credit at https://crucibl.sh/billing |
| `concurrency_limit` | 3 | The workspace or key runs as many jobs as it may (4 agent jobs). Wait for one to finish, or `cancel_job` one you no longer need |
| `rate_limited` | 3 | Too many new jobs per second. Retry after the milliseconds in the detail |
| `queue_full` | 3 | Every slot is busy and the queue is full. Retry in a minute or two |
| `queue_timeout` | 3 | The job waited 15 min in the queue. Retry later |
| `no_capacity` | 3 | No host has a free slot (or AWS has no capacity). Retry with backoff: 30 s, 1 min, 2 min |
| `admission_closed` | 3 | Crucible stopped admitting jobs for now (maintenance). Retry in a few minutes |
| `blob_budget_exceeded` | 3 | The workspace's upload budget is full. Retry when running jobs finish |
| `missing_blobs` | 3 | Uploaded files expired before the job started. Run again |
| `stale_base` | 3 | The repo's base changed while submitting. Run again |
| `unknown_commit` | 64 | GitHub has no such commit, branch, tag or pull request in the repo (`ref`, `pr` or `base`). Push it first, or fix the name |
| `patch_rejected` | 64 | The `diff` cannot become a run; the detail starts with why: `malformed`, `binary`, `too_large`, `does_not_apply` (make `base` the commit the diff was made against) or `base_too_far`. For binary or large changes, or a far base, use the CLI's `crucible test` |
| `unknown_job` | 64 | No such job, or it is not yours. Check the `job_id` |
| `custom_command_forbidden` | 64 | Agent keys run only the repo's detected tests: use `crucible test`, never `crucible run -- <cmd>` |
| `scope_forbidden` | 64 | The call needs a maintainer key. Do not get one: ask the human to do it themselves |

## Other failures

| Symptom | Do |
|---|---|
| The MCP tools answer HTTP 401, or the client says the server needs sign-in | Ask the human to sign in (Claude Code: `/mcp`, then `crucible`) ([clients.md](clients.md)) |
| `status()` shows no workspace | The human creates one at https://crucibl.sh/start (or, where it shows Request access, writes to access@crucibl.sh) |
| The diff does not apply | Make `ref` the commit the diff was made against (`git merge-base HEAD origin/HEAD`), and pushed |
| `API: … Unauthenticated` from the CLI (exit 64) | The key is missing, revoked or mistyped: the human runs `crucible login` in their terminal with a new agent key from https://crucibl.sh/keys |
| `API: … PermissionDenied` (exit 64) | The key cannot do this: agent keys run tests and read their own jobs |
| A TLS error naming the certificate (exit 64) | The config file is for another Crucible endpoint: the human deletes it, reinstalls the CLI and logs in again ([cli.md](cli.md)) |
| `no API endpoint: …` or `no API key: …` (exit 64) | The CLI is not installed or the human has not run `crucible login` ([cli.md](cli.md)) |
| `no git checkout … found`, `has no origin remote`, or `is not a github.com repository` (exit 64) | Run inside a checkout whose `origin` is the GitHub repo, or set `repo = "OWNER/NAME"` in `.crucible.toml` |
| A pull request has no `Crucible` check | The repo's PR CI is GitHub Actions, the default: for Crucible's own check the human chooses **Crucible check** for the repo at https://crucibl.sh/github, or the default branch's `.crucible.toml` sets `[checks] pull_requests = true` |
| The PR check stays queued | No credit, or the base is building: `status()` |
| The PR check failed at once saying no base can come | The default branch's base build failed or was skipped (no test setup found, the repo quota): fix the default branch, push, re-run the check |
| An Actions job never starts | Its label is not a `crucible-*-arm` class alone, the Runner App does not cover the repo, or the repo belongs to a personal account ([actions.md](actions.md)) |

Report a Crucible bug to access@crucibl.sh with the `job_id`, the command or tool call, the JSON
answer and the time (UTC).
