

![Block Clankers](https://i.imgur.com/idhWdxC.jpeg)

# Block Clankers

Auto-block known spam bot accounts from your **personal GitHub account** and/or **every org you admin**. Pulls a community-maintained list and syncs your block lists on a schedule.

Default source: [UnsafeLabs/Bounty-Hunters `clankers.json`](https://github.com/UnsafeLabs/Bounty-Hunters/blob/main/clankers.json).

---

## Quick start - fork & go

1. **Fork this repo.**
2. Create a [Personal Access Token](https://github.com/settings/tokens) (classic) with scopes:
   - `user` - personal-account blocks (required for the default `@me` target)
   - `admin:org` - only if you also want to sync orgs
   - `read:org` - only if you want `@auto` to discover orgs you admin

   (Fine-grained tokens work too - grant the equivalent permissions.)
3. In the fork: **Settings → Secrets and variables → Actions → New repository secret**
   - Name: `BLOCKER_TOKEN`
   - Value: your PAT
4. Go to the **Actions** tab, enable workflows for the fork.
5. Click **Block Clankers → Run workflow** for a first sync, or wait for the schedule.

The bundled workflow at [.github/workflows/block.yml](.github/workflows/block.yml) calls the action with `targets: "@me"` — syncing only your personal account by default. To also cover orgs, change it to `"@auto"` (personal + all admin orgs) or list them explicitly, e.g. `"@me,my-org-1,my-org-2"`.

---

## Use it as an Action in your own repo

If you'd rather not fork, reference it from any repo's workflow:

```yaml
name: Block Clankers
on:
  schedule: [{ cron: "*/30 * * * *" }]
  workflow_dispatch:

jobs:
  block:
    runs-on: ubuntu-latest
    steps:
      - uses: cyrisxd/block-clankers@v1
        with:
          token: ${{ secrets.BLOCKER_TOKEN }}
          targets: "@me"
```

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `token` | yes | - | PAT with scopes matching your targets (`user`, `admin:org`, `read:org`). |
| `targets` | no | `@me` | What to sync. See **Targets** below. |
| `source-url` | no | UnsafeLabs `clankers.json` | URL to a JSON array of `{ "username": "..." }` objects. |
| `dry-run` | no | `false` | Log only, no API writes. |
| `unblock-removed` | no | `false` | Unblock users no longer in source list. Off by default. |

### Targets

A newline- or comma-separated list. Tokens:

| Token | Effect | Endpoint | Required scope |
|---|---|---|---|
| `@me` | Your personal account | `/user/blocks` | `user` |
| `@auto` | `@me` + every org you admin | both | `user`, `admin:org`, `read:org` |
| `org-name` | A specific org | `/orgs/{name}/blocks` | `admin:org` |

Examples:
```yaml
targets: "@me"                            # personal account only (default)
targets: "@auto"                          # personal + every org you admin
targets: "@me,my-org-1,my-org-2"          # personal + two explicit orgs
```

---

## How it works

1. Fetches the source JSON (with retries; fails closed if empty).
2. Expands targets (`@auto` → list orgs via `/user/memberships/orgs`).
3. For each target: lists current blocks, diffs against the source.
4. Writes the delta - `PUT` to block, optional `DELETE` to unblock.
5. Writes a per-target + totals summary to `$GITHUB_STEP_SUMMARY`.

Steady-state runs do 1 LIST per target and 0 writes. Idempotent.

---

## Being gentle on the GitHub API

- **Diff-only writes.** Nothing happens if the list hasn't changed.
- **Throttled writes** - default ~1 request/sec, well under the ~80/min secondary rate-limit ceiling for mutating endpoints.
- **Per-run cap** (`MAX_WRITES_PER_RUN`, default 200, **shared across all targets**) - big first-run backlogs spread over multiple ticks instead of bursting.
- **Retry with `Retry-After`** on 429 / 403-secondary-limit; exponential backoff + jitter on 5xx and network errors.
- **Primary rate-limit floor** - watches `x-ratelimit-remaining` and pauses until reset if it falls below `RATE_LIMIT_FLOOR`.
- **Graceful 404/422** - missing users and "already in state" responses are counted as skipped, not failed.
- **Bounded retries** so a hard outage doesn't loop forever.

### Tuning knobs (env vars)

| Var | Default | Purpose |
|---|---|---|
| `WRITE_DELAY_MS` | `1100` | Delay between writes. |
| `MAX_WRITES_PER_RUN` | `200` | Global cap on writes per invocation. |
| `MAX_RETRIES` | `6` | Per-request retry attempts. |
| `BASE_BACKOFF_SECS` | `2` | Starting exponential backoff. |
| `MAX_BACKOFF_SECS` | `60` | Cap on backoff sleep. |
| `RATE_LIMIT_FLOOR` | `100` | Pause if remaining REST quota drops below this. |
| `REQUEST_TIMEOUT_SECS` | `30` | Per-request timeout. |

Override in a workflow:
```yaml
- uses: cyrisxd/block-clankers@v1
  with:
    token: ${{ secrets.BLOCKER_TOKEN }}
  env:
    WRITE_DELAY_MS: "500"
    MAX_WRITES_PER_RUN: "500"
```

---

## Notes & caveats

- **Org-level blocking only affects orgs whose admin tokens you provide.** GitHub has no global-block API.
- **PAT rotation.** Classic PATs with `admin:org` expire - rotate the secret periodically.
- **Trust the source.** Anyone running this is trusting `source-url`. Pin to a fork or commit if you want full control:
  ```yaml
  source-url: https://raw.githubusercontent.com/UnsafeLabs/Bounty-Hunters/<commit-sha>/clankers.json
  ```
- **`unblock-removed: true`** propagates list removals to every target you manage. Leave off unless you trust the source maintainers to vet removals.

&nbsp;

## Special thanks

A special thanks to Coolify Developer [@heyandras](https://x.com/heyandras/status/2054512710017298463) and [@shadowarcanist](https://x.com/shadowarcanist) for creating and maintaining the source list. Check out their [Clankers Leaderboard](https://clankers-leaderboard.pages.dev/)



## Help me turn coffee into code

[!["Buy Me A Coffee"](https://buymeacoffee.com/assets/img/custom_images/yellow_img.png)](https://buymeacoffee.com/firmvxozh)


## License

MIT
