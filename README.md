# socket-fix-demo

Test repo for `socket fix --autopilot` in GitHub Actions.

## Setup

1. Add `SOCKET_CLI_API_TOKEN` to repo secrets (Settings > Secrets > Actions)
2. Run the "Socket Fix" workflow manually, or wait for the cron schedule
3. Socket will open PRs to fix vulnerable dependencies

## What's in here

Intentionally outdated dependencies with known CVEs to test the fix pipeline.
