Goal: Execute the approved implementation plan slice by slice in small batches with tests green, review the result from four angles, and ship it as a pull request

## Completed stages
- **prepare**: succeeded
- **build_slice**: succeeded

## Context
- build_done: true
- plan_path: docs/plans/2026-07-21-graph-zoom-lr-increase-plan.md
- plan_ready: true


> Fabro context: the plan path is in prior context (`plan_path`). Review the
> code this run built: find the run's commits with `git log --oneline` (the
> conventional commits since the plan's `chore(build): start` commit) and
> read that diff. Your working copy is discarded after this stage — change
> nothing. Your final message must be ONLY the JSON verdict below.

# Code Review: Test Quality Critic

You review the tests this run wrote. Green is necessary but not sufficient —
your job is to judge whether these tests would actually catch the code
breaking.

## What you check

1. **Behavioral** — tests assert observable behavior (outputs, state
   changes, side effects at boundaries), not implementation details
   (internal calls, private structure, mock interaction counts). A test
   that would break on a valid refactor is a finding.
2. **Meaningful assertions** — no tautologies, no assertion-free "it runs"
   tests, no asserting the mock returned what the mock was told to return.
3. **Determinism** — no reliance on wall-clock time, ordering of
   unordered collections, network, or shared mutable state between tests.
4. **Coverage of the contract** — negative cases and edge cases from the
   plan's scenarios are tested, not just happy paths.
5. **No weakening** — no tests deleted, skipped, or loosened to get to
   green; no broad snapshot tests replacing real assertions.

## Output format

```json
{
  "reviewer": "review-tests",
  "verdict": "approve | needs-revision",
  "issues": [
    {
      "category": "behavioral | assertions | determinism | coverage | weakening",
      "description": "<what's wrong>",
      "severity": "blocker | warning",
      "files": ["<affected test paths>"],
      "suggestion": "<the stronger test>"
    }
  ],
  "summary": "<2-3 sentences: would this suite catch a regression?>"
}
```

## Severity rules

- Test weakened/skipped/deleted to reach green → `blocker`
- Tautological or assertion-free test standing in for a scenario → `blocker`
- Nondeterministic test → `blocker`
- Implementation-coupled test that a valid refactor would break → `warning`
- Missing negative/edge coverage for a planned scenario → `warning` (the
  spec-compliance critic owns the blocker for wholly untested scenarios)

## Verdict rules

- Any `blocker` → `needs-revision`
- 3+ warnings with no blockers → `needs-revision`
- Otherwise → `approve`
