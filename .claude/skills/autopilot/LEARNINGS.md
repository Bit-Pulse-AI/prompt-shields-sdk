# Autopilot learnings

Lessons from earlier ticks. Read every tick. Add entries only through the
self-improve PR (see `SKILL.md` phase 7). Newest first. Each entry:

```
## YYYY-MM-DD — <short title>
What happened: <one or two sentences, link journal entries>
Do instead: <the concrete rule>
```

A learning can never override a ground rule in `SKILL.md` or a limit in
`config.md`. If one seems to, it is wrong — propose a config change instead.

## Seed knowledge (from the repo as of 2026-10-10)

- There is no repository-level CI in `.github/workflows/` (only the inherited
  ones under `gateway/.github/`, which GitHub does not run). Local suites in
  `references/qa.md` are the merge gate until CI exists.
- `gateway/` has no aggregate `npm test`; run both `test:gateway` and
  `test:plugins`.
- Collector and integration tests need `PYTHONPATH=packages:packages/collector`;
  SDK tests need `PYTHONPATH=packages/sdk`.
- Integration tests need PostgreSQL with pgvector (`docker compose up -d db`).
  If Docker is unavailable in the session, record it and run the
  database-free suites only — do not merge PRs touching `packages/collector`
  or `packages/db` that tick.
- The README quickstart was fixed to work on a clean machine in PR #26; keep
  it working — it is the post-release smoke test.
