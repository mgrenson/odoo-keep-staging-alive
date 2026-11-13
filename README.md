# odoo-keep-staging-alive

Keeps an odoo.sh staging / dev build awake (it sleeps after ~3 minutes without a request, and a sleeping build runs
no scheduled action) by calling `/web/health` every minute.

## How it works

- GitHub's scheduler cannot go below 5 minutes, so each run loops by itself: one request per minute for ~5h55
  (`SECONDS < 21300`, job timeout 358 min, under GitHub's 6 h limit).
- A new run is scheduled every 5 hours (`0 */5 * * *`): runs overlap by about 1 hour, so a delayed or skipped
  schedule does not leave a gap.
- Cost: ~1.2 billed minutes per elapsed minute (private repo: 2000 free minutes per month, so keep it for a day or
  two at a time, or make the repo public).

## Setup

```
gh secret set KEEPALIVE_URL --repo mgrenson/odoo-keep-staging-alive   # base URL, e.g. https://<build>.dev.odoo.com
gh variable set KEEPALIVE_ENABLED --body true --repo mgrenson/odoo-keep-staging-alive
gh workflow run keepalive.yml --repo mgrenson/odoo-keep-staging-alive   # start now instead of waiting for the schedule
```

The build URL changes at each rebuild of a dev branch: update the secret (`gh secret set KEEPALIVE_URL`).

## Stop

The flag is read when a run starts, so setting it to `false` does not end a run already looping:

```
gh variable set KEEPALIVE_ENABLED --body false --repo mgrenson/odoo-keep-staging-alive
gh workflow disable keepalive.yml --repo mgrenson/odoo-keep-staging-alive
gh run list --workflow keepalive.yml --status in_progress --repo mgrenson/odoo-keep-staging-alive   # then: gh run cancel <id>
```

Re-enable with `gh workflow enable keepalive.yml` and the flag back to `true`.
