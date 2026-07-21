Goal: Execute the approved implementation plan slice by slice in small batches with tests green, review the result from four angles, and ship it as a pull request

## Completed stages
- **prepare**: succeeded
- **build_slice**: succeeded

## Context
- plan_path: docs/plans/2026-07-21-resizable-interview-dock-plan.md
- plan_ready: true


> Fabro context: the plan path is in prior context (`plan_path`, emitted by
> the Locate Plan stage). You are one iteration of a loop — each visit builds
> exactly ONE slice, then routes back for the next. If you arrive from the
> Blocked gate, the human's guidance is in prior context
> (`human.gate.*.answer`): apply it before anything else.

# Build One Slice

You are the implementer. Execute the next unfinished slice of the plan —
exactly one slice per visit. Follow the plan exactly; if the plan is wrong or
contradictory, route to Blocked rather than deviating silently.

## Pick the slice

Read the plan's `## Build Progress`. Pick the first unchecked slice whose
`Depends-on` slices are all checked, respecting the `## Parallelization` wave
order. If every slice is already checked, skip to **Finishing** below.

## Small-batch cadence (non-negotiable)

Work the slice's steps in order. Every step is one behavior, built as
IMPLEMENT → TEST → REFACTOR:

1. **IMPLEMENT** exactly one behavior — nothing beyond what the step
   requires, no cleanup.
2. **TEST** — write the test for that behavior (from the slice's Gherkin
   scenarios), immediately after the code. Run the full test suite. Hard
   gate: all green before moving on. On failure, state a one-line cause
   hypothesis before correcting — never re-run a failed command unmodified.
   If two consecutive fix attempts leave the same tests failing with the
   same errors, stop patching: commit a `wip: dead-end checkpoint` and route
   to Blocked with the diagnosis.
3. **REFACTOR** on every green — structure, naming, duplication; never
   behavior, never tests (a test change means going back to TEST). Re-run
   the suite; still green. A one-line "nothing worth changing" satisfies the
   phase; skipping the check does not.

Prohibited shapes: all the code then all the tests, all the tests then all
the code, refactors deferred to the end of the build.

Each step must leave the codebase committable; commit as you go with
conventional messages.

## After the slice's steps

1. Run the FULL suite once more — the whole suite, not just this slice's
   tests, and it must be green. A pre-existing failure is still a failure:
   fix it or route to Blocked, never wave it past as someone else's.
2. Check the slice and its steps off in `## Build Progress` and commit.

## Routing

End your final message with exactly one JSON object on its own line:

- Unchecked slices remain → a one-line slice status report, then
  `{"preferred_next_label": "Next slice"}`
- **Finishing** — every slice checked, full suite green → change
  `**Status**: in-progress` to `**Status**: implemented`, commit, paste the
  final suite output as evidence, then
  `{"preferred_next_label": "Build complete", "context_updates": {"build_done": "true"}}`
- Blocked — dead-end failure, contradictory/incomplete plan, or a decision
  above your authority (spec or architecture change) → describe the blocker,
  your root-cause diagnosis, and the options a human should choose between,
  then
  `{"preferred_next_label": "Blocked"}`
