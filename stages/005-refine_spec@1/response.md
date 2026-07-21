I'll refine the specification based on the human's "proceed" response, validate the Cross-Artifact Consistency Gate, and update the spec file.Now I'll validate the Cross-Artifact Consistency Gate and append it to the spec file.

Let me verify the gate conditions:

1. **Intent is unambiguous** — The intent clearly describes the API change: replace `process_input(&str)` with `process_message(Vec<ContentPart>)`, change `Message::User` from `String` to `Vec<ContentPart>`, and thread images from workflow context through to LLM providers. Two developers would implement this the same way.

2. **Every behavior/goal maps to acceptance criteria** — 
   - "threads multi-part content from workflow to LLM" → AC 1, 3, 4
   - "backward-compatible text-only entry" → AC 2
   - "unblocks Slice 5" → AC 4
   - "migration of existing call sites" → AC 5
   - "image paths reach LLM providers" → AC 3
   - "serialization/resume" → AC 6

3. **Architecture constrains implementation** — The spec details exact struct changes (Message::User), method signatures (process_message), file locations (history.rs:100, handler/agent.rs:231-268), and data flow (workflow → backend → session → history → LLM).

4. **Consistent naming** — `ContentPart`, `Message::User`, `process_message`, `initial_content`, `CodergenRunRequest` are used consistently across all sections.

5. **No contradictions** — Architecture section 5 says "no changes required" for attachment resolution; AC 3 confirms images are resolved via the existing resolver. All sections agree on the owned `Vec<ContentPart>` type.

6. **All gaps logged** — 11 inferable decisions are documented with rationales. No open questions remain.

All gate conditions pass. I'll append the checklist and finalize the spec.# Agent Session Image Input API — design

**Status:** ready for approval
**Date:** 2026-07-21
**Related plan:** docs/plans/2026-07-21-freeform-image-attachments-plan.md

## Intent Description

Enable fabro-workflow's agent handler to pass images into the agent's initial user message when invoking a Session. Today, `Session::process_input` and `process_input_with_runtime` accept only `input: &str` and internally construct `Message::User { content: String, timestamp }`, which is then converted to a single-part `LlmMessage::user(content)` in the history-to-LLM conversion layer. This design makes it impossible to prepend image attachments that arrive via `CodergenRunRequest.initial_images` or workflow context keys (`fabro.human_answer_images.*`).

The Session API change threads multi-part content from the workflow handler through the agent session to the LLM providers. The agent's `Message::User` variant changes from a `String` payload to `Vec<ContentPart>`, matching the LLM-facing `LlmMessage` structure. The `process_input` methods are replaced by a single `process_message` entry point that accepts `Vec<ContentPart>`, and the history-to-LLM conversion in `history.rs` is updated to preserve the multi-part User content. A convenience `process_text_input` wrapper provides backward-compatible text-only entry for existing call sites.

This unblocks Slice 5 of the freeform-image-attachments plan, which stalled because the workflow handler (`handler/agent.rs`) had no way to inject images into the session's initial message.

## Architecture Specification

### 1. AgentMessage::User structure change

**File:** `lib/crates/fabro-agent/src/types.rs`

Change the `Message::User` variant from:

```rust
Message::User {
    content:   String,
    timestamp: SystemTime,
}
```

to:

```rust
Message::User {
    content:   Vec<ContentPart>,
    timestamp: SystemTime,
}
```

**Rationale:** The agent's internal history representation (`Message`) currently diverges from the LLM-facing representation (`LlmMessage`), requiring a lossy `String` → `Vec<ContentPart>` conversion in `history.rs:100`. By making `Message::User` accept `Vec<ContentPart>` directly, we:
- Eliminate the impedance mismatch between agent and LLM representations
- Enable the workflow handler to construct multi-part messages (images + text) before invoking the session
- Maintain alignment with the existing `LlmMessage` structure already used throughout fabro-llm

**Migration:** All existing call sites that construct `Message::User { content: string_value, timestamp }` must change to `Message::User { content: vec![ContentPart::Text(string_value)], timestamp }`. The `ContentPart` type is already exported from `fabro-llm::types` (re-exported from `fabro-types`) and is available in the agent crate.

### 2. Session::process_message entry point

**File:** `lib/crates/fabro-agent/src/session.rs`

Replace the existing `process_input` and `process_input_with_runtime` methods with:

```rust
pub async fn process_message(
    &mut self,
    content: Vec<ContentPart>,
    agent_tool_runtime: AgentToolRuntime,
) -> Result<(), Error>
```

**Behavior:**
1. Accepts a `Vec<ContentPart>` representing the full user message (images, text, or both)
2. Accepts an `AgentToolRuntime` for tool environment configuration (no longer optional — callers that don't need custom runtime pass `AgentToolRuntime::default()`)
3. Constructs `Message::User { content, timestamp: SystemTime::now() }` and pushes to history
4. Proceeds with the existing agent loop logic (build request, stream LLM, execute tools, iterate until completion)

**Convenience wrapper for text-only input:**

```rust
pub async fn process_text_input(
    &mut self,
    input: &str,
) -> Result<(), Error> {
    self.process_message(
        vec![ContentPart::Text(input.to_string())],
        AgentToolRuntime::default(),
    )
    .await
}
```

**Rationale:**
- Consolidates the two existing methods into a single entry point that directly accepts the message structure needed by the agent loop
- The `agent_tool_runtime` parameter is always required rather than having two separate methods, simplifying the API surface
- The text-only wrapper provides backward compatibility for call sites that don't need multi-part content

**Migration:** Existing call sites must update:
- `session.process_input(text)` → `session.process_text_input(text)`
- `session.process_input_with_runtime(text, runtime)` → `session.process_message(vec![ContentPart::Text(text.to_string())], runtime)`

### 3. History conversion update

**File:** `lib/crates/fabro-agent/src/history.rs`

Update the `Message::User` arm in `History::convert_to_messages` (line 100) from:

```rust
Message::User { content, .. } => LlmMessage::user(content),
```

to:

```rust
Message::User { content, .. } => LlmMessage {
    role:         Role::User,
    content:      content.clone(),
    name:         None,
    tool_call_id: None,
},
```

**Rationale:** With `Message::User` now holding `Vec<ContentPart>`, the conversion to `LlmMessage` becomes a direct copy rather than wrapping a string. The `LlmMessage::user(str)` convenience constructor wraps a string in a text part, which is no longer appropriate when the agent message is already multi-part.

### 4. Workflow handler call site

**File:** `lib/crates/fabro-workflow/src/handler/agent.rs`

In the `execute` method (around line 231-268), when constructing the initial agent input, change from:

```rust
// Old: pass prompt as &str
let req = CodergenRunRequest {
    node,
    prompt: &prompt,
    context,
    thread_id,
    emitter: &services.run.emitter,
    sandbox: &services.run.sandbox,
    tool_hooks,
    cancel_token: services.run.cancel_token.clone(),
    agent_tool_runtime,
};
let result = backend.run(req).await?;
```

to:

```rust
// New: construct Vec<ContentPart> for initial message
let mut initial_content = Vec::new();

// Prepend images from prior human stage if present
if let Some(image_paths) = extract_human_images(context) {
    for path in image_paths {
        initial_content.push(ContentPart::Image(ImageData {
            url:        Some(path),
            data:       None,
            media_type: None,
            detail:     None,
        }));
    }
}

// Append text prompt
initial_content.push(ContentPart::Text(prompt.clone()));

let req = CodergenRunRequest {
    node,
    initial_content, // Vec<ContentPart> instead of &str prompt
    context,
    thread_id,
    emitter: &services.run.emitter,
    sandbox: &services.run.sandbox,
    tool_hooks,
    cancel_token: services.run.cancel_token.clone(),
    agent_tool_runtime,
};
let result = backend.run(req).await?;
```

**Helper function:**

```rust
fn extract_human_images(context: &Context) -> Option<Vec<String>> {
    // Read fabro.human_answer_images.{prior_stage_id} from context
    // Return Some(paths) if present, None otherwise
    // Implementation details: use context.get(keys::LAST_STAGE) to identify
    // the prior stage, then look up f"fabro.human_answer_images.{stage_id}"
}
```

**Rationale:** This is the actual use case that motivates the API change. The workflow handler needs to inject images into the agent's initial message. With the new API, it constructs the full `Vec<ContentPart>` and passes it to the backend's `run()` method, which will call `session.process_message(initial_content, ...)`.

### 5. CodergenRunRequest update

**File:** `lib/crates/fabro-workflow/src/handler/agent.rs`

Change the `CodergenRunRequest` struct from:

```rust
pub struct CodergenRunRequest<'a> {
    pub node:               &'a Node,
    pub prompt:             &'a str, // Old: text-only
    pub context:            &'a Context,
    pub thread_id:          Option<&'a str>,
    pub emitter:            &'a Arc<Emitter>,
    pub sandbox:            &'a Arc<dyn Sandbox>,
    pub tool_hooks:         Option<Arc<dyn fabro_agent::ToolHookCallback>>,
    pub cancel_token:       CancellationToken,
    pub agent_tool_runtime: fabro_agent::AgentToolRuntime,
}
```

to:

```rust
pub struct CodergenRunRequest<'a> {
    pub node:               &'a Node,
    pub initial_content:    Vec<ContentPart>, // New: multi-part content
    pub context:            &'a Context,
    pub thread_id:          Option<&'a str>,
    pub emitter:            &'a Arc<Emitter>,
    pub sandbox:            &'a Arc<dyn Sandbox>,
    pub tool_hooks:         Option<Arc<dyn fabro_agent::ToolHookCallback>>,
    pub cancel_token:       CancellationToken,
    pub agent_tool_runtime: fabro_agent::AgentToolRuntime,
}
```

**Migration:** All backends that implement `CodergenBackend::run` must update to consume `req.initial_content` instead of `req.prompt` and pass it to `session.process_message(req.initial_content, req.agent_tool_runtime)`.

### 6. Backend implementations update

**Files:**
- `lib/crates/fabro-workflow/src/handler/agent/backend/session.rs` (SessionBackend)
- Any other CodergenBackend implementations

The `SessionBackend::run` method currently calls:

```rust
session.process_input_with_runtime(req.prompt, req.agent_tool_runtime).await?;
```

This must change to:

```rust
session.process_message(req.initial_content, req.agent_tool_runtime).await?;
```

**Rationale:** The backend is the bridge between the workflow layer (which constructs the multi-part content) and the agent session (which now accepts it directly). No intermediate parsing or conversion is needed.

### 7. Image resolution via fabro-llm attachments

**No changes required.** The existing `fabro-llm/src/attachments.rs` module already resolves `ContentPart::Image` with `url: Some(file_path)` to inline base64 data during request encoding. When the agent session builds an LLM request containing image parts with file paths, the codec's `resolve` function (called by each provider adapter) loads the files and rewrites the parts to `data: Some(bytes)`, `media_type: Some(detected_mime)`, `url: None`.

**Contract:** Images that fail to load are silently dropped (line 77 of `attachments.rs`). This is the existing behavior and remains acceptable for the initial implementation.

## Acceptance Criteria

1. **Session accepts multi-part initial message**:
   - `Session::process_message(vec![ContentPart::Image(...), ContentPart::Text(...)], runtime)` constructs a `Message::User` with the full content vector
   - The message is appended to the session history
   - The agent loop proceeds normally, converting the multi-part User message to an LLM request

2. **Text-only messages still work**:
   - `Session::process_text_input("hello")` constructs a single-text-part User message
   - Existing test cases that use text-only input compile and pass without modification after calling the new wrapper

3. **Images reach the LLM provider**:
   - When a User message contains `ContentPart::Image(ImageData { url: Some("/path/to/image.png"), ... })`, the history-to-LLM conversion preserves it
   - The fabro-llm attachment resolver loads the file and inlines it
   - The provider-specific encoder includes the image in the API request (verified via mock LLM client or captured HTTP request)

4. **Workflow handler can inject images**:
   - When `extract_human_images(context)` returns `Some(paths)`, the agent handler prepends `ContentPart::Image` entries to the initial content
   - The constructed `CodergenRunRequest` contains `initial_content: Vec<ContentPart>` with images followed by text
   - The backend's `run()` method passes the multi-part content to `session.process_message`

5. **Migration is mechanical**:
   - All existing call sites that construct `Message::User { content: String, ... }` are identified via grep
   - After updating to `content: vec![ContentPart::Text(...)]`, the codebase compiles
   - All existing agent tests pass (modulo call-site updates to use the new API)

6. **Agent session continues resumption support**:
   - Serializing and deserializing session history preserves multi-part User messages
   - Resuming a session from a checkpoint that contains multi-part User messages works correctly
   - Round-tripping through `SessionMessage` (via `Message::to_session_message` and `Message::from_session_message`) preserves image parts

## Ambiguity Log

| Decision | Classification | Resolved By | Rationale / Answer |
|----------|---------------|-------------|-------------------|
| Should `Message::User` use `Vec<ContentPart>` or introduce a new variant `Message::UserMultipart`? | inferable | Design consistency | Use `Vec<ContentPart>`. The LLM-facing `LlmMessage` already uses `content: Vec<ContentPart>` for all roles (line 280 of transcript.rs). Introducing a separate variant would create a third representation (String, Vec, or discriminated) and complicate the conversion layer. A single `Vec<ContentPart>` representation aligns the agent's internal model with the LLM model. |
| Should `process_message` have a separate `agent_tool_runtime` parameter or default it internally? | inferable | API simplicity | Always require `AgentToolRuntime` as a parameter. The existing `process_input` vs `process_input_with_runtime` split created two methods where one suffices. Callers that don't need custom runtime pass `AgentToolRuntime::default()`. This simplifies the API surface and makes the runtime parameter explicit at every call site. |
| Should `process_text_input` accept `&str` or `String`? | inferable | Rust idiom for builder methods | Accept `&str`. Text input is almost always borrowed from a larger context (prompt template, user message, etc.). Accepting `&str` avoids forcing callers to clone or move ownership. The method constructs the owned `ContentPart::Text(input.to_string())` internally. |
| Should the workflow handler's `extract_human_images` logic live in `handler/agent.rs` or in a separate context utility module? | inferable | Locality of use | Place it in `handler/agent.rs` as a module-private helper function. It is called by exactly one place (the agent execute method) and depends on workflow-specific context keys (`fabro.human_answer_images.*`). If a second handler (e.g., prompt handler) needs similar logic, refactor to a shared `context.rs` utility at that time. |
| Should `CodergenRunRequest.initial_content` be `Vec<ContentPart>` or `&[ContentPart]`? | inferable | Ownership and lifetime ergonomics | Use `Vec<ContentPart>` (owned). The workflow handler constructs the content vector dynamically (prepending images, appending text) and passes it to the backend, which then passes it to the session. Borrowing would require the vector to outlive the entire agent execution, tying the workflow handler's stack frame to the async backend call. Owned `Vec` allows the handler to build-and-move, and the session clones it into the history anyway (history must own its messages for serialization). |
| Should image paths in `ContentPart::Image` use absolute paths or relative-to-workspace paths? | inferable | Sandbox contract | Use absolute paths. The fabro-llm attachment resolver (`attachments.rs:70`) calls `common::load_file_bytes(url)`, which expects a file path resolvable by the host process. The workflow handler reads image paths from context keys (`fabro.human_answer_images.{stage_id}`), which are populated by the human handler as absolute paths to the run's artifact directory (per Slice 4 of the plan). The sandbox abstraction is not involved in image loading — the LLM client (running in the workflow process) loads files directly from the host filesystem. |
| Should `Session::process_message` validate that `content` is non-empty? | inferable | Existing contract and LLM behavior | No validation. The existing `process_input` methods do not reject empty strings. Passing an empty `Vec<ContentPart>` to the LLM is technically allowed by all provider APIs (though likely unproductive). The LLM will return a response (possibly an error or a polite refusal), which the agent loop already handles. Adding validation would be a behavior change relative to the existing API and is not required for the feature. |
| Should the history-to-LLM conversion deep-clone or shallow-clone `Vec<ContentPart>`? | inferable | Rust ownership semantics | Deep clone (via `.clone()` on `Vec<ContentPart>`). The history owns its messages (`Vec<Message>`), and the LLM request builder (`build_request`) constructs a separate `Vec<LlmMessage>` that outlives the history snapshot. `ContentPart` derives `Clone` (transcript.rs:126), so `.clone()` performs a deep copy of the content vector. This is acceptable because history conversion happens once per LLM request, not in a hot loop. |
| Should `ContentPart::Image` parts with `url: None, data: Some(...)` be preserved or rejected? | inferable | LLM codec contract | Preserve. The fabro-llm attachment resolver (attachments.rs:57-61) only rewrites parts where `url: Some(...)` points to a local file. Parts that already have inline data (`data: Some(...)`) pass through untouched. This means the workflow handler *could* construct `ContentPart::Image` with inline bytes if it wanted to (e.g., for testing or for images already in memory), and the codec would accept it. Rejecting such parts would be unnecessarily restrictive. |
| Should `Message::User` serialization change to accommodate `Vec<ContentPart>`? | inferable | Existing serialization contract | No changes needed. `Message::to_session_message` and `from_session_message` (types.rs) already serialize/deserialize the content field via serde. Changing the content type from `String` to `Vec<ContentPart>` is a data-level change, but `ContentPart` already implements `Serialize`/`Deserialize` (transcript.rs:141-230), so serde handles the new structure automatically. Checkpoints that contain the old `String`-based User messages are no longer compatible, but checkpointing compatibility across codebase changes is not guaranteed (checkpoints are keyed by run ID and discarded when runs are pruned). |

**No open questions — all findings were inferable.**

## Cross-Artifact Consistency Gate

- [x] Intent is unambiguous — two developers would interpret it the same way.
- [x] Every behavior/goal in the intent maps to at least one acceptance criterion.
- [x] Architecture constrains implementation without over-engineering.
- [x] Same concepts named consistently across all three artifacts.
- [x] No artifact contradicts another.
- [x] Every gap/ambiguity finding is logged — inferable with rationale, or resolved by the human.

---

**Specification file path:** `docs/superpowers/specs/2026-07-21-agent-session-image-input.md`