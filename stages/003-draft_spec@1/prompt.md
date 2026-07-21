Goal: Produce an approved specification and implementation plan (no code) for the feature described at the Describe Feature gate

## Completed stages
- **describe**: succeeded

## Context
- human.gate.describe.answer: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.
- human.gate.describe.question: Describe the feature to spec and plan. Multi-line text welcome; type 'goal' to use the run goal instead.
- human.gate.label: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.
- human.gate.selected: freeform
- human.gate.text: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.


# Draft Specification

You are the spec author. The feature to specify is the text the human entered
at the Describe Feature gate (`human.gate.describe.answer` in prior context).
If that answer is empty or just "goal", use the run goal instead: Produce an approved specification and implementation plan (no code) for the feature described at the Describe Feature gate

Produce specification artifacts only — no implementation code, no tests, no
scaffolding. One feature per spec.

## Steps

1. Explore the repository enough to understand where this change fits. Focused
   reading, not research.
2. If a spec already exists under `docs/specs/` for this feature, use it as the
   base and say so.
3. Draft the three artifacts:
   - **Intent Description** — what the change achieves and why. Plain language,
     1–3 paragraphs.
   - **Architecture Specification** — components, interfaces, dependencies,
     constraints. Constrain to what the intent requires; no over-engineering.
   - **Acceptance Criteria** — observable outcomes with pass/fail conditions.
     No Gherkin — scenarios are authored per slice in the plan phase.
4. Self-critique the draft. Categorize every finding as **gap**, **ambiguity**,
   **conflict**, or **scope violation**, each with a specific reference to the
   artifact text.
5. Run the Ambiguity Resolution Protocol on every gap and ambiguity:
   - `inferable` — a reasonable developer working from the codebase and domain
     would reliably make the same choice. Document the inference and rationale.
   - `requires-stakeholder-input` — product/business intent not evident from
     context; two reasonable developers would choose differently. Naturalness
     or simplicity does NOT make a decision inferable. When in doubt, classify
     as requires-stakeholder-input.
6. Scope check: if the request bundles genuinely unrelated features (not just
   multiple slices of one feature), flag it and propose a split as one of the
   open questions. Do not split on your own.

## Output

Write the draft to `docs/specs/<slug>.md` (slugified feature name; create
`docs/specs/` if missing) with sections: Intent Description, Architecture
Specification, Acceptance Criteria, and an `## Ambiguity Log` table
(| Decision | Classification | Resolved By | Rationale / Answer |) containing
every finding from step 5.

End your final message with exactly one of:

- A numbered list titled **Open questions** containing every
  `requires-stakeholder-input` item, phrased so a stakeholder can answer each
  in one sentence.
- The line **No open questions — all findings were inferable.**

The next step shows this message to a human who will answer the questions.