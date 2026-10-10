# Autopilot configuration

Change these values only through a PR a human merges. Autopilot may *propose*
changes here in its self-improve PR, within the limits in `SKILL.md`.

## Budgets (per tick)

| Key | Value | Notes |
|---|---|---|
| `max_new_issues_per_tick` | 1 | New implementation PRs started per tick. |
| `max_open_autopilot_prs` | 3 | Stop picking up issues at this many open autopilot PRs. |
| `max_reviews_per_tick` | 3 | |
| `max_merges_per_tick` | 0 | Merging off until the hardening from PR #49's review lands. Raise to 2 to turn on. |
| `max_minutes_per_issue` | 35 | Then push a draft and label `autopilot:blocked`. |
| `max_fix_attempts` | 3 | Consecutive ticks a PR may stay red before `autopilot:needs-human`. |

## Timing

| Key | Value | Notes |
|---|---|---|
| `lock_ttl` | 90 minutes | A `LOCK` older than this is stale and may be taken over. |
| `merge_cooloff` | 2 hours | Minimum time a PR is open before autopilot merges it. |
| `journal_lookback` | 24 entries | Roughly the last day of ticks. |
| `learning_threshold` | 2 | Repeats of a surprise before it becomes a lesson. |

## Labels

Create any that are missing on first use.

| Label | Meaning |
|---|---|
| `autopilot:ready` | Human-approved: autopilot may pick this issue up. |
| `autopilot:claimed` | Autopilot is working on it. |
| `autopilot:blocked` | Autopilot got stuck; details in the PR or issue. |
| `autopilot:needs-human` | A decision or access only a human can give. Autopilot will not touch it again until the label is removed. |
| `autopilot` | PR opened by autopilot. |
| `autopilot:merge-ok` | Human allows autopilot to merge this PR despite high risk or human authorship. |
| `autopilot:self-improve` | PR changing the autopilot skill. Human merge only. |
| `autopilot:release` | Release PR. |
| `autopilot:journal` | The single journal issue. |
| `autopilot:pause` | On the journal issue: kill switch. Every tick exits immediately. |
| `regression` | Found by autopilot QA. |

Autopilot only picks up issues a human has labelled `autopilot:ready`. It may
*suggest* the label in a comment on issues it thinks are a good fit, but never
applies it itself.

## Risk classes

A PR is **high** risk if it touches any of these, otherwise **low**:

- `packages/db/alembic/**`, `packages/db/models.py` — schema and migrations
- Telemetry payload shapes: `packages/sdk/prompt_shields/**` event/telemetry
  fields, `gateway/src/middlewares/ps-telemetry.ts` — privacy contract
- Upstream Portkey files in `gateway/` (anything not listed as ours in
  `gateway/FORK_NOTICE.md`) — fork divergence
- Authentication / API keys: collector auth, `api_key_fingerprint`
- Public SDK API removals or signature changes — semver
- `LICENSE`, `NOTICE`, `gateway/LICENSE`, `SECURITY.md`
- `.github/**`, `docker-compose.yml`, `Dockerfile`s, `pyproject.toml`
  dependency changes, `package.json` / lockfile dependency changes
- `.claude/**`
- More than 400 changed lines excluding tests and lockfiles

High-risk PRs need `autopilot:merge-ok` from a human before autopilot merges.

## Release policy

| Key | Value | Notes |
|---|---|---|
| `release_package` | `packages/sdk` (`prompt-shields`) | Only the SDK is versioned for release. |
| `release_tag_format` | `sdk-v{version}` | |
| `release_min_interval` | 7 days | Since the last `sdk-v*` tag. |
| `release_min_changes` | 3 | User-facing CHANGELOG `[Unreleased]` bullets. |
| `release_requires_human` | true | Release PR needs `autopilot:merge-ok`. Set to false only once CI and QA have a track record. |
| `release_publish` | false | Publishing to PyPI needs credentials autopilot does not hold. Tagging only. |
