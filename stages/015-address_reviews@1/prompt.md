Goal: Produce an approved specification and implementation plan (no code) for the feature described at the Describe Feature gate

## Completed stages
- **describe**: succeeded
- **draft_spec**: succeeded
  - Files: docs/superpowers/specs/2026-07-21-agent-session-image-input.md
- **clarify**: succeeded
- **refine_spec**: succeeded
  - Files: docs/superpowers/specs/2026-07-21-agent-session-image-input.md
- **spec_gate**: succeeded
- **plan_feature**: succeeded
  - Files: plans/2026-07-21-agent-session-image-input.md
- **reviews**: succeeded
- **merge_reviews**: succeeded

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
- parallel.branch_count: 5
- parallel.fan_in.best_head_sha: 9ec28fbafbbf3e62068130130e9a4bb3f7ab4b3e
- parallel.fan_in.best_id: review_acceptance
- parallel.fan_in.best_outcome: succeeded
- parallel.results: [{"id":"review_acceptance","status":"succeeded","head_sha":"9ec28fbafbbf3e62068130130e9a4bb3f7ab4b3e"},{"id":"review_design","status":"succeeded","head_sha":"c585c79c5c74c2bd38bc235b38c00b802fbd0371"},{"id":"review_ux","status":"succeeded","head_sha":"2b54e70c0996526208b220e154c43be2ab663cee"},{"id":"review_strategic","status":"succeeded","head_sha":"5b743721f5312925f51531d9d26a9f3ddfdc4bb2"},{"id":"review_parallel","status":"succeeded","head_sha":"63c9435d6c797ac036a8998b38abc6d8ae8a78d4"}]


# Address Plan Reviews

The plan file path is in prior context (`plans/<slug>.md`).

You are entered from one of two places:

1. **After the reviewer fan-out** — the merged reviewer verdicts are in prior
   context (also available as `parallel_results.json`). Each reviewer returned
   JSON with a `verdict` and issues carrying `severity`.
2. **From the approval gate** — the human typed a change request
   (`human.gate.*.answer` in context). Apply it. Reviewer verdicts from the
   earlier pass still apply.

## Steps

1. Collect every `blocker` issue across reviewers (and any human change
   request). Revise the plan file to resolve each one. Warnings are judgment
   calls: apply the cheap ones, record the rest.
2. Keep the plan's invariants intact after revision: every acceptance
   criterion still covered by a scenario, waves still consistent with
   Depends-on, no same-wave file collisions, no low-value work in a slice.
3. Append (or replace) a `## Plan Review Summary` section in the plan file:
   reviewer verdicts, blockers fixed, warnings and observations kept.
4. Set `**Status**: reviewed`.

## Output

Your final message: the plan path, a short summary of what each reviewer
found, what you changed, and anything you deliberately did not change and why.
A human reads this message to approve the plan — make it decision-ready, not a
transcript.
