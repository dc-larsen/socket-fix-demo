# socket-fix-demo

Reference repo for running `socket fix` in GitHub Actions, constrained to
minor and patch upgrades with auto-merge on.

The dependencies here are intentionally outdated and carry known CVEs, so the
workflow has real work to do on the first run.

## Fork and run it

1. Fork this repo.
2. Change `orgSlug` in `socket.json` to your Socket org slug.
3. Add repo secrets under Settings > Secrets and variables > Actions:
   - `SOCKET_CLI_API_TOKEN` - a Socket API token with `full-scans:create` and `packages:list`
   - `SOCKET_FIX_PAT` - a GitHub PAT (or App token) with `contents: write` and `pull-requests: write`
4. Turn on **Settings > General > Allow auto-merge**.
5. Run the "Socket Fix" workflow from the Actions tab, or wait for the cron.

Steps 3 and 4 are the two that quietly break things. See below.

## The flags

| Flag | What it does |
|---|---|
| `--no-major-updates` | Restricts upgrades to minor and patch bumps, for direct and transitive dependencies both. Some CVEs will produce no PR at all, because the only version that clears them is a major bump. |
| `--autopilot` | Turns on GitHub's native auto-merge on the PRs Socket opens, so required status checks become the approval gate instead of a person. |
| `--minimum-release-age 7d` | Skips versions published in the last week, so you never auto-merge into a release nobody has looked at yet. |
| `--pr-limit 5` | Counts Socket Fix PRs already open and backs off, so the queue can't run away from you. |

For a dry run that opens nothing, drop `--autopilot` and add
`--no-apply-fixes --output-file suggested-fixes.json`. That gets you a volume
estimate before anything merges.

For a monorepo, scope with `--include "packages/*"`.

## Two things that stall it silently

**"Allow auto-merge" must be on.** Without it, `--autopilot` can't arm and the
PRs sit open indefinitely. This repo demonstrated that the hard way: the first
run opened ten fix PRs in March and every one of them stayed open for five
months, because the setting was off.

**Use a PAT, not the default `GITHUB_TOKEN`.** PRs opened with the default token
do not trigger other workflows, so your required checks never run and auto-merge
waits forever. The workflow prefers `SOCKET_FIX_PAT` and falls back to
`GITHUB_TOKEN` so it still runs without one, but auto-merge won't complete.

## The fix engine can trip your own policy

`socket fix` installs its fix engine (`@coana-tech/cli`) through the
Socket-wrapped package manager. That engine ships frequent releases, so an org
policy that blocks `recentlyPublished` will stop the run before a single fix is
computed. The failure looks like this:

```
@coana-tech/cli@15.10.36:
  recentlyPublished (moderate; blocked)
Socket pnpm exiting due to risks.
```

The workflow sets `SOCKET_CLI_ACCEPT_RISKS: '1'` on the fix step to allow it.
That override applies to installing the first-party engine, not to how your own
dependencies get evaluated. Check this before a pilot: if your org blocks
`recentlyPublished`, you will hit it on the first run.

## Scheduled workflows expire

GitHub disables cron workflows after ~60 days of repo inactivity. That happened
here between March and June. If the schedule goes quiet, re-enable the workflow
in the Actions tab and push a commit.

## pnpm note

Pin your package manager with a top-level `"packageManager"` field, as this repo
does. Avoid `devEngines.packageManager`: on pnpm 11 it changes `pnpm-lock.yaml`
to a multi-document file, which affects how the lockfile is parsed during
scanning.
