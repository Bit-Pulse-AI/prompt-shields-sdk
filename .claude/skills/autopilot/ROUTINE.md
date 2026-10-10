# Setting up the autopilot Routine

This file is for the human operator. It is not read during a tick.

## 1. Prepare the repository

- Create the labels listed in `config.md` (or let the first tick create them).
- Label a few small, well-specified issues `autopilot:ready`. Autopilot only
  ever picks up issues a human has labelled.
- Recommended before relying on auto-merge: add a root CI workflow
  (`.github/workflows/ci.yml`) that runs the suites in `references/qa.md`, and
  make it a required check on `main`. Until then the merge gate is autopilot's
  own test runs.
- Recommended: branch protection on `main` that forbids force-push.

## 2. Cloud environment

In claude.ai/code, use an environment that:

- has this repository attached with push access,
- allows network access to PyPI and npm (for installing test dependencies),
- runs a setup script that installs Python 3.11+, Node 20+ and, if available,
  Docker — without Docker the integration suite is skipped and DB-touching PRs
  will not be merged.

## 3. The Routine (dispatcher)

A Routine that starts a fresh session on each firing gets no repository
attached (and cannot attach one), so the schedule is set up as a
**dispatcher**:

- Routine **prompt-shields autopilot (hourly dispatcher)** fires hourly into a
  long-lived Claude session that has the `create_session` tool.
- Each firing, that session archives the previous finished tick and calls
  `create_session` with this repository as `source_url`, `model:
  claude-sonnet-5-5`, tag `autopilot`, and the **tick prompt**.
- The tick prompt is stored inside the Routine's own prompt, between
  `=== TICK PROMPT START ===` and `=== TICK PROMPT END ===`.

## 4. What to edit where

| You want to change | Edit |
|---|---|
| How often it runs | The Routine's schedule in claude.ai → Routines |
| Which model ticks use | `model:` line in the Routine prompt |
| What each tick is told up front | Text between the TICK PROMPT markers in the Routine prompt |
| What a tick actually does (phases, rules) | `SKILL.md` and `references/*.md` in this directory |
| Limits, labels, risk paths, release policy | `config.md` |
| Lessons | `LEARNINGS.md` |

Skill files are read fresh from GitHub on every tick, so a pushed or merged
edit takes effect on the next tick. Until PR #49 merges, ticks read them from
the `claude/vigilant-meitner-fehfx1` branch; after it merges, from `main`.
Autopilot never merges changes to this directory itself.

## 5. Operating it

- **Watch:** the "Autopilot journal" issue has one entry per tick.
- **Pause:** add `autopilot:pause` to the journal issue. Remove it to resume.
- **Approve risky work:** add `autopilot:merge-ok` to a high-risk PR or a
  release PR.
- **Unblock:** answer the question on an `autopilot:needs-human` item, then
  remove the label.
- **Improve the loop:** review and merge (or close) the `autopilot:self-improve`
  PR. This is the only way the skill changes itself.
- **Start slow:** consider running every 3 hours with `max_merges_per_tick: 0`
  for the first few days, read the reviews it writes, then turn merging on.
