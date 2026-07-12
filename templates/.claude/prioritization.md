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
