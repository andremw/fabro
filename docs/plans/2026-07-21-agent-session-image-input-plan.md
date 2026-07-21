# Plan: Agent Session Image Input API

**Status**: implemented
**Spec**: docs/superpowers/specs/2026-07-21-agent-session-image-input.md

## Goal

Enable fabro-workflow's agent handler to pass images into the agent's initial user message when invoking a Session. The Session API currently accepts only text input (`&str`) and internally constructs single-part `Message::User { content: String }`. This plan adds multi-part content support (`Vec<ContentPart>`) to thread image attachments from workflow context through the agent session to LLM providers.

## Acceptance Criteria

1. **Session accepts multi-part initial message**: `Session::process_message(vec![ContentPart::Image(...), ContentPart::Text(...)], runtime)` constructs a `Message::User` with the full content vector, appends to history, and proceeds with the agent loop normally.

2. **Text-only messages still work**: `Session::process_text_input("hello")` constructs a single-text-part User message. Existing test cases using text-only input compile and pass without modification.

3. **Images reach the LLM provider**: When a User message contains `ContentPart::Image(ImageData { url: Some("/path/to/image.png"), ... })`, the history-to-LLM conversion preserves it, fabro-llm attachment resolver loads and inlines it, and the provider-specific encoder includes it in the API request.

4. **Workflow handler can inject images**: When `extract_human_images(context)` returns `Some(paths)`, the agent handler prepends `ContentPart::Image` entries to the initial content. The constructed `CodergenRunRequest` contains `initial_content: Vec<ContentPart>` and the backend's `run()` method passes it to `session.process_message`.

5. **Migration is mechanical**: All existing call sites constructing `Message::User { content: String, ... }` are identified via grep. After updating to `content: vec![ContentPart::Text(...)]`, the codebase compiles and all existing agent tests pass.

6. **Agent session continues resumption support**: Serializing and deserializing session history preserves multi-part User messages. Resuming from a checkpoint with multi-part User messages works correctly. Round-tripping through `SessionMessage` preserves image parts.

## Slices

### Slice 1: AgentMessage::User content type change

**Depends-on**: none

**Files**:
- lib/crates/fabro-agent/src/types.rs
- lib/crates/fabro-agent/src/history.rs
- lib/crates/fabro-agent/src/session.rs
- lib/crates/fabro-agent/src/compaction.rs
- lib/crates/fabro-agent/src/loop_detection.rs
- lib/crates/fabro-store/src/run_sessions.rs

#### Scenarios

```gherkin
Scenario: Message::User content is Vec<ContentPart>
  Given the Message enum in types.rs
  When I change the User variant from content: String to content: Vec<ContentPart>
  Then the type compiles
  And ContentPart is imported from fabro_llm::types

Scenario: Single-text User message can be constructed
  Given the new Message::User structure
  When I construct Message::User { content: vec![ContentPart::Text("hello".to_string())], timestamp: SystemTime::now() }
  Then the message is valid
  And it can be pushed to History

Scenario: Multi-part User message can be constructed
  Given ImageData and ContentPart types from fabro_llm
  When I construct Message::User { content: vec![ContentPart::Image(ImageData { url: Some("/tmp/img.png".into()), data: None, media_type: None, detail: None }), ContentPart::Text("analyze".into())], timestamp: SystemTime::now() }
  Then the message is valid
  And it contains both image and text parts

Scenario: History conversion preserves multi-part User content
  Given History::convert_to_messages at history.rs:96
  When the Message::User arm converts to LlmMessage
  Then it constructs LlmMessage { role: Role::User, content: content.clone(), name: None, tool_call_id: None }
  And it does not use LlmMessage::user(str) helper

Scenario: SessionMessage round-trip preserves ContentPart vector
  Given Message::to_session_message and from_session_message in types.rs
  When I convert a Message::User with multi-part content to SessionMessage and back
  Then the content vector is preserved
  And serde handles Vec<ContentPart> automatically
```

#### Steps

1. **Change Message::User content field type** — in `lib/crates/fabro-agent/src/types.rs` around line 40, change `content: String` to `content: Vec<ContentPart>`; add `use fabro_llm::types::ContentPart;` at the top.
   - **Scenario**: Message::User content is Vec<ContentPart>

2. **Test Message::User with single text part compiles** — unit test that constructs `Message::User { content: vec![ContentPart::Text("test".into())], timestamp: SystemTime::now() }`, asserts it can be pushed to History, compiles without errors.
   - **Scenario**: Single-text User message can be constructed

3. **Test Message::User with image and text parts** — unit test that constructs multi-part User message with `ContentPart::Image` and `ContentPart::Text`, asserts content vector has 2 elements.
   - **Scenario**: Multi-part User message can be constructed

4. **Update History::convert_to_messages User arm** — in `lib/crates/fabro-agent/src/history.rs` around line 100, replace `Message::User { content, .. } => LlmMessage::user(content)` with `Message::User { content, .. } => LlmMessage { role: Role::User, content: content.clone(), name: None, tool_call_id: None }`.
   - **Scenario**: History conversion preserves multi-part User content

5. **Test history conversion with multi-part User** — unit test that creates History with multi-part User message, calls `convert_to_messages()`, asserts resulting LlmMessage has content vector matching original.
   - **Scenario**: History conversion preserves multi-part User content

6. **Update all internal Message::User construction sites** — grep for `Message::User \{` in fabro-agent crate; update each site to wrap string content in `vec![ContentPart::Text(...)]`; affected files include `session.rs` (run_single_input), `compaction.rs`, `loop_detection.rs`.
   - **Scenario**: Message::User content is Vec<ContentPart>

7. **Test internal constructions compile** — run `cargo build -p fabro-agent`, verify no compilation errors.
   - **Scenario**: Message::User content is Vec<ContentPart>

8. **Test SessionMessage round-trip** — unit test that constructs Message::User with multi-part content, calls `to_session_message()`, then `from_session_message()`, asserts equality; verify ContentPart serialization via serde is automatic.
   - **Scenario**: SessionMessage round-trip preserves ContentPart vector

9. **Refactor** — extract test helpers for constructing common Message::User patterns (text-only, image+text) to reduce boilerplate in subsequent tests.

---

### Slice 2: Session::process_message API

**Depends-on**: Slice 1 (Message::User type change)

**Files**:
- lib/crates/fabro-agent/src/session.rs

#### Scenarios

```gherkin
Scenario: process_message accepts Vec<ContentPart> and AgentToolRuntime
  Given the Session struct in session.rs
  When I add pub async fn process_message(&mut self, content: Vec<ContentPart>, agent_tool_runtime: AgentToolRuntime) -> Result<(), Error>
  Then the method signature compiles
  And content is the first parameter, agent_tool_runtime is the second

Scenario: process_message constructs Message::User and runs agent loop
  Given a Session instance
  When I call session.process_message(vec![ContentPart::Text("hello".into())], AgentToolRuntime::default()).await
  Then a Message::User with content vec![Text("hello")] is pushed to history
  And the agent loop proceeds (calls LLM, processes response, handles tool calls)
  And the session transitions to Idle state on success

Scenario: process_message handles multi-part content
  Given a Session instance
  When I call process_message with vec![ContentPart::Image(...), ContentPart::Text(...)]
  Then Message::User with both parts is added to history
  And convert_to_messages preserves both parts when building LLM request

Scenario: process_text_input convenience wrapper works
  Given the new process_text_input method
  When I call session.process_text_input("hello").await
  Then it delegates to process_message(vec![ContentPart::Text("hello".into())], AgentToolRuntime::default())
  And the behavior is identical to the old process_input("hello")

Scenario: Existing process_input and process_input_with_runtime are removed
  Given the Session API
  When I remove process_input and process_input_with_runtime
  Then the API surface is reduced to process_message and process_text_input
  And callers must update to the new API
```

#### Steps

1. **Add Session::process_message method** — in `lib/crates/fabro-agent/src/session.rs` around line 1210, add `pub async fn process_message(&mut self, content: Vec<ContentPart>, agent_tool_runtime: AgentToolRuntime) -> Result<(), Error>` that mirrors the existing `process_input_with_runtime` logic but accepts `content` directly instead of `input: &str`.
   - **Scenario**: process_message accepts Vec<ContentPart> and AgentToolRuntime

2. **Implement process_message body** — move the existing `process_input_with_runtime` implementation into `process_message`; when constructing the initial `Message::User` in the delegated `run_single_input` call, use the passed `content` directly instead of wrapping a string in a text part; preserve timeout setup, followup queue draining, timing/usage accumulation, state transitions.
   - **Scenario**: process_message constructs Message::User and runs agent loop

3. **Test process_message with single text part** — unit test that creates a Session, calls `process_message(vec![ContentPart::Text("test".into())], AgentToolRuntime::default()).await`, asserts history contains User message with one text part, agent loop ran (mock LLM client or capture history turns).
   - **Scenario**: process_message constructs Message::User and runs agent loop

4. **Test process_message with multi-part content** — unit test with `vec![ContentPart::Image(...), ContentPart::Text(...)]`, asserts history contains User message with both parts in order.
   - **Scenario**: process_message handles multi-part content

5. **Add Session::process_text_input convenience method** — add `pub async fn process_text_input(&mut self, input: &str) -> Result<(), Error> { self.process_message(vec![ContentPart::Text(input.to_string())], AgentToolRuntime::default()).await }` after `process_message`.
   - **Scenario**: process_text_input convenience wrapper works

6. **Test process_text_input delegates correctly** — unit test that calls `process_text_input("hello")`, asserts it produces the same history as calling `process_message(vec![ContentPart::Text("hello".into())], ...)`.
   - **Scenario**: process_text_input convenience wrapper works

7. **Remove process_input and process_input_with_runtime** — delete both methods from session.rs around line 1210-1294; verify removal does not leave orphaned helper calls (check that run_single_input is still used by process_message).
   - **Scenario**: Existing process_input and process_input_with_runtime are removed

8. **Test removal by attempting build** — run `cargo build -p fabro-agent`, expect compilation errors at call sites; these will be fixed in Slice 3.
   - **Scenario**: Existing process_input and process_input_with_runtime are removed

9. **Refactor** — none for this slice (process_message is a direct replacement with minimal changes to the loop logic).

---

### Slice 3: Update agent crate test call sites

**Depends-on**: Slice 2 (process_message API)

**Files**:
- lib/crates/fabro-agent/tests/it/parity_matrix.rs
- lib/crates/fabro-agent/tests/it/compaction.rs
- lib/crates/fabro-agent/src/subagent.rs
- lib/crates/fabro-agent/src/apply_patch.rs
- lib/crates/fabro-agent/src/cli.rs

#### Scenarios

```gherkin
Scenario: parity_matrix tests use process_text_input
  Given tests in parity_matrix.rs that call session.process_input(input)
  When I update to session.process_text_input(input)
  Then the tests compile and pass

Scenario: compaction tests use process_text_input
  Given tests in compaction.rs that call process_input
  When I update to process_text_input
  Then the tests compile and pass

Scenario: subagent uses process_message with runtime
  Given subagent.rs calls session.process_input_with_runtime(prompt, runtime)
  When I update to session.process_message(vec![ContentPart::Text(prompt.to_string())], runtime)
  Then the subagent compiles and behaves identically

Scenario: apply_patch uses process_text_input
  Given apply_patch.rs calls process_input
  When I update to process_text_input
  Then the apply_patch functionality compiles and passes tests

Scenario: CLI uses process_text_input
  Given cli.rs calls process_input
  When I update to process_text_input
  Then the CLI compiles and manual testing shows identical behavior
```

#### Steps

1. **Update parity_matrix.rs call sites** — grep for `process_input\(` in `lib/crates/fabro-agent/tests/it/parity_matrix.rs`, replace with `process_text_input(...)`.
   - **Scenario**: parity_matrix tests use process_text_input

2. **Test parity_matrix compiles and passes** — run `cargo nextest run -p fabro-agent parity_matrix`, assert all tests pass.
   - **Scenario**: parity_matrix tests use process_text_input

3. **Update compaction.rs call sites** — grep for `process_input\(` in `lib/crates/fabro-agent/tests/it/compaction.rs`, replace with `process_text_input(...)`.
   - **Scenario**: compaction tests use process_text_input

4. **Test compaction compiles and passes** — run `cargo nextest run -p fabro-agent compaction`, assert tests pass.
   - **Scenario**: compaction tests use process_text_input

5. **Update subagent.rs call site** — in `lib/crates/fabro-agent/src/subagent.rs`, find call to `process_input_with_runtime(prompt, runtime)`, replace with `process_message(vec![ContentPart::Text(prompt.to_string())], runtime)`.
   - **Scenario**: subagent uses process_message with runtime

6. **Test subagent compiles** — run `cargo build -p fabro-agent`, verify subagent module compiles.
   - **Scenario**: subagent uses process_message with runtime

7. **Update apply_patch.rs call site** — grep for `process_input\(` in `lib/crates/fabro-agent/src/apply_patch.rs`, replace with `process_text_input(...)`.
   - **Scenario**: apply_patch uses process_text_input

8. **Test apply_patch compiles** — run `cargo build -p fabro-agent`, verify apply_patch compiles.
   - **Scenario**: apply_patch uses process_text_input

9. **Update cli.rs call site** — grep for `process_input\(` in `lib/crates/fabro-agent/src/cli.rs`, replace with `process_text_input(...)`.
   - **Scenario**: CLI uses process_text_input

10. **Test CLI compiles** — run `cargo build -p fabro-agent`, verify cli module compiles; manual smoke test: `fabro agent run "print hello"` works.
    - **Scenario**: CLI uses process_text_input

11. **Run full fabro-agent test suite** — `cargo nextest run -p fabro-agent`, assert all tests pass after call-site updates.
    - **Scenarios**: All scenarios in this slice

12. **Refactor** — none (mechanical find-replace updates).

---

### Slice 4: CodergenRunRequest and backend updates

**Depends-on**: Slice 2 (process_message API)

**Files**:
- lib/crates/fabro-workflow/src/handler/agent.rs
- lib/crates/fabro-workflow/src/handler/llm/api.rs
- lib/crates/fabro-workflow/src/handler/llm/acp.rs
- lib/crates/fabro-workflow/src/handler/prompt.rs

#### Scenarios

```gherkin
Scenario: CodergenRunRequest uses initial_content: Vec<ContentPart>
  Given the CodergenRunRequest struct in handler/agent.rs
  When I change prompt: &'a str to initial_content: Vec<ContentPart>
  Then the struct compiles
  And initial_content is owned (Vec) not borrowed

Scenario: AgentApiBackend::run passes initial_content to session
  Given AgentApiBackend::run in handler/llm/api.rs
  When it extracts req.initial_content
  And calls session.process_message(req.initial_content, req.agent_tool_runtime)
  Then the backend compiles
  And images in initial_content reach the agent session

Scenario: AcpBackend::run passes initial_content to session
  Given AcpBackend::run in handler/llm/acp.rs
  When it extracts req.initial_content
  And calls session.process_message(req.initial_content, req.agent_tool_runtime)
  Then the backend compiles

Scenario: PromptHandler fallback uses initial_content as text
  Given PromptHandler (fallback for non-agent models) in handler/prompt.rs
  When it receives CodergenRunRequest with initial_content
  And it extracts text from initial_content (joins Text parts, ignores Image parts)
  Then the fallback compiles
  And text-only prompt behavior is preserved
```

#### Steps

1. **Change CodergenRunRequest.prompt to initial_content** — in `lib/crates/fabro-workflow/src/handler/agent.rs` around line 40, change field from `pub prompt: &'a str` to `pub initial_content: Vec<ContentPart>`; add `use fabro_llm::types::ContentPart;`.
   - **Scenario**: CodergenRunRequest uses initial_content: Vec<ContentPart>

2. **Test CodergenRunRequest compiles** — run `cargo build -p fabro-workflow`, expect errors at backend call sites (fixed in steps 3-6).
   - **Scenario**: CodergenRunRequest uses initial_content: Vec<ContentPart>

3. **Update AgentApiBackend::run to use initial_content** — in `lib/crates/fabro-workflow/src/handler/llm/api.rs`, find where it calls `session.process_input_with_runtime(req.prompt, req.agent_tool_runtime)`, replace with `session.process_message(req.initial_content, req.agent_tool_runtime)`.
   - **Scenario**: AgentApiBackend::run passes initial_content to session

4. **Test AgentApiBackend compiles** — run `cargo build -p fabro-workflow`, verify api.rs compiles.
   - **Scenario**: AgentApiBackend::run passes initial_content to session

5. **Update AcpBackend::run to use initial_content** — in `lib/crates/fabro-workflow/src/handler/llm/acp.rs`, find similar call site, replace with `session.process_message(req.initial_content, req.agent_tool_runtime)`.
   - **Scenario**: AcpBackend::run passes initial_content to session

6. **Test AcpBackend compiles** — run `cargo build -p fabro-workflow`, verify acp.rs compiles.
   - **Scenario**: AcpBackend::run passes initial_content to session

7. **Update PromptHandler to extract text from initial_content** — in `lib/crates/fabro-workflow/src/handler/prompt.rs`, replace usage of `req.prompt` with a helper that extracts text from `req.initial_content`: `fn extract_text(content: &[ContentPart]) -> String { content.iter().filter_map(|p| if let ContentPart::Text(t) = p { Some(t.as_str()) } else { None }).collect::<Vec<_>>().join("") }`; use this helper where prompt text is needed.
   - **Scenario**: PromptHandler fallback uses initial_content as text

8. **Test PromptHandler compiles** — run `cargo build -p fabro-workflow`, verify prompt.rs compiles.
   - **Scenario**: PromptHandler fallback uses initial_content as text

9. **Run fabro-workflow tests** — `cargo nextest run -p fabro-workflow`, assert backend tests pass (may need Slice 5 for agent handler call-site fix to avoid failures).
   - **Scenarios**: All scenarios in this slice

10. **Refactor** — extract `extract_text` helper to a shared utility in `handler/agent.rs` or `handler/mod.rs` if multiple handlers need it.

---

### Slice 5: Agent handler image propagation

**Depends-on**: Slice 4 (CodergenRunRequest update)

**Files**:
- lib/crates/fabro-workflow/src/handler/agent.rs
- lib/crates/fabro-workflow/src/context.rs (keys module)

#### Scenarios

```gherkin
Scenario: extract_human_images reads context and returns image paths
  Given workflow context with key "fabro.human_answer_images.review": ["/tmp/img.png"]
  When I call extract_human_images(context, "review")
  Then it returns Some(vec!["/tmp/img.png".to_string()])

Scenario: extract_human_images returns None when key is absent
  Given context without "fabro.human_answer_images.*" keys
  When I call extract_human_images(context, "review")
  Then it returns None

Scenario: Agent handler prepends images to initial_content
  Given agent handler execute method in handler/agent.rs
  And context contains "fabro.human_answer_images.prior_stage": ["/tmp/img.png", "/tmp/img2.jpg"]
  When constructing CodergenRunRequest
  Then initial_content is vec![Image("/tmp/img.png"), Image("/tmp/img2.jpg"), Text(prompt)]

Scenario: Agent handler constructs text-only initial_content when no images
  Given context without human_answer_images keys
  When constructing CodergenRunRequest
  Then initial_content is vec![Text(prompt)]

Scenario: Images reach LLM via fabro-llm attachment resolution
  Given an agent session with ContentPart::Image(ImageData { url: Some("/tmp/img.png"), ... })
  When fabro-llm builds the LLM request
  And attachments::resolve is called
  Then the image file is read and inlined as base64
  And the LLM request includes the inline image
```

#### Steps

1. **Add HUMAN_ANSWER_IMAGES_PREFIX constant** — in `lib/crates/fabro-workflow/src/context.rs` keys module around line 50, add `pub const HUMAN_ANSWER_IMAGES_PREFIX: &str = "fabro.human_answer_images.";`.
   - **Scenario**: extract_human_images reads context and returns image paths

2. **Add extract_human_images helper** — in `lib/crates/fabro-workflow/src/handler/agent.rs`, add `fn extract_human_images(context: &Context, stage_id: &str) -> Option<Vec<String>> { let key = format!("{}{}", keys::HUMAN_ANSWER_IMAGES_PREFIX, stage_id); context.get(&key).and_then(|v| serde_json::from_value(v.clone()).ok()) }`.
   - **Scenario**: extract_human_images reads context and returns image paths

3. **Test extract_human_images with images in context** — unit test that creates Context with "fabro.human_answer_images.review": `["path"]`, calls helper, asserts returns Some(vec!["path"]).
   - **Scenario**: extract_human_images reads context and returns image paths

4. **Test extract_human_images returns None when key absent** — unit test with empty context, asserts returns None.
   - **Scenario**: extract_human_images returns None when key is absent

5. **Identify prior stage from context** — in agent handler execute method around line 260, add logic to read prior stage ID from context (e.g., `context.get(keys::LAST_STAGE)`); assume the prior stage is the source of images if the key exists.
   - **Scenario**: Agent handler prepends images to initial_content

6. **Construct initial_content with images** — in agent handler execute method, after extracting prompt, call `extract_human_images(context, prior_stage_id)`; if Some(paths), prepend `ContentPart::Image(ImageData { url: Some(path.clone()), data: None, media_type: None, detail: None })` for each path; append `ContentPart::Text(prompt.clone())`; assign to `initial_content`.
   - **Scenario**: Agent handler prepends images to initial_content

7. **Test initial_content construction with images** — integration test that simulates agent stage execution with context containing human_answer_images, asserts CodergenRunRequest.initial_content starts with Image parts.
   - **Scenario**: Agent handler prepends images to initial_content

8. **Test initial_content construction without images** — integration test with no human_answer_images key, asserts initial_content is vec![Text(prompt)].
   - **Scenario**: Agent handler constructs text-only initial_content when no images

9. **Update CodergenRunRequest construction call site** — in agent handler execute method around line 290, change `prompt: &prompt` to `initial_content` (constructed in step 6).
   - **Scenarios**: Agent handler prepends images to initial_content, Agent handler constructs text-only initial_content when no images

10. **Test fabro-llm attachment resolution** — review `lib/crates/fabro-llm/src/attachments.rs` to confirm `resolve` function handles `ContentPart::Image` with `url: Some(...)` by loading file and inlining; write integration test if needed to prove file-to-inline resolution.
    - **Scenario**: Images reach LLM via fabro-llm attachment resolution

11. **Run full fabro-workflow tests** — `cargo nextest run -p fabro-workflow`, assert tests pass.
    - **Scenarios**: All scenarios in this slice

12. **Refactor** — if prior_stage_id logic is complex (requires graph traversal or event log lookup), extract to a dedicated helper `fn prior_stage_id(context: &Context) -> Option<String>`.

---

### Slice 6: Server handler and external call sites

**Depends-on**: Slice 2 (process_message API)

**Files**:
- lib/crates/fabro-server/src/server/handler/sessions.rs
- lib/crates/fabro-store/src/run_sessions.rs (if it constructs Message::User)

#### Scenarios

```gherkin
Scenario: Server sessions handler uses process_text_input
  Given sessions.rs calls session.process_input(input)
  When I update to session.process_text_input(input)
  Then the server handler compiles and sessions API works

Scenario: fabro-store constructs Message::User with Vec<ContentPart>
  Given run_sessions.rs constructs Message::User if it does
  When I update content field to vec![ContentPart::Text(...)]
  Then fabro-store compiles

Scenario: All fabro crates compile after migration
  Given all call sites updated
  When I run cargo build --workspace
  Then the build succeeds with no errors
```

#### Steps

1. **Update server sessions handler call site** — grep for `process_input\(` in `lib/crates/fabro-server/src/server/handler/sessions.rs`, replace with `process_text_input(...)`.
   - **Scenario**: Server sessions handler uses process_text_input

2. **Test server compiles** — run `cargo build -p fabro-server`, verify sessions handler compiles.
   - **Scenario**: Server sessions handler uses process_text_input

3. **Check fabro-store for Message::User construction** — grep for `Message::User \{` in `lib/crates/fabro-store/src/run_sessions.rs`, update any occurrences to use `content: vec![ContentPart::Text(...)]`.
   - **Scenario**: fabro-store constructs Message::User with Vec<ContentPart>

4. **Test fabro-store compiles** — run `cargo build -p fabro-store`, verify no errors.
   - **Scenario**: fabro-store constructs Message::User with Vec<ContentPart>

5. **Run workspace build** — `cargo build --workspace`, assert all crates compile.
   - **Scenario**: All fabro crates compile after migration

6. **Run workspace tests** — `cargo nextest run --workspace`, assert all tests pass (may have expected failures in E2E tests requiring credentials; focus on unit tests).
   - **Scenario**: All fabro crates compile after migration

7. **Refactor** — none (final integration verification).

---

### Slice 7: Documentation and plan completion

**Depends-on**: Slice 6 (all code changes complete)

**Files**:
- docs/public/reference/sdk.mdx (if it documents Session API)
- lib/crates/fabro-agent/README.md (if it shows process_input examples)

#### Scenarios

```gherkin
Scenario: SDK documentation reflects new API
  Given sdk.mdx documents Session usage
  When I update examples from process_input to process_text_input
  Then the documentation is accurate

Scenario: fabro-agent README reflects new API
  Given README.md has Session examples
  When I update to show process_message and process_text_input
  Then the README is accurate

Scenario: Plan is marked complete
  Given all slices implemented and tested
  When I verify acceptance criteria
  Then all criteria are met
  And the plan status is updated to "complete"
```

#### Steps

1. **Check SDK documentation for Session examples** — grep for `process_input` in `docs/public/reference/sdk.mdx`, update to `process_text_input` if present.
   - **Scenario**: SDK documentation reflects new API

2. **Check fabro-agent README for Session examples** — grep for `process_input` in `lib/crates/fabro-agent/README.md`, update to show `process_text_input` and optionally `process_message`.
   - **Scenario**: fabro-agent README reflects new API

3. **Verify acceptance criteria** — review each AC from the plan, run corresponding tests, assert all pass.
   - **Scenario**: Plan is marked complete

4. **Update plan status** — change plan status from "draft" to "complete".
   - **Scenario**: Plan is marked complete

5. **Refactor** — none (documentation-only slice).

---

## Parallelization

### Wave 0 (no dependencies)
- Slice 1: AgentMessage::User content type change

### Wave 1 (depends on Wave 0)
- Slice 2: Session::process_message API
- Slice 3: Update agent crate test call sites
- Slice 4: CodergenRunRequest and backend updates

### Wave 2 (depends on Wave 1)
- Slice 5: Agent handler image propagation (depends on Slice 4)
- Slice 6: Server handler and external call sites (depends on Slice 2)

### Wave 3 (depends on Wave 2)
- Slice 7: Documentation and plan completion (depends on Slice 6)

**File collision check**:
- Wave 1: Slice 2 (session.rs), Slice 3 (tests, subagent.rs, cli.rs), Slice 4 (agent.rs, api.rs, acp.rs, prompt.rs) — no overlap ✓
- Wave 2: Slice 5 (agent.rs, context.rs), Slice 6 (sessions.rs, run_sessions.rs) — Slice 5 touches agent.rs which was also touched by Slice 4 in Wave 1, but Wave 2 starts after Wave 1 completes, so no concurrent collision ✓

## Skipped (low value)

None. All spec findings were high-value (type changes, API refactors, call-site migrations, observable behavior changes). No low-value findings were identified.

## Risks & Open Questions

1. **Checkpoint compatibility**: Changing `Message::User` from `String` to `Vec<ContentPart>` breaks compatibility with existing checkpoints. Old checkpoints that contain `Message::User` with string content cannot be deserialized after the change. **Mitigation**: Fabro's checkpoint contract does not guarantee forward compatibility across codebase changes. Checkpoints are run-scoped and pruned with runs. Document this as a known breaking change; users resuming from old checkpoints after upgrade will encounter deserialization errors and must restart runs. This is acceptable per existing checkpoint design (see spec line 291).

2. **Prior stage detection**: Slice 5 assumes the workflow context contains `keys::LAST_STAGE` or similar to identify the immediately prior stage. If this key is unreliable or missing, the `extract_human_images` helper will fail to locate the correct context key. **Mitigation**: Review context population in the workflow engine (likely in `fabro-workflow/src/operations.rs` or similar); ensure `LAST_STAGE` is set after each stage completes. If `LAST_STAGE` does not exist, add it as a minimal context update. This should be verified in Slice 5 Step 5.

3. **PromptHandler text extraction**: Slice 4 Step 7 adds logic to extract text from `Vec<ContentPart>` for the PromptHandler fallback (non-agent models). This assumes that concatenating text parts in order is semantically correct. If the handler needs to handle images (e.g., for vision-capable prompt models), this extraction is lossy. **Mitigation**: For the initial implementation, PromptHandler ignores image parts (as documented in Slice 4). If future work adds vision support to PromptHandler, revisit this logic. Document the limitation in a code comment.

4. **Image file path assumptions**: The workflow handler in Slice 5 uses absolute file paths from context (`fabro.human_answer_images.<stage_id>`) and passes them as `ContentPart::Image(ImageData { url: Some(path), ... })`. The `fabro-llm` attachment resolver expects paths resolvable by the host process. If the workflow runs in a sandboxed environment where the agent process cannot access the run artifact directory, image resolution will fail. **Mitigation**: The current architecture assumes the workflow process and agent session run in the same host environment with shared filesystem access to the run directory. This is consistent with the Docker-based sandbox model (agent runs in a container, workflow process mounts run artifacts). No changes needed for the current design, but document this assumption in the code.

5. **Serialization schema drift**: Changing `Message::User` affects the `SessionMessage` serialization schema (used by the API and storage). The spec notes that `ContentPart` already implements `Serialize`/`Deserialize`, so serde handles the new structure automatically (spec line 291). However, clients consuming the API (e.g., fabro-web) may expect `SessionMessage.User.content` to be a string. **Mitigation**: The spec states this is a backend-only change for the initial implementation (the API boundary is not modified). If future work exposes multi-part User messages via the API, update the OpenAPI schema and regenerate clients. For now, server-side only; no API schema change needed.

6. **Test coverage for image resolution**: Slice 5 Step 10 reviews existing `fabro-llm/src/attachments.rs` logic but does not add new tests if coverage is sufficient. If attachment resolution has edge cases (missing files, permission errors, unsupported MIME types), they may not be caught. **Mitigation**: Review existing tests in `fabro-llm` for `attachments::resolve`. If coverage is insufficient, add integration test in Slice 5 that writes a test image file, constructs `ContentPart::Image` with file path, runs through agent session, and verifies inline data in LLM request. This is a "nice to have" rather than a blocker.

7. **Concurrent modifications to agent.rs**: Slice 4 and Slice 5 both modify `lib/crates/fabro-workflow/src/handler/agent.rs`. Slice 4 changes `CodergenRunRequest` and call sites (Wave 1); Slice 5 adds image propagation logic to the same execute method (Wave 2). These are sequential waves, so no true conflict, but the engineer implementing Slice 5 must rebase on Slice 4's changes. **Mitigation**: Ensure Slice 4 is fully merged and tested before starting Slice 5. Alternatively, implement both slices sequentially by the same engineer to avoid rebase overhead.

8. **Prompt text construction in agent handler**: Slice 5 Step 6 assumes `prompt` is already a `String` in the agent handler execute method. If the prompt is constructed from multiple context variables or templates, the logic to build `initial_content` must account for that. **Mitigation**: Review the prompt construction code in `handler/agent.rs` execute method (around line 260). Ensure the prompt is finalized as a `String` before constructing `initial_content`. If prompt construction is complex, refactor to ensure a single final `prompt: String` variable exists before Step 6.

## Plan Review Summary

The plan underwent parallel review by five specialized reviewers covering acceptance criteria, design integrity, UX impact, strategic alignment, and parallelization correctness. All five reviewers returned "succeeded" verdicts with no blocker-severity issues identified.

### Reviewer Verdicts

- **review_acceptance** (best outcome): Succeeded - confirmed all acceptance criteria are testable and covered by scenarios
- **review_design**: Succeeded - validated API design, migration strategy, and architectural coherence
- **review_ux**: Succeeded - verified developer experience impact and migration path clarity
- **review_strategic**: Succeeded - confirmed alignment with fabro-agent architecture and checkpoint model
- **review_parallel**: Succeeded - validated wave dependencies, file collision analysis, and slice boundaries

### Changes Applied

No changes were required. All reviewers found the plan ready for implementation with no blockers or critical warnings.

### Observations Preserved

The plan's existing "Risks & Open Questions" section already documents the key concerns raised during spec review:
- Checkpoint backward compatibility (accepted as intentional breaking change per checkpoint design)
- Prior stage detection relying on `keys::LAST_STAGE` context (verification deferred to Slice 5)
- PromptHandler text extraction being lossy for image parts (acceptable limitation documented)
- Image file path assumptions requiring shared filesystem access (consistent with current architecture)
- SessionMessage serialization schema impact (backend-only change, no API modification needed)

These observations remain unchanged as they represent known trade-offs consistent with the codebase's existing design principles.

## Build Progress

### Wave 0
- [x] Slice 1: AgentMessage::User content type change
  - [x] Change Message::User content field type
  - [x] Test Message::User with single text part compiles
  - [x] Test Message::User with image and text parts
  - [x] Update History::convert_to_messages User arm
  - [x] Test history conversion with multi-part User
  - [x] Update all internal Message::User construction sites
  - [x] Test internal constructions compile
  - [x] Test SessionMessage round-trip
  - [x] Refactor

### Wave 1
- [x] Slice 2: Session::process_message API
  - [x] Add Session::process_message method
  - [x] Implement process_message body
  - [x] Test process_message with single text part
  - [x] Test process_message with multi-part content
  - [x] Add Session::process_text_input convenience method
  - [x] Test process_text_input delegates correctly
  - [x] Remove process_input and process_input_with_runtime
  - [x] Test removal by attempting build
  - [x] Refactor
- [x] Slice 3: Update agent crate test call sites
  - [x] Update parity_matrix.rs call sites
  - [x] Test parity_matrix compiles and passes
  - [x] Update compaction.rs call sites
  - [x] Test compaction compiles and passes
  - [x] Update subagent.rs call site
  - [x] Test subagent compiles
  - [x] Update apply_patch.rs call site
  - [x] Test apply_patch compiles
  - [x] Update cli.rs call site
  - [x] Test CLI compiles
  - [x] Run full fabro-agent test suite
  - [x] Refactor
- [x] Slice 4: CodergenRunRequest and backend updates
  - [x] Change CodergenRunRequest.prompt to initial_content
  - [x] Test CodergenRunRequest compiles
  - [x] Update AgentApiBackend::run to use initial_content
  - [x] Test AgentApiBackend compiles
  - [x] Update AcpBackend::run to use initial_content
  - [x] Test AcpBackend compiles
  - [x] Update PromptHandler to extract text from initial_content
  - [x] Test PromptHandler compiles
  - [x] Run fabro-workflow tests
  - [x] Refactor

### Wave 2
- [x] Slice 5: Agent handler image propagation
  - [x] Add HUMAN_ANSWER_IMAGES_PREFIX constant
  - [x] Add extract_human_images helper
  - [x] Test extract_human_images with images in context
  - [x] Test extract_human_images returns None when key absent
  - [x] Identify prior stage from context
  - [x] Construct initial_content with images
  - [x] Test initial_content construction with images
  - [x] Test initial_content construction without images
  - [x] Update CodergenRunRequest construction call site
  - [x] Test fabro-llm attachment resolution
  - [x] Run full fabro-workflow tests
  - [x] Refactor
- [x] Slice 6: Server handler and external call sites
  - [x] Update server sessions handler call site
  - [x] Test server compiles
  - [x] Check fabro-store for Message::User construction
  - [x] Test fabro-store compiles
  - [x] Run workspace build
  - [x] Run workspace tests
  - [x] Refactor

### Wave 3
- [x] Slice 7: Documentation and plan completion
  - [x] Check SDK documentation for Session examples
  - [x] Check fabro-agent README for Session examples
  - [x] Verify acceptance criteria
  - [x] Update plan status
  - [x] Refactor
