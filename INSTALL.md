# Installing Claude Automation in a Repo

The global setup (done once per GitHub account) is assumed:

- This repo is shared with your other repos: `gh api -X PUT repos/<you>/claude-automation/actions/permissions/access -f access_level=user`
- The [Claude GitHub App](https://github.com/apps/claude) is installed (all repos).
- You have a classic PAT with `repo,project` scopes (fine-grained PATs cannot access user-owned Projects v2).
- You have a subscription OAuth token from `claude setup-token`. Careful when copying it from the terminal: it is ~108 characters and wraps across two lines.

Everything below is per repo.

```bash
REPO="<you>/<your-repo>"
```

## 1. Project board

One board can serve many repos — the prioritizer only considers issues
belonging to the repo it runs in. Reuse an existing board or create one:

```bash
gh project create --owner <you> --title "<your-repo>"
gh project link <BOARD_NUMBER> --owner <you> --repo "$REPO"
```

Optionally rename the Status columns in the web UI to:
Todo / In Progress / In Review / Icebox / Done.

## 2. Variable and secrets

```bash
gh variable set PROJECT_NUMBER --repo "$REPO" --body "<BOARD_NUMBER>"
printf '%s' '<sk-ant-oat01-...>' | gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo "$REPO"
printf '%s' '<ghp_...>'          | gh secret set PROJECT_TOKEN --repo "$REPO"
```

## 3. Labels

```bash
gh label create claude-work --repo "$REPO" --color 1D76DB --description "Selected by prioritizer; worker will pick up"
gh label create in-review   --repo "$REPO" --color 0E8A16 --description "PR opened, awaiting human review"
gh label create needs-info  --repo "$REPO" --color D93F0B --description "Blocked; prioritizer asked a question"
```

Only needed if you're enabling tester feedback (see step 4 and "Daily use" below):

```bash
gh label create feedback-raw     --repo "$REPO" --color FBCA04 --description "Raw tester feedback, not yet triaged"
gh label create feedback-triaged --repo "$REPO" --color C5DEF5 --description "Reformatted into ticket format by triage"
```

## 4. Copy the templates

From `templates/` in this repo into the target repo:

| Template | Destination | Edit |
| --- | --- | --- |
| `claude-automation.yml` | `.github/workflows/claude-automation.yml` | Replace `your-user` in all three `uses:` lines; set `project_owner` (and `owner_type: organization` if applicable). |
| `.claude/prioritization.md` | `.claude/prioritization.md` | Add project-specific "never pick" rules (payments, auth, data migrations, …). |
| `CLAUDE.md` | `CLAUDE.md` | Repo structure, how to run lint/tests, path denylist. This file is what makes the worker produce good PRs. |
| `.claude/ticket-format.md` | `.claude/ticket-format.md` | Optional but recommended: the ticket shape the prioritizer/worker expect, referenced by the triage step. |
| `.github/ISSUE_TEMPLATE/tester-feedback.yml` | `.github/ISSUE_TEMPLATE/tester-feedback.yml` | Only if enabling tester feedback (see "Daily use"). Free-text form testers use instead of writing a ticket by hand. |

Make sure `.gitignore` does not ignore `.claude/` — narrow it to
`.claude/settings.local.json` if needed.

## 5. Commit and smoke-test

```bash
git add .github/workflows/claude-automation.yml .claude/prioritization.md CLAUDE.md
git commit -m "Add Claude automation" && git push
gh workflow run claude-automation.yml --repo "$REPO"
```

With an empty board the run should finish green with `top_pick: null`.

## Daily use

File issues **in the target repo**, add them to the board, set Status =
Todo. The cron (07:00/13:00 UTC weekdays) picks at most one ticket,
labels it `claude-work`, and the worker opens a PR and moves the issue
to `in-review`. Merge the PR to close the issue. `workflow_dispatch`
triggers a run any time.

**Tester feedback (optional).** If you've copied `tester-feedback.yml`
and created the `feedback-raw`/`feedback-triaged` labels, testers can
file feedback via the repo's "Feedback (for testers)" issue form — plain
free text, no ticket format required. It's auto-labeled `feedback-raw`;
the triage step rewrites it into the standard ticket shape (label flips
to `feedback-triaged`) before the prioritizer ever sees it, and it can
only rewrite text — it cannot close or reject a ticket. Tester issues
still need to land on the Project board the same way any other issue
does: either add them manually, or turn on the board's built-in "auto-add
items" workflow in the Project's own settings (this repo's automation
does not do that step for you).

## Nightly report setup (once, on the central repo)

The per-repo steps above are all the automation needs to run. The nightly
review is separate and lives entirely in `claude-automation`:

```bash
CENTRAL="<you>/claude-automation"

# Which repos the report covers (space-separated).
gh variable set AUTOMATION_REPOS --repo "$CENTRAL" --body "<you>/repo-a <you>/repo-b"

# The same two secrets the target repos use. PROJECT_TOKEN needs the classic
# PAT here too: it reads workflow runs and downloads artifacts cross-repo.
printf '%s' '<sk-ant-oat01-...>' | gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo "$CENTRAL"
printf '%s' '<ghp_...>'          | gh secret set PROJECT_TOKEN --repo "$CENTRAL"

# Smoke-test against today.
gh workflow run nightly-report.yml --repo "$CENTRAL"
```

Reports land in `reports/YYYY-MM-DD.txt`, raw events in
`logs/YYYY-MM-DD.jsonl`, both committed to the central repo. Re-running the
same day overwrites that day rather than appending.

Optional overrides, as repo variables on the central repo: `REPORT_MODEL`
(default `claude-sonnet-5`) and `REPORT_MAX_TURNS` (default `40`).

## Troubleshooting

- **`startup_failure` with no logs or annotations** — the caller jobs are
  missing their explicit `permissions:` blocks (a called workflow may
  never exceed the caller's permissions), or `PROJECT_NUMBER` is passed
  without `fromJSON`. The template includes both fixes; don't strip them.
- **HTTP 422 "workflow was not found" on dispatch** — this central repo
  isn't shared: re-run the `actions/permissions/access` command above.
- **401 "Claude Code is not installed on this repository"** — install the
  GitHub App for that repo.
- **401 Invalid bearer token** — the OAuth secret was truncated when
  copied; re-run `claude setup-token` and paste the full wrapped token.
- **Prioritizer never picks a ticket that is on the board** — the issue
  lives in a different repo than the one the workflow runs in; transfer
  it (`gh issue transfer`).
- **Worker run is green but no PR appears** — tools were denied. Check the
  run's job summary or `logs/*.jsonl` for `claude.permission_denials`; a
  non-zero count names the cause. Remember that `--allowedTools` *replaces*
  the default tool set: the list must keep `Read,Write,Edit,Glob,Grep` as
  well as the Bash patterns. If the project needs another CLI (e.g. `cargo`,
  `go`), extend the list in `reusable-worker.yml` without dropping those.
- **Prioritizer fails with `--json-schema was provided but Claude did not
  return structured_output`** — same cause. A `result` block showing
  `num_turns: 1` with `is_error: true` means Claude died before doing any
  work, usually because it was told to read a file it had no `Read` tool for.
- **Every scheduled run is green but nothing ever gets picked** — the WIP
  limit is short-circuiting the job. An open issue still labelled
  `claude-work` blocks all future picks. The run log records this as
  `outcome: wip_limit_reached` rather than letting it pass as a success.
