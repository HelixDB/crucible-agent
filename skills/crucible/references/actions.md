# GitHub Actions runners

> Run GitHub Actions jobs on Crucible microVMs with `runs-on: crucible-2vcpu-arm`: labels, setup, what works, limits and prices.

Change `runs-on` and nothing else. Each job runs in its own microVM (Ubuntu 24.04, **arm64**) as
a one-job runner of your GitHub organization, and is billed per second to the workspace.

| `runs-on` | vCPU | RAM | Disk | Per minute |
|---|---|---|---|---|
| `crucible-2vcpu-arm` | 2 | 8 GiB | 50 GB | $0.0025 |
| `crucible-4vcpu-arm` | 4 | 16 GiB | 100 GB | $0.005 |
| `crucible-8vcpu-arm` | 8 | 32 GiB | 200 GB | $0.010 |
| `crucible-16vcpu-arm` | 16 | 64 GiB | 400 GB | $0.020 |

- Use one label alone (case does not matter): never beside `self-hosted`, `linux` or `ARM64`.
- ARM only: there is no x86_64 class. Keep jobs that need x86-only binaries or images on
  GitHub's own runners, or build multi-arch with `docker/setup-qemu-action`.

```yaml
jobs:
  test:
    runs-on: crucible-2vcpu-arm   # one class's label, alone
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci && npm test
```

## Set up (once per organization, an admin)

1. At https://crucibl.sh/github, connect the **Crucible Runners** App
   (https://github.com/apps/crucible-runners) and install it on the GitHub **organization** (not
   a personal account: those are refused) with "Only select repositories": the repos whose
   workflows should run on Crucible. It is separate from the Helix Crucible App that runs PR
   checks.
2. Crucible creates a runner group named **Crucible** in the organization and keeps it to exactly
   the App's repos (public ones allowed only when one of them is public). Edits to it are
   overwritten. Where GitHub refuses the group, the dashboard shows **Default runner group**:
   runners then join the organization's Default group, and any repo it admits can take a
   Crucible runner, billed to the workspace.
3. Change `runs-on` in the workflows. `actions_setup(repo: "OWNER/NAME")` checks steps 1 and 2 and
   returns a snippet.
4. Per repo, the dashboard's **Actions runners** switch (https://crucibl.sh/github, on by default)
   turns Crucible's runners off: the repo's `crucible-*` jobs get a notice instead of a runner and
   the group leaves it out. If the repo's PR CI is these workflows, turn its **Crucible PR checks**
   off there too.

A job whose `runs-on` names a `crucible-` label that is no class (`crucible-32vcpu-arm`, or a
class label beside another label), or that comes from an installation no workspace linked, gets
a failed check run **`Crucible runner (run <id>)`** on its commit saying why and listing the
labels. Fix `runs-on` or link the installation, then re-run the workflow.

## What works

- Private, internal and public repos the App covers; shell steps, `actions/checkout`, artifacts,
  logs, summaries; passwordless `sudo`.
- Docker: `services:`, `container:`, Docker actions, `docker build`, compose and buildx, with
  images from Docker Hub (through Crucible's mirror), ghcr.io, quay.io, mcr.microsoft.com and
  registry.k8s.io.
- `setup-node`, `setup-python`, `setup-go`, `setup-java` for the preinstalled versions (Node
  20/22/24, Python 3.10 to 3.14, Go 1.24 to 1.27, Temurin 17/21); rustup and stable Rust.
- `actions/cache` and the setup-* caches, served by Crucible's own cache (10 GiB per repo).
- Warm caches with no workflow change: a job starts with the package and build caches its
  workflow's last green run left (npm, pnpm, yarn, Go, Cargo's registry, uv, pip, Playwright).
  Keep build output too with `run: crucible-volume add target node_modules` after checkout
  (`command -v crucible-volume` is false on other runners).
- The internet by host name on any TCP port (IPv4; no UDP, no bare IP addresses; private and
  cloud-metadata addresses and mail ports are refused).
- Up to 4 GiB uploaded per job; 20 units at once per workspace (a unit is 2 vCPU: a
  `crucible-16vcpu-arm` job takes 8).

Not available: x86_64, GPUs, macOS, Windows, IPv6, UDP, a static outbound IP, custom runner
groups (`runs-on: {group: …}`), jobs over 6 h 15 min.

## Public repos and forks

- Installing the Runners App on a public repo opts it in; its minutes are billed to you.
- Set Settings, Actions, General, **Require approval for all external contributors**: a run
  waiting for approval boots no VM and costs nothing.
- Fork runs get GitHub's read-only token, no secrets, a fresh VM, and a read-only cache.
- `pull_request_target` runs with your secrets on any runner: never check out and run a fork's
  code in it.
- To keep first-time contributors on GitHub's runners:

```yaml
jobs:
  test:
    runs-on: ${{ (github.event_name != 'pull_request' || contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.pull_request.author_association)) && 'crucible-2vcpu-arm' || 'ubuntu-24.04-arm' }}
```

## Timing, limits and interruptions

- GitHub hands the job over in about 6 to 12 s from queued to running.
- A job runs at most the workspace's **Actions timeout**: 1 h by default, up to 6 h 15 min on
  request (access@crucibl.sh), whatever `timeout-minutes` says.
- A job ended by a spot interruption or an urgent host patch is free: re-run it from GitHub.
- While running, a job holds its worst-case cost from the credit: its class's price for the
  Actions timeout plus $0.51 of egress ($0.66 for `crucible-2vcpu-arm` at 1 h).
