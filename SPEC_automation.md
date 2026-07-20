# SPEC: Autonomous Ticket Prioritization and Execution Agent (GitHub Projects)

## 1. Summary

A per-repository automation system that reads all tickets from a GitHub Projects v2 kanban board on a schedule, prioritizes them against project-specific criteria, explicitly decides what NOT to work on, and dispatches the highest-priority ticket to Claude Code for autonomous implementation. Claude Code implements the ticket and opens a pull request. The system runs without manual intervention up to the merge gate.

The system is built almost entirely from existing components. The custom work is one orchestrator workflow, one prioritization prompt file, and configuration.

## 2. Components

| Component | Type | Source |
|---|---|---|
| Worker (ticket → PR) | Existing product | `anthropics/claude-code-action@v1` |
| Triage (raw feedback → ticket) | Custom, shared | One reusable GitHub Actions workflow (`reusable-triage.yml`) |
| Orchestrator (board → decision → dispatch) | Custom | One GitHub Actions workflow (cron) |
| Prioritization policy | Custom, per repo | `.claude/prioritization.md` |
| Coding standards | Custom, per repo | `CLAUDE.md` at repo root |
| Board access | Existing API | GitHub GraphQL API (Projects v2), via `gh api graphql` |
| Auth | Existing | Claude GitHub App + `CLAUDE_CODE_OAUTH_TOKEN` secret (Pro/Max subscription, generated with `claude setup-token`) |
| Model | Existing | `claude-fable-5`, with `claude-sonnet-5` as automatic fallback |

## 3. Architecture

Ahead of the prioritizer, an optional triage stage handles tickets filed
by non-technical testers. Testers use a "Feedback (for testers)" GitHub
Issue Form (two free-text boxes) instead of writing a ticket by hand; the
form auto-applies the `feedback-raw` label. `reusable-triage.yml` fires
on `issues: opened` for that label, sends the raw text plus
`.claude/ticket-format.md` to a cheap model, and gets back structured
`{title, body}` JSON only — the LLM has no Bash/`gh` access and cannot
close, reject, or judge the ticket. A separate deterministic step applies
the rewrite via `gh issue edit` and flips the label to `feedback-triaged`.
The prioritizer's board-fetch step drops any item still labeled
`feedback-raw`, so a ticket is never evaluated until triage has rewritten
it. From there it flows through the normal pipeline below like any other
ticket.

```
┌─────────────────────────────────────────────────────────┐
│  Scheduled workflow: prioritizer.yml (cron, e.g. 2x/day) │
│                                                          │
│  Step 1  gh api graphql → fetch all Project v2 items     │
│          (title, body, status column, labels, assignee,  │
│           linked PRs, updatedAt)                         │
│                                                          │
│  Step 2  claude-code-action in automation mode           │
│          prompt = board JSON + .claude/prioritization.md │
│          output = structured JSON decision               │
│                                                          │
│  Step 3  Parse decision:                                 │
│          - top_pick → add label "claude-work"            │
│          - deprioritized[] → move to "Icebox" column,    │
│            comment with reasoning                        │
│          - blocked[] → comment asking for missing info   │
└──────────────────────┬───────────────────────────────────┘
                       │ label "claude-work" applied
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Worker workflow: claude-worker.yml                      │
│  trigger: issues.labeled == "claude-work"                │
│                                                          │
│  claude-code-action@v1 reads the issue, reads CLAUDE.md, │
│  implements the change on a branch, runs tests,          │
│  opens a PR linked to the issue,                         │
│  moves board item to "In Review"                         │
└──────────────────────┬───────────────────────────────────┘
                       ▼
              Merge gate (see §7)
```

## 4. Repository files to create

### 4.1 `.github/workflows/prioritizer.yml`

```yaml
name: Ticket Prioritizer
on:
  schedule:
    - cron: "0 7,13 * * 1-5"   # 07:00 and 13:00 UTC, weekdays
  workflow_dispatch: {}         # manual trigger for testing

permissions:
  contents: read
  issues: write

jobs:
  prioritize:
    runs-on: ubuntu-latest
    # Guard: skip if a claude-work item is already in flight
    steps:
      - uses: actions/checkout@v4

      - name: Check WIP limit
        id: wip
        run: |
          COUNT=$(gh issue list --label claude-work --state open --json number --jq 'length')
          echo "count=$COUNT" >> "$GITHUB_OUTPUT"
        env:
          GH_TOKEN: ${{ secrets.PROJECT_TOKEN }}

      - name: Fetch project board
        if: steps.wip.outputs.count == '0'
        id: board
        run: |
          gh api graphql -f query='
            query($org: String!, $number: Int!) {
              organization(login: $org) {
                projectV2(number: $number) {
                  items(first: 100) {
                    nodes {
                      id
                      fieldValueByName(name: "Status") {
                        ... on ProjectV2ItemFieldSingleSelectValue { name }
                      }
                      content {
                        ... on Issue {
                          number title body url updatedAt
                          labels(first: 10) { nodes { name } }
                          assignees(first: 5) { nodes { login } }
                        }
                      }
                    }
                  }
                }
              }
            }' -f org="$ORG" -F number="$PROJECT_NUMBER" > board.json
        env:
          GH_TOKEN: ${{ secrets.PROJECT_TOKEN }}
          ORG: your-org-or-username
          PROJECT_NUMBER: 1

      - name: Prioritize with Claude
        if: steps.wip.outputs.count == '0'
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: |
            Read board.json (GitHub Project items) and the policy in
            .claude/prioritization.md.

            Decide:
            1. top_pick: exactly one issue number to work on now, or null
               if nothing is ready.
            2. deprioritized: issue numbers that should NOT be worked on,
               each with a one-sentence reason.
            3. blocked: issue numbers that lack the information needed to
               implement them, each with the specific question to ask.

            Respond ONLY with JSON:
            {"top_pick": {"number": N, "reason": "..."},
             "deprioritized": [{"number": N, "reason": "..."}],
             "blocked": [{"number": N, "question": "..."}]}
          claude_args: "--max-turns 6 --model claude-fable-5 --fallback-model claude-sonnet-5"

      - name: Dispatch decision
        if: steps.wip.outputs.count == '0'
        run: |
          # parse decision.json produced by previous step
          TOP=$(jq -r '.top_pick.number // empty' decision.json)
          if [ -n "$TOP" ]; then
            gh issue edit "$TOP" --add-label "claude-work"
            gh issue comment "$TOP" --body "Prioritizer selected this ticket. Reason: $(jq -r '.top_pick.reason' decision.json)"
          fi
          jq -c '.blocked[]?' decision.json | while read -r item; do
            NUM=$(echo "$item" | jq -r '.number')
            Q=$(echo "$item" | jq -r '.question')
            gh issue comment "$NUM" --body "Blocked by prioritizer. Missing info: $Q"
            gh issue edit "$NUM" --add-label "needs-info"
          done
          jq -c '.deprioritized[]?' decision.json | while read -r item; do
            NUM=$(echo "$item" | jq -r '.number')
            R=$(echo "$item" | jq -r '.reason')
            gh issue comment "$NUM" --body "Deprioritized: $R"
          done
        env:
          GH_TOKEN: ${{ secrets.PROJECT_TOKEN }}
```

Note. Resolved in the implementation: pass a `--json-schema '<inline schema>'` flag in `claude_args`; the action validates Claude's answer against it and exposes the result as the `structured_output` action output. The "Dispatch decision" step reads `steps.<id>.outputs.structured_output` (via an env var) and writes it to `decision.json`, rather than relying on Claude writing the file itself.

### 4.2 `.github/workflows/claude-worker.yml`

```yaml
name: Claude Worker
on:
  issues:
    types: [labeled]

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  implement:
    if: github.event.label.name == 'claude-work'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: |
            Implement issue #${{ github.event.issue.number }}:
            "${{ github.event.issue.title }}"

            Requirements:
            - Follow CLAUDE.md conventions strictly.
            - Make the minimal change that fully resolves the issue.
            - Write or update tests for the change.
            - Run the test suite. Do not open a PR if tests fail;
              instead comment on the issue with the failure.
            - Open a PR titled "fix: <issue title> (#N)" that closes
              the issue, with a summary of the approach and any
              tradeoffs.
            - Remove the "claude-work" label and add "in-review".
          claude_args: "--max-turns 30 --model claude-fable-5 --fallback-model claude-sonnet-5"
```

### 4.3 `.claude/prioritization.md` (per project, this is where project-specificity lives)

```markdown
# Prioritization Policy for <PROJECT NAME>

## Ranking criteria, in order
1. Production bugs affecting users (label: bug + prod)
2. Tickets blocking other tickets (referenced by "blocked by" in bodies)
3. Small, well-specified features with clear acceptance criteria
4. Tech debt with a measurable payoff stated in the ticket
5. Everything else

## Hard rules
- Never pick a ticket without acceptance criteria. Classify as blocked
  and ask for them.
- Never pick tickets labeled "discuss", "design", or "wontfix".
- Never pick tickets touching auth, payments, or data migrations.
  These require human implementation. Classify as deprioritized with
  reason "requires human owner".
- Prefer tickets estimated ≤ 1 day (label: size/S or size/M).
- Max one ticket in flight at a time (WIP limit enforced by workflow).

## Definition of "not worth working on"
- Stale (no update in 60 days) and no priority label → deprioritize.
- Duplicate of an open PR or issue → deprioritize with link.
- Vague one-liners → blocked, ask a concrete clarifying question.
```

### 4.4 `CLAUDE.md` (repo root, per project)

Coding conventions, test commands, branch naming, PR format. The worker reads this automatically. Example skeleton:

```markdown
# Project conventions
- Stack: <e.g. FastAPI + PostgreSQL + React>
- Run tests: `make test`
- Branch naming: claude/<issue-number>-<slug>
- Never modify: infra/, .github/workflows/, migrations/
- All new endpoints need an integration test.
```

### 4.5 `.github/workflows/reusable-triage.yml` (central, shared)

Reformats non-technical tester feedback into the ticket shape defined by
`.claude/ticket-format.md`, before the prioritizer ever considers it.

- Trigger: `issues: opened` where the issue carries label `feedback-raw`
  (applied automatically by the `.github/ISSUE_TEMPLATE/tester-feedback.yml`
  form).
- Model: cheap (`claude-haiku-4-5` by default) — this step only rewrites
  text, it does not need a strong reasoning model.
- Tool access: none. No `--allowedTools` is passed, so Bash/`gh` stay
  denied by default in CI. The model returns structured `{title, body}`
  JSON via `--json-schema`; a separate deterministic step applies it with
  one `gh issue edit` call and swaps the label to `feedback-triaged`.
  There is no `gh issue close` (or delete) anywhere in the workflow — it
  is structurally, not just by prompt instruction, incapable of killing a
  ticket.
- The rewritten body always ends with `## Original report`, quoting the
  tester's exact original title and body, so nothing is lost in
  translation and a human can always check the reformatting against the
  source.
- Uses `github.token`, not `PROJECT_TOKEN`: it only edits the issue that
  triggered it, never queries the Projects v2 board. This also means the
  label swap it performs does not itself trigger a new workflow run
  (`GITHUB_TOKEN`-authored events don't fire new runs), avoiding a loop.

## 5. Setup steps

1. In Claude Code terminal, run `/install-github-app` in each target repo. This installs the Claude GitHub App and configures secrets.
2. Run `claude setup-token` locally (Pro/Max subscription) and store the result as the `CLAUDE_CODE_OAUTH_TOKEN` secret, so runs bill against the subscription instead of API usage.
3. Create a fine-grained PAT (or GitHub App token) with `project` read/write scope and store as `PROJECT_TOKEN`. The default `GITHUB_TOKEN` cannot read Projects v2 boards.
4. Add the four files above. Adjust `ORG` and `PROJECT_NUMBER`.
5. Create labels: `claude-work`, `in-review`, `needs-info`.
6. Add "Icebox" and "In Review" columns to the board if not present.
7. Test via `workflow_dispatch` before enabling cron.

## 6. Multi-repo reuse with strict per-repo scoping

### 6.1 Requirements

- The system must be installable in any repo without copy-pasting workflow logic.
- Each installation must ONLY see and act on: (a) its own configured Project board and (b) issues belonging to its own repository. Board items from other repos must be filtered out even if they appear on the same board.

### 6.2 Design: central reusable workflows + thin per-repo callers

Create one central repo, e.g. `claude-automation`, containing the full logic as reusable workflows:

```
claude-automation/
  .github/workflows/
    reusable-prioritizer.yml   # full prioritizer logic, `on: workflow_call`
    reusable-worker.yml        # full worker logic, `on: workflow_call`
```

Both reusable workflows accept inputs and secrets:

```yaml
on:
  workflow_call:
    inputs:
      project_number: { required: true, type: number }
      project_owner:  { required: true, type: string }
      owner_type:     { required: false, type: string, default: "user" }
      wip_limit:      { required: false, type: number, default: 1 }
      model:          { required: false, type: string, default: "claude-fable-5" }
      fallback_model: { required: false, type: string, default: "claude-sonnet-5" }
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: { required: true }
      PROJECT_TOKEN:           { required: true }
```

Each target repo then carries only three small files plus one variable:

```
target-repo/
  .github/workflows/claude-automation.yml   # ~15 line caller
  .claude/prioritization.md                 # per-repo policy
  CLAUDE.md                                 # per-repo conventions
```

Caller example:

```yaml
name: Claude Automation
on:
  schedule:
    - cron: "0 7,13 * * 1-5"
  workflow_dispatch: {}
  issues:
    types: [labeled]

jobs:
  prioritize:
    if: github.event_name != 'issues'
    uses: your-user/claude-automation/.github/workflows/reusable-prioritizer.yml@main
    with:
      project_number: ${{ vars.PROJECT_NUMBER }}
      project_owner: your-user
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      PROJECT_TOKEN: ${{ secrets.PROJECT_TOKEN }}

  work:
    if: github.event_name == 'issues' && github.event.label.name == 'claude-work'
    uses: your-user/claude-automation/.github/workflows/reusable-worker.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Store `CLAUDE_CODE_OAUTH_TOKEN` and `PROJECT_TOKEN` as organization or user level secrets once, so target repos don't duplicate them. `PROJECT_NUMBER` is a repository variable, set per repo.

### 6.3 Repo scoping filter (mandatory)

Projects v2 boards can contain issues from multiple repositories. The reusable prioritizer must filter fetched board items to the calling repository before Claude ever sees them. Include the repository in the GraphQL selection:

```graphql
content {
  ... on Issue {
    number title body url updatedAt
    repository { nameWithOwner }
    ...
  }
}
```

Then filter in the workflow:

```bash
jq --arg repo "$GITHUB_REPOSITORY" \
  '[.data.organization.projectV2.items.nodes[]
    | select(.content.repository.nameWithOwner == $repo)]' \
  raw_board.json > board.json
```

This guarantees an instance running in repo A can never select or comment on tickets from repo B, regardless of board configuration. All dispatch commands (`gh issue edit`, `gh issue comment`) run without `--repo` overrides so they default to the current repository as a second layer of scoping.

Note. If the board belongs to a user rather than an organization, the GraphQL root is `user(login:)` instead of `organization(login:)`. The reusable workflow should accept an `owner_type` input (`user` | `organization`) and select the query accordingly.

### 6.4 Adding the system to a new repo (checklist, ~5 minutes)

1. Run `/install-github-app` in the repo (once per repo).
2. Create or pick the Project board, note its number.
3. Set repository variable `PROJECT_NUMBER`.
4. Add the caller workflow, `.claude/prioritization.md`, and `CLAUDE.md`.
5. Create labels `claude-work`, `in-review`, `needs-info`.
6. Test via `workflow_dispatch`.

### 6.5 Versioning

Pin callers to a tag (`@v1`) instead of `@main` once stable, so a change in the central repo cannot silently alter behavior across all projects. Upgrade repos deliberately by bumping the tag.

## 7. Merge gate (the one intentional human touchpoint)

Phase 1 (weeks 1 to 3): Claude opens PRs, human merges. Review every PR.
Phase 2: enable GitHub auto-merge for PRs where CI passes AND the issue had label `size/S`. Everything else waits for human review.
Phase 3 (only if Phase 2 error rate is acceptable): auto-merge all green PRs, human reviews post-merge.

Recommendation: never skip Phase 1. An agent that merges its own unreviewed code into main is the failure mode that erases all time savings.

## 8. Guardrails

- WIP limit of 1 enforced in the prioritizer (skip run if a `claude-work` issue is open).
- `--max-turns` caps on both workflows to bound cost and prevent loops.
- Worker never opens a PR on failing tests, it comments instead.
- Path denylist in CLAUDE.md (workflows, infra, migrations) so the agent cannot modify its own automation or sensitive code.
- Concurrency group on the worker workflow (and on the triage workflow, per issue) to prevent parallel runs on the same issue.
- Bot-comment exclusion: prioritizer ignores issues opened by bots to avoid feedback loops.
- Triage step has no Bash/`gh` access and cannot close, reject, or judge tickets — structural, not policy-enforced. It only ever rewrites title/body and swaps one label.

## 9. Cost estimate

The action authenticates with a Pro/Max subscription OAuth token (`claude setup-token`), so runs consume subscription usage rather than per-token API billing. Prioritizer: ~2 runs/day with small context; Worker: depends on ticket size. `--max-turns` caps on both workflows bound consumption per run. If subscription limits ever become a constraint, switching to `ANTHROPIC_API_KEY` per-token billing is a one-line change in the caller secrets; comparable API-billed setups report $15 to $25/month for 10 to 15 PRs/week.

## 10. Out of scope for v1 (candidates for v2)

- Cross-repo dependency-aware scheduling
- Learning from merge/revert outcomes to adjust prioritization weights
- Slack notifications on dispatch and PR-open events
- Estimation: having the prioritizer write size labels onto unestimated tickets
- Replacing the cron orchestrator with a long-running Claude Agent SDK service (only needed if you outgrow GitHub Actions limits)

## 11. Verification checklist

- [ ] `workflow_dispatch` run of prioritizer produces valid decision JSON
- [ ] Label application triggers worker exactly once
- [ ] Worker PR links and closes the issue, board column updates
- [ ] Blocked ticket receives a specific clarifying question, not a generic one
- [ ] WIP limit prevents a second dispatch while one is in flight
- [ ] Worker cannot edit `.github/workflows/`
- [ ] Scoping test: add an issue from another repo to the board, confirm the prioritizer never selects, comments on, or labels it
- [ ] Tester-filed feedback (via the issue form) is reformatted into `.claude/ticket-format.md` shape, keeps an intact `## Original report` section, flips to `feedback-triaged`, and is then picked up by the prioritizer like any other ticket
- [ ] Prioritizer never comments on or selects a ticket still labeled `feedback-raw`
