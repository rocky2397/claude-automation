Raw structured event log, one file per UTC day, `YYYY-MM-DD.jsonl`.

One JSON object per instrumented job, emitted by the three reusable workflows
and collected nightly. Fields:

  ts, repo, stage          when, where, and which stage (prioritize|triage|work)
  outcome                  what actually happened, independent of exit code:
                           picked | no_pick | wip_limit_reached | retriaged |
                           pr_opened | pr_opened_but_job_failed | no_pr | error
  detail                   one-line human-readable expansion of `outcome`
  issue                    the ticket involved, or null
  job_status, claude_step  GitHub's own verdicts, for comparison with `outcome`
  claude.ran               false when Claude never executed at all
  claude.is_error          true on a failed agent run
  claude.num_turns         1 with is_error=true means it died before working
  claude.permission_denials  non-zero means the --allowedTools list is too narrow
  claude.result_text       the agent's final message, when present
  run_url                  link to the run

Grep this before reading a report:

  jq -r 'select(.outcome|test("error|no_pr|wip_limit")) | .ts+" "+.stage+" "+.detail' logs/*.jsonl
