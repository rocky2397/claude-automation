# Ticket format

Every ticket the prioritizer and worker act on should look like this
(this is the exact shape of `NormChat#1`, the first ticket this system
processed):

```markdown
## Problem
<what's wrong, in concrete terms>

## Expected
<what should happen instead>

## Scope
- <file or area to touch>
- <explicit "nothing else" / "do not touch X" boundary>

## Acceptance criteria
- [ ] <a checkable statement>
- [ ] <another one>

## Original report
> **Title:** <tester's original issue title, verbatim>
>
> <tester's original issue body, verbatim>
```

Tickets in this shape are what the worker (`reusable-worker.yml`) needs to
implement something without guessing: a bounded scope and checkable
acceptance criteria. `## Original report` is provenance only — it's
populated once by `reusable-triage.yml` and never edited afterward by the
prioritizer or worker.

## Where this comes from

Testers don't write tickets in this format — they file free-text feedback
through the "Feedback (for testers)" issue form, which is intentionally
just two open text boxes. The `reusable-triage.yml` workflow (triggered by
the `feedback-raw` label that form applies) sends that raw text to a cheap
model along with this file, and gets back a ticket in the format above.

The triage step only reformats. It never judges whether feedback is worth
acting on, and it never closes or rejects a ticket — that decision belongs
to the prioritizer (`.claude/prioritization.md`), which sees every
triaged ticket like any other. If the raw feedback was too vague to infer
a real scope, the triaged ticket says so explicitly ("Needs clarification:
...") instead of the triage step guessing or discarding it.
