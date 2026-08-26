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

## Run log and nightly report

Every job in all three reusable workflows ends with an `if: always()` **Emit
run log** step. It records a semantic `outcome` — what the job actually
accomplished — alongside GitHub's own exit code, and uploads it as an
`automation-log-*` artifact together with the Claude SDK message log. This
distinction matters: a prioritizer run that short-circuits on the WIP limit
and a worker run that opens no PR both report `conclusion: success`, and
without the `outcome` field they are indistinguishable from real work.

`.github/workflows/nightly-report.yml` runs here at 21:00 UTC daily. It walks
every repo in the `AUTOMATION_REPOS` variable, collects that day's runs,
artifacts, error annotations, and issue/PR movement, then has Claude write a
plain-text review to `reports/YYYY-MM-DD.txt`. The raw events land in
`logs/YYYY-MM-DD.jsonl`. Both are committed to this repo.

The report has a fixed shape; section 4, **silent failures**, is the one to
read first — jobs that went green while accomplishing nothing. If the reviewer
step itself fails, a mechanical fallback report is written from the same
evidence, so no day goes unrecorded.

This repo needs two secrets of its own for the nightly job:
`CLAUDE_CODE_OAUTH_TOKEN` and `PROJECT_TOKEN` (the classic PAT, used to read
runs and artifacts across target repos). Add a repo to the review with:

```bash
gh variable set AUTOMATION_REPOS --repo <you>/claude-automation \
  --body "<you>/repo-a <you>/repo-b"
```

Run it for a past day, or re-run a day, with:

```bash
gh workflow run nightly-report.yml --repo <you>/claude-automation -f date=2026-08-26
```

## `--allowedTools` is an allowlist, not an addition

The single most damaging bug this system has had: `--allowedTools` **replaces**
the default tool set rather than extending it. Listing only `Bash(...)`
patterns removed `Read`, `Write`, and `Edit`, so the prioritizer could not read
its own `board.json` and the worker could not edit a file. Both failed on every
run for four weeks while the board sat idle. Any edit to an `--allowedTools`
list must keep the file tools (`Read,Glob,Grep`, plus `Write,Edit` for the
worker) listed first.

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
- Board items are filtered to the calling repo, and closed issues plus
  bot-authored issues are dropped, before Claude ever sees them. The
  closed-issue filter matters more than it looks: a board's Status field does
  not update when a PR closes an issue, so without it the prioritizer picks
  work that already shipped.
- Every job emits a semantic outcome to the run log, so a green run that did
  no work is visible rather than silent.
