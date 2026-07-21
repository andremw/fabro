Goal: Execute the approved implementation plan slice by slice in small batches with tests green, review the result from four angles, and ship it as a pull request

## Completed stages
- **prepare**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **build_slice**: succeeded
- **reviews**: succeeded
- **merge_reviews**: succeeded

## Context
- build_done: true
- parallel.branch_count: 4
- parallel.fan_in.best_head_sha: 9b183025088f31006aa5b3e72d0323437b8c075c
- parallel.fan_in.best_id: review_quality
- parallel.fan_in.best_outcome: succeeded
- parallel.results: [{"id":"review_spec","status":"succeeded","head_sha":"c3534d65a2b2ea27ccbcb058ab753af05f848c08"},{"id":"review_quality","status":"succeeded","head_sha":"9b183025088f31006aa5b3e72d0323437b8c075c"},{"id":"review_security","status":"succeeded","head_sha":"a019327bfccd88ffeeb04958ebe62b3ed788378e"},{"id":"review_tests","status":"succeeded","head_sha":"d10e07e8e4c1368bc9c22377be594b97161093c2"}]
- plan_path: docs/plans/2026-07-21-agent-session-image-input-plan.md
- plan_ready: true


# Address Review Findings

The plan path is in prior context (`plan_path`).

You are entered from one of two places:

1. **After the reviewer fan-out** — the merged reviewer verdicts are in prior
   context (also available as `parallel_results.json`). Each reviewer
   returned JSON with a `verdict` and issues carrying `severity`.
2. **From the ship gate** — the human typed a change request
   (`human.gate.*.answer` in context). Apply it. Reviewer verdicts from the
   earlier pass still apply.

## Steps

1. Collect every `blocker` issue across reviewers (and any human change
   request). Fix each one in the code. Warnings are judgment calls: apply
   the cheap ones, record the rest.
2. Fixes follow the same discipline as the build: behavior changes get a
   test, refactors keep tests frozen, and the FULL suite must be green
   after every fix — paste the final suite output as evidence.
3. Append (or replace) a `## Ship Review Summary` section in the plan file:
   reviewer verdicts, blockers fixed, warnings applied or deliberately kept.
4. Commit everything with conventional messages.

## Output

Your final message: a short summary of what each reviewer found, what you
changed, what you deliberately did not change and why, and the final suite
evidence. A human reads this message at the ship gate to decide whether the
PR opens — make it decision-ready, not a transcript.
