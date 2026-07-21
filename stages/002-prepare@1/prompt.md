Goal: Execute the approved implementation plan slice by slice in small batches with tests green, review the result from four angles, and ship it as a pull request


# Locate the Approved Plan

You are the build orchestrator's first stage. Locate the plan, verify it is
approved, and set up progress tracking. Do not implement anything.

Requested plan: `docs/plans/2026-07-21-graph-zoom-lr-increase-plan.md`

## Steps

1. If the requested plan is a path, read that file. If it is `auto`, list
   `plans/*.md` and pick the most recently modified one whose header says
   `**Status**: approved` (a plan the specs-plan workflow produced and a
   human approved at its plan gate; `reviewed` does not count).
2. If the plan has no `## Build Progress` section, append one: a checkbox
   per slice, with a nested checkbox per step, mirroring `## Slices` —
   this section on disk is the build's durable state across stages.
3. Change `**Status**: approved` to `**Status**: in-progress` and commit the
   plan file (`chore(build): start <slug>`).

## Output

Found and prepared a plan — final message states the plan path, the slice
count, and the wave order, ending with exactly this JSON on its own line:

```json
{"preferred_next_label": "Plan ready", "context_updates": {"plan_ready": "true", "plan_path": "plans/<slug>.md"}}
```

No approved plan found — say so, list what `plans/` contains and each file's
status, tell the human to run the specs-plan workflow (or approve a draft
plan) first, and end with:

```json
{"preferred_next_label": "No approved plan"}
```