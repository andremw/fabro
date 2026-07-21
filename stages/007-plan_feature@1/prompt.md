Goal: Produce an approved specification and implementation plan (no code) for the feature described at the Describe Feature gate

## Completed stages
- **describe**: succeeded
- **draft_spec**: succeeded
  - Files: docs/superpowers/specs/2026-07-21-agent-session-image-input.md
- **clarify**: succeeded
- **refine_spec**: succeeded
  - Files: docs/superpowers/specs/2026-07-21-agent-session-image-input.md
- **spec_gate**: succeeded

## Context
- human.gate.clarify.answer: Ok go ahead
- human.gate.clarify.question: Answer the open questions above. Type 'proceed' to accept all documented inferences.
- human.gate.describe.answer: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.
- human.gate.describe.question: Describe the feature to spec and plan. Multi-line text welcome; type 'goal' to use the run goal instead.
- human.gate.label: [A] Approve spec
- human.gate.selected: A
- human.gate.spec_gate.answer: A
- human.gate.spec_gate.label: [A] Approve spec
- human.gate.spec_gate.question: Approve this spec and continue to planning? Anything else you type is treated as a change request.
- human.gate.text: Ok go ahead


# Write Implementation Plan

You are the planner. Create a structured implementation plan — do not
implement anything. No code, no scaffolding, no file edits beyond the plan
file.

The approved spec for this feature is under `docs/specs/` (path in prior
context). Read it: it is the primary source for goals, constraints, and
acceptance criteria. Read whatever code you need to plan confidently — focused
exploration, not research.

## Decompose into vertical slices

A slice is a vertically deliverable increment — independently testable and,
ideally, independently shippable. Sequence slices so trunk stays releasable at
every step (feature toggle or abstraction for incomplete behavior;
expand-before-contract for data changes).

**A slice is also the review unit: it ships as its own PR.** Size each slice
to roughly 300 changed lines or less where practical. When a slice can't stay
near that budget (a generated file, a mechanical rename), say so in the slice
and keep the hand-written portion small.

For each slice, author the **Gherkin scenarios** that define its observable
behavior — this is where the behavioral contract is written. Cover:

- **Happy path** — the primary success behavior.
- **Negative cases** — invalid, unauthorized, missing, malformed input.
- **Edge cases** — empty collections, boundary values, concurrency, idempotency.
- **Error scenarios** — observable error behavior, not just "should fail".

Scenarios must be implementation-independent (no databases, selectors, or
internal data structures in step text) and deterministic. Every acceptance
criterion in the spec must be covered by at least one scenario.

For each slice, list **TDD steps**: one behavior per step, each an
IMPLEMENT → TEST → REFACTOR cycle, each leaving the codebase committable, each
traceable to one or more of the slice's scenarios.

Findings the spec's Ambiguity Log classified as low-value (no branching logic,
no observable outcome, already covered by a higher-layer test) go in a
`## Skipped (low value)` section with one-line rationale — never into a slice.

## Plan file structure

Write to `plans/<slug>.md` (create `plans/` if missing):

```markdown
# Plan: <Feature Name>

**Status**: draft
**Spec**: docs/specs/<slug>.md

## Goal
## Acceptance Criteria     (from the spec — PR-checklist material)
## Slices
### Slice N: <name>
**Depends-on**: <slice refs or none>
**Files**: <files this slice touches>
#### Scenarios              (the slice's Gherkin)
#### Steps                  (TDD steps)
## Parallelization          (wave table: slices grouped by dependency depth;
                             flag any same-wave slices whose Files overlap)
## Skipped (low value)
## Risks & Open Questions
```

Derive the waves from the Depends-on graph yourself and double-check that no
two same-wave slices touch the same file — a collision there corrupts
concurrent builds.

## Output

Your final message: the plan path, the slice list with one line each, the wave
table, and any risks. Reviewer agents will read the plan file next.
