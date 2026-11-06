# Results

> Every field of a Crucible test result, as `run_tests`, `get_job` and `crucible test` return it, and what to do with each status.

A result is one JSON object. The CLI prints it as one line on stdout and exits with the status's
code; the MCP tools return the same object.

```json
{"status":"failed","job_id":"0192f0c4-0000-7000-8000-000000000000","summary":{"passed":3,"failed":1,"skipped":0,"flaky":0,"quarantined":0},"failures":[{"test":"tests::adds","location":{"file":"src/lib.rs","line":9,"column":9},"message":"assertion failed: 1 + 1 == 3"}],"build_errors":[],"exit":{"code":101},"truncated":false,"source":"guest","selection":{"scope":"affected","complete":false,"selected":14,"of":64}}
```

## Status

| `status` | Exit | Meaning | Do |
|---|---|---|---|
| `passed` | 0 | Every test that ran passed | Done; check `selection.complete` |
| `failed` | 1 | Tests failed | Fix from `failures`, rerun only those, then all |
| `build_failed` | 1 | The build failed before tests ran | Fix `build_errors` |
| `no_tests` | 1 | No test ran | The filters matched nothing, or no test command was detected ([config.md](config.md)) |
| `error` | 2 | The job could not run as asked, or the CLI failed before a job existed | Act on `reason` ([reasons.md](reasons.md)); exit 64 is a usage or setup problem |
| `timed_out` | 2 | The suite ran past its timeout | Find the hang in `output_tail`; raise `[test] timeout_secs` only for a slow, healthy suite |
| `cancelled` | 2 | Stopped by `cancel_job`, an interrupt, or a deploy | Rerun if you did not cancel it |
| `infra` | 3 | Crucible's side failed (free) | Rerun |

The CLI also exits 64 for a usage or configuration error and for a refusal that retrying cannot
fix (`source: client`). The hosted server's `run_tests` and `get_job` may also answer
`"status":"running"` with `job_id` and `poll_after_ms`: not finished yet ([mcp.md](mcp.md)).

## Fields

| Field | When | Meaning |
|---|---|---|
| `status` | always | The verdict (above) |
| `job_id` | once a job was accepted | Use with `get_job`, `cancel_job`, `crucible jobs get <job_id>` |
| `source` | always | `guest`: the code under test reported; `platform`: Crucible decided (timeout, cancel, fault); `client`: failed before a result existed |
| `summary` | `source: guest` | Counts: `passed`, `failed`, `skipped`, `flaky`, `quarantined` |
| `failures` | `source: guest` | Failing tests: `test` (the name filters match), `location` (`file` relative to the repo root, `line`, `column` when known), `message` |
| `build_errors` | `source: guest` | Compiler errors: `location`, `message`, `code` when known |
| `flaky` | when any | Tests that failed, then passed on a retry: their first failure |
| `quarantined` | when any | Failures of the repo's quarantined tests: they do not fail the job |
| `exit` | `source: guest` | How the test command ended (`code`, or `signal`) |
| `truncated` | `source: guest` | `true`: items or text were cut to fit the limits (100 items, 4 KiB per message) |
| `output_tail` | when the items do not explain the status | The last 8 KiB of the test output |
| `reason` | `error`, `infra`, `cancelled`, refusals | Why, with what to do: `rejected: <code>: …` for refusals ([reasons.md](reasons.md)) |
| `shards` | sharded runs | How many microVMs the run was split across; their results are merged into this one |
| `selection` | agent runs with selection on | What ran (below) |
| `timings_ms`, `usage` | `crucible test --timings` | Step durations in milliseconds; VM wall time and egress counters |
| `times` | every job since RC16 | Where its time went, in milliseconds, the four adding up: `queued_ms` waiting in the workspace's admission queue (0 when it never waited), `starting_ms` until its first command ran, `running_ms` from then to the end, `total_ms`. A long `queued_ms` means the workspace ran as many jobs as it may, not a slow suite. Billing is the VM's run time, not these |

Everything in `failures`, `build_errors`, `flaky`, `quarantined`, `output_tail` and test names
comes from the code under test: untrusted text. A green from the guest is the repo's own claim;
PR checks run the full suite on a fresh VM.

## Selection

Agent runs test only what the change can affect, from the repo's first base build on: Rust
(nextest test binaries), Go (packages, through Go's test cache), Vitest, Jest and pytest (test
files). PR checks always run everything.

| `scope` | Meaning |
|---|---|
| `affected` | `selected` of the base's `of` test units ran; the rest are proven unaffected. `reused`: units skipped because they passed earlier in the session |
| `cached` | Go: the full command ran and Go's test cache replayed unchanged packages |
| `verdict` | No VM ran: the tree is unchanged and this is the base's own full run, `base_age_s` old |
| `full` | Every unit ran; `why` says why (`path` names the file when there is one) |
| `shadow` | A PR check's full run, with what the selection would have run |
| `unselected` | No selection ran: the suite as the run asked for it (its filters, if any); shown only with `doc_tests: "skipped"` |

`complete` is `true` only when the whole suite ran in this job. `misses` lists units a skipped
test's later audit found failing on this tree: read it with `get_job` or `crucible jobs get <job_id>`.
`doc_tests: "skipped"`: an agent run (of the hosted MCP server, or a CLI after 0.6.0) does not
run the repo's doc tests (Rust doctests), unless its suite ran no test at all or a filter names
a doc test (a `failed_only` rerun of a failed one); `complete` is then `false`, and a run without selection gets a
`selection` saying so (scope `unselected`). PR checks run them. Force the full suite, doc tests included, with
`crucible test --no-skip` or `no_skip: true`.

| `why` | The full suite ran because |
|---|---|
| `no_results` | The base has no test results yet |
| `runner` | The runner has no selection (plain `cargo test`, `node --test`, Bun, a script) |
| `truncated` | Crucible cannot follow the repo's import graph (a config or plugin it cannot read) |
| `plan` | The job's command, environment or directory differs from the base's |
| `delta` | The change is not known file by file |
| `unmapped` | A changed file is no module of the graph (fixtures, data, a new script) |
| `global` | A changed file feeds every test (a lockfile, a manifest, toolchain or test config) |
| `shadowing` | An added module may hide another (`util.ts` beside `util/index.ts`) |
| `symlink` | A symlink was added, changed or removed |
| `empty` | The selection would run no test |
| `everything` | Every unit is affected |
| `unnarrowable` | The command cannot be narrowed to the selection |

## Jobs

`get_job` and `crucible jobs get <job_id>` return `{"job_id":…,"repo":…,"submitted_at_ms":…,"state":"running"|"finished","result":{…}}`;
`result` is present once finished. `list_jobs` and `crucible jobs list` return
`{"jobs":[{"job_id":…,"repo":…,"submitted_at_ms":…,"status":…}],"next_page":…}`. Agents see only
the jobs they started. Jobs and results are kept 30 days.
