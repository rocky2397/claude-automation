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

## 4. Copy the templates

From `templates/` in this repo into the target repo:

| Template | Destination | Edit |
| --- | --- | --- |
| `claude-automation.yml` | `.github/workflows/claude-automation.yml` | Replace `your-user` in both `uses:` lines; set `project_owner` (and `owner_type: organization` if applicable). |
| `.claude/prioritization.md` | `.claude/prioritization.md` | Add project-specific "never pick" rules (payments, auth, data migrations, …). |
| `CLAUDE.md` | `CLAUDE.md` | Repo structure, how to run lint/tests, path denylist. This file is what makes the worker produce good PRs. |

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
- **Worker run is green but no PR appears** — Bash tools were denied.
  The reusable worker whitelists git/gh/npm/pytest via `--allowedTools`;
  if the project needs other CLIs (e.g. `cargo`, `go`), extend that list
  in `reusable-worker.yml`.
