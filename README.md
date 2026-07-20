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
- `.github/workflows/reusable-triage.yml` — triggered when a new issue
  carries label `feedback-raw` (applied by the optional
  `tester-feedback.yml` issue form); rewrites free-text tester feedback
  into the ticket format the prioritizer/worker expect, using a cheap
  model with no Bash/`gh` access — it can only rewrite the issue's title
  and body and swap the label to `feedback-triaged`, never close or judge
  it. The prioritizer ignores anything still labeled `feedback-raw`.

Both use `anthropics/claude-code-action@v1`, authenticated with a Claude
subscription OAuth token (no API billing). The model defaults to
`claude-fable-5` with `claude-sonnet-5` as automatic fallback; both are
overridable per repo via the `model` and `fallback_model` inputs.

## Installing in a target repo

See [INSTALL.md](INSTALL.md) for the full per-repo checklist with commands
and a troubleshooting section. In short: create/link a Projects v2 board,
set `PROJECT_NUMBER` + the two secrets (`CLAUDE_CODE_OAUTH_TOKEN` from
`claude setup-token`, `PROJECT_TOKEN` — a **classic** PAT with
`repo,project` scopes; fine-grained PATs and the default `GITHUB_TOKEN`
cannot access user-owned Projects v2 boards), create the three labels,
copy the three files from `templates/`, and smoke-test via
`workflow_dispatch`.

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
