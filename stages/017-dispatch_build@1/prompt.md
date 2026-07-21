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
- **address_reviews**: succeeded
  - Files: plans/2026-07-21-agent-session-image-input.md
- **plan_gate**: succeeded

## Context
- human.gate.clarify.answer: Ok go ahead
- human.gate.clarify.question: Answer the open questions above. Type 'proceed' to accept all documented inferences.
- human.gate.describe.answer: Spec and plan the fabro-agent Session API change needed to pass images into an agent's initial user message. Today Session::process_input/process_input_with_runtime take input: &str and internally build Message::User { content: String }, so CodergenRunRequest.initial_images (already added) has no path through the backend into the message. Design the least-invasive API that lets fabro-workflow's handler/agent.rs prepend ContentPart::Image entries ahead of the text content, covering: the Session method signature and how it threads to the llm/api.rs backend, whether AgentMessage::User needs Vec<ContentPart> instead of String, migration of existing CodergenRunRequest call sites and tests, and how image paths from fabro.human_answer_images.* context keys reach the LLM providers. This unblocks Slice 5 of docs/plans/2026-07-21-freeform-image-attachments-plan.md, which stalled because it assumed the workflow handler could construct the message itself.
- human.gate.describe.question: Describe the feature to spec and plan. Multi-line text welcome; type 'goal' to use the run goal instead.
- human.gate.label: [A] Approve plan
- human.gate.plan_gate.answer: A
- human.gate.plan_gate.label: [A] Approve plan
- human.gate.plan_gate.question: Approve this plan? Approval dispatches the build-ship child run. Anything else you type is treated as a change request.
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


# Dispatch Build

The human just approved the implementation plan. Your only job is to spawn the
build-ship workflow as a child run and report what you did. Do not implement
anything and do not wait for the child to finish.

1. Identify the approved plan file: the most recently updated `plans/*.md`
   (its path was also stated in the prior stage's output).
2. Call the `fabro_run_create` tool:

   ```json
   {
     "runs": [
       {
         "workflow": "build-ship",
         "goal": "Implement <plan path> slice by slice and ship it as a draft PR.",
         "labels": { "source": "specs-plan" },
         "start": true
       }
     ]
   }
   ```

3. Report the child run id and its status. If the child was created in a
   `pending` approval state, say so — the operator approves it from the Fabro
   UI.

If the tool is unavailable or creation fails, say exactly that and stop — the
spec/plan PR still opens when this run completes, and the operator can run
build-ship manually. Never claim the build was dispatched when it wasn't.
