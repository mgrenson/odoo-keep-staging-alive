# odoo-keep-staging-alive

A GitHub Action that keeps an odoo.sh staging / development build awake by calling `/web/health` every minute.

odoo.sh puts a build to sleep after ~3 minutes without a request, and a sleeping build runs no scheduled action
(cron). This is handy for long-running crons, such as a data backfill started by hand on a staging.

## How it works

- GitHub's scheduler cannot go below 5 minutes, so each run loops by itself: one request per minute for ~5h55
  (`SECONDS < 21300`, job timeout 358 min, under GitHub's 6 h limit).
- A new run is scheduled every 5 hours (`0 */5 * * *`): runs overlap by about 1 hour, so a delayed or skipped
  schedule does not leave a gap.
- Configuration lives in your fork, not in the code:
  - secret `KEEPALIVE_URL`: base URL of the build, without `/web/health` (e.g. `https://<build>.dev.odoo.com`)
  - variable `KEEPALIVE_ENABLED`: `true` to allow runs, anything else to skip them
- Cost: public repository = free and unlimited with standard GitHub-hosted runners. On a private repository, a run is
  billed ~1.2 minutes per elapsed minute (about 1 700 minutes per day of keepalive), against a free quota of
  2 000 minutes per month on a Free plan.

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

4. Start now instead of waiting for the next scheduled slot:

   ```
   gh workflow run keepalive.yml
   ```

5. Check the logs of your build: a `GET /web/health` 200 should appear every minute, from a GitHub runner IP (these IPs
   change, filter on the path).

Scheduled workflows run from the default branch only. The build URL of a development branch changes at each
rebuild: update the secret (`gh secret set KEEPALIVE_URL`).

## Stop

The flag is read when a run starts, so setting it to `false` does not end a run already looping. To stop completely:

```
gh variable set KEEPALIVE_ENABLED --body false
gh workflow disable keepalive.yml
gh run list --workflow keepalive.yml --status in_progress   # then: gh run cancel <id>
```

Re-enable with `gh workflow enable keepalive.yml` and the flag back to `true`.
