# odoo-keep-staging-alive

A GitHub Action that keeps an odoo.sh staging / development build awake by calling `/web/health` every minute.

odoo.sh puts a build to sleep after ~3 minutes without a request, and a sleeping build runs no scheduled action
(cron). This is handy for long-running crons, such as a data backfill started by hand on a staging.

## How it works

- GitHub's scheduler cannot go below 5 minutes, so each run loops by itself: one request per minute for ~5h50
  (`SECONDS < 21000`, job timeout 358 min, under GitHub's 6 h limit).
- Each run starts the next one itself (`gh workflow run`, at ~5h40), so runs overlap ~10 minutes. The relay does not
  depend on GitHub's scheduler, which may delay a scheduled run by hours or skip it.
- The schedule (`*/30 * * * *`) is only a safety net: a scheduled run exits at once when another run is looping, and
  otherwise starts a new chain (first start, relay that failed, run cancelled by mistake). Worst case: ~30 minutes
  without request, so the build may sleep for that time.
- Configuration lives in your fork, not in the code:
  - secret `KEEPALIVE_URL`: base URL of the build, without `/web/health` (e.g. `https://<build>.dev.odoo.com`)
  - variable `KEEPALIVE_ENABLED`: `true` to allow runs, anything else to skip them (checked when a run starts)
- Cost: public repository = free and unlimited with standard GitHub-hosted runners. On a private repository, a run is
  billed about 1 minute per elapsed minute (about 1 500 minutes per day of keepalive, plus 1 minute per scheduled run
  that exits at once, about 50 per day), against a free quota of 2 000 minutes per month on a Free plan.

## Setup

1. Fork this repository (public forks of public repositories are public; keep it public to stay free).
2. In your fork, open the Actions tab and enable workflows: GitHub disables them in forks by default, scheduled ones
   included.
3. Set the secret and the flag, from a clone of your fork (or add `--repo <you>/odoo-keep-staging-alive` to each
   command, or use Settings > Secrets and variables > Actions in the web UI):

   ```
   gh secret set KEEPALIVE_URL --body "https://<build>.dev.odoo.com"
   gh variable set KEEPALIVE_ENABLED --body true
   ```

4. Start now instead of waiting for the next scheduled slot (up to 30 minutes, or more if GitHub is late):

   ```
   gh workflow run keepalive.yml
   ```

5. Check the logs of your build: a `GET /web/health` 200 should appear every minute, from a GitHub runner IP (these IPs
   change, filter on the path).

Scheduled workflows run from the default branch only. The build URL of a development branch changes at each
rebuild: update the secret (`gh secret set KEEPALIVE_URL`).

## Stop

```
gh variable set KEEPALIVE_ENABLED --body false
gh run list --workflow keepalive.yml --status in_progress   # then: gh run cancel <id>
```

With the flag at `false`, no new run starts (neither the relay nor the schedule), so the chain ends with the run in
progress; cancel it to stop at once. To also drop the scheduled runs from the Actions tab:
`gh workflow disable keepalive.yml` (re-enable with `gh workflow enable keepalive.yml`).
