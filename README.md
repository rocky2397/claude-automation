# claude-automation

Central reusable workflows for autonomous ticket prioritization and execution
against a GitHub Projects v2 board. Implements the design in
[`SPEC_automation.md`](./SPEC_automation.md).

Each target repo calls two reusable workflows from here instead of
copy-pasting orchestration logic:

- `.github/workflows/reusable-prioritizer.yml` — reads the target repo's
  Project v2 board (scoped to that repo only), decides a `top_pick`,
  `deprioritized`, and `blocked` set per `.claude/prioritization.md`, and
  dispatches by labeling/commenting on issues.
- `.github/workflows/reusable-worker.yml` — triggered when an issue is
  labeled `claude-work`; implements it on a branch and opens a PR.

Both use `anthropics/claude-code-action@v1`, authenticated with a Claude
subscription OAuth token (no API billing). The model defaults to
`claude-fable-5` with `claude-sonnet-5` as automatic fallback; both are
overridable per repo via the `model` and `fallback_model` inputs.

## Installing in a target repo

1. Run `/install-github-app` in the target repo (installs the Claude GitHub
   App and configures secrets).
2. Create a fine-grained PAT (or GitHub App token) with `project` read/write
   scope, store it as the `PROJECT_TOKEN` secret. The default
   `GITHUB_TOKEN` cannot read Projects v2 boards.
3. Run `claude setup-token` locally (requires a Pro/Max subscription) and
   store the result as the `CLAUDE_CODE_OAUTH_TOKEN` secret (org or user
   level, so repos don't duplicate it).
4. Copy the three files from `templates/` into the target repo:
   - `templates/claude-automation.yml` → `.github/workflows/claude-automation.yml`
     (update `your-user` and `project_owner` to point at this repo and at
     the target's own owner/org)
   - `templates/.claude/prioritization.md` → `.claude/prioritization.md`
     (edit ranking criteria for that project)
   - `templates/CLAUDE.md` → `CLAUDE.md` (edit conventions for that project)
5. Set the repository variable `PROJECT_NUMBER` to the board's number.
6. Create labels `claude-work`, `in-review`, `needs-info`.
7. Add "Icebox" and "In Review" columns to the board if not present.
8. Test via `workflow_dispatch` before relying on the cron schedule.

## Versioning

Once stable, pin caller workflows to a tag (`@v1`) instead of `@main`, so a
change here can't silently alter behavior across every installed repo.
Upgrade repos deliberately by bumping the tag.

## Merge gate

Claude opens PRs; a human merges. Never skip this in phase 1 — see
`SPEC_automation.md` §7 for the phased auto-merge plan.

## Guardrails already built in

- WIP limit (default 1) enforced by the prioritizer.
- `--max-turns` caps on both workflows.
- Worker never opens a PR on failing tests; it comments instead.
- Path denylist (`.github/workflows/`, `infra/`, `migrations/`) belongs in
  each target repo's `CLAUDE.md`.
- Concurrency group on the worker per issue number.
- Board items are filtered to the calling repo, and bot-authored issues are
  dropped, before Claude ever sees them.
