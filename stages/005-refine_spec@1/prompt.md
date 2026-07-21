Goal: Produce an approved specification and implementation plan (no code) for the feature described at the Describe Feature gate

## Completed stages
- **describe**: succeeded
- **draft_spec**: succeeded
  - Files: docs/superpowers/specs/2026-07-21-agent-session-image-input.md
- **clarify**: succeeded

## Context
- human.gate.clarify.answer: Ok go ahead
- human.gate.clarify.question: Answer the open questions above. Type 'proceed' to accept all documented inferences.
- human.gate.describe.answer: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.
- human.gate.describe.question: Describe the feature to spec and plan. Multi-line text welcome; type 'goal' to use the run goal instead.
- human.gate.label: Ok go ahead
- human.gate.selected: freeform
- human.gate.text: Ok go ahead


# Refine Specification

You are refining the spec drafted earlier in this run (see prior context for
its path under `docs/specs/`).

The human has just responded. Their text is in the prior context
(`human.gate.*.answer` keys). It is either:

- **Answers to the open questions** — record each answer in the
  `## Ambiguity Log` (Classification `requires-stakeholder-input`,
  Resolved By `human`, the answer verbatim in the Rationale/Answer column) and
  update the artifacts accordingly. Preserve the human's language; improve
  precision only.
- **"proceed"** — no open questions remained or the human accepted the
  documented inferences. Continue.
- **A change request** (when re-entered from the approval gate) — apply it,
  updating the Ambiguity Log if it resolves or adds a decision.

Then validate the Cross-Artifact Consistency Gate and append it to the spec
file as a checklist:

- [ ] Intent is unambiguous — two developers would interpret it the same way.
- [ ] Every behavior/goal in the intent maps to at least one acceptance criterion.
- [ ] Architecture constrains implementation without over-engineering.
- [ ] Same concepts named consistently across all three artifacts.
- [ ] No artifact contradicts another.
- [ ] Every gap/ambiguity finding is logged — inferable with rationale, or resolved by the human.

If any item fails, fix the artifacts until it passes; if you cannot, say
exactly which item fails and why — do not claim the gate passed.

## Output

Update `docs/specs/<slug>.md` in place. Your final message: the complete spec
file content, then its path. A human reviews this message to approve the spec.
