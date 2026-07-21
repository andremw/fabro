# Specification: Agent Session Image Input API

## Intent Description

The fabro-agent `Session` API currently accepts user input only as a text string (`&str`) through `process_input()` and `process_input_with_runtime()`. This text is internally converted to a `Message::User { content: String, ... }` variant that gets pushed onto the session's history. When the session builds LLM requests, it converts `Message::User` to `LlmMessage::user(content)`, which creates a single-text-ContentPart message.

The freeform-image-attachments feature (docs/plans/2026-07-21-freeform-image-attachments-plan.md) requires that when a human stage provides images alongside a text answer, those images must be automatically prepended to the next agent stage's initial user message as `ContentPart::Image` entries. Currently, there is no path for `CodergenRunRequest` to pass image attachments through the Session API into the LLM conversation.

This spec defines the minimal API changes needed to support multimodal user input (text + images) in fabro-agent sessions, enabling fabro-workflow's agent handler to construct rich initial messages from human-stage image attachments.

The change must preserve backward compatibility: all existing call sites that pass plain text strings must continue to work without modification.

## Architecture Specification

### 1. Session API Extension

#### 1a. New Input Type: `UserInput`

**File:** `lib/crates/fabro-agent/src/types.rs` (after line 72, near the `Message` enum)

Define a new input type that can represent either plain text or text with images:

```rust
/// User input for a session turn. Supports plain text or text with image attachments.
#[derive(Debug, Clone)]
pub enum UserInput {
    /// Plain text input (backward compatible with &str)
    Text(String),

    /// Multimodal input with optional leading images and required text
    WithImages {
        /// Images to prepend before the text content. Must be non-empty when using this variant.
        images: Vec<ImageData>,
        /// Text content. May be empty if images alone suffice, but workflow layer enforces non-empty text for human answers.
        text: String,
    },
}

impl<'a> From<&'a str> for UserInput {
    fn from(text: &'a str) -> Self {
        Self::Text(text.to_string())
    }
}

impl From<String> for UserInput {
    fn from(text: String) -> Self {
        Self::Text(text)
    }
}

impl UserInput {
    /// Create multimodal input with images and text.
    /// Images will be prepended before the text in the resulting message.
    pub fn with_images(images: Vec<ImageData>, text: impl Into<String>) -> Self {
        Self::WithImages {
            images,
            text: text.into(),
        }
    }

    /// Extract text content regardless of variant
    pub fn text(&self) -> &str {
        match self {
            Self::Text(s) => s.as_str(),
            Self::WithImages { text, .. } => text.as_str(),
        }
    }
}
```

**Rationale:**
- Enum provides type-safe distinction between text-only and multimodal input
- `From<&str>` and `From<String>` impls enable existing `process_input("text")` calls to work unchanged via Rust's implicit conversion
- `WithImages` variant ensures images are explicit in the type signature
- Helper methods provide ergonomic construction and access

#### 1b. Update Session Methods

**File:** `lib/crates/fabro-agent/src/session.rs` (lines 1210-1232)

Change method signatures to accept `impl Into<UserInput>`:

```rust
// Line 1210
pub async fn process_input(&mut self, input: impl Into<UserInput>) -> Result<(), Error> {
    self.process_input_with_runtime(input, AgentToolRuntime::default())
        .await
}

// Line 1228
pub async fn process_input_with_runtime(
    &mut self,
    input: impl Into<UserInput>,
    agent_tool_runtime: AgentToolRuntime,
) -> Result<(), Error> {
    let user_input = input.into();
    // ... existing timeout setup ...

    // Process the initial input, then drain any followups
    let mut result = self
        .run_single_input(user_input, &agent_tool_runtime, &mut timing, &mut usage)
        .await;
    // ... rest unchanged ...
}
```

**File:** `lib/crates/fabro-agent/src/session.rs` (line 1296)

Update `run_single_input` signature and implementation:

```rust
// Line 1296
async fn run_single_input(
    &mut self,
    input: UserInput,
    agent_tool_runtime: &AgentToolRuntime,
    timing: &mut SessionInputTiming,
    usage_accumulator: &mut TokenCounts,
) -> Result<(), Error> {
    const STREAM_CONSUME_RETRIES: usize = 3;

    if self.state == SessionState::Closed {
        return Err(Error::SessionClosed);
    }

    self.transition(SessionState::Thinking);

    // Expand skill references in input text
    let input_text = input.text();
    let expanded = if self.skills.is_empty() {
        ExpandedInput {
            text:       input_text.to_string(),
            skill_name: None,
        }
    } else {
        expand_skill(&self.skills, input_text).map_err(Error::InvalidState)?
    };
    if let Some(ref name) = expanded.skill_name {
        self.activated_skill_context_observed = true;
        self.event_emitter
            .emit(self.id.clone(), AgentEvent::SkillActivated {
                skill_name: name.clone(),
                source:     SkillActivationSource::Slash,
            });
    }
    let expanded_input_text = expanded.text;

    // Construct Message::User with optional images
    // Line 1331 (current: self.history.push(Message::User { content: expanded_input.clone(), ... }))
    let user_message = match input {
        UserInput::Text(_) => Message::User {
            content: expanded_input_text.clone(),
            timestamp: SystemTime::now(),
        },
        UserInput::WithImages { images, .. } => Message::UserWithImages {
            images,
            content: expanded_input_text.clone(),
            timestamp: SystemTime::now(),
        },
    };

    self.history.push(user_message);
    self.event_emitter
        .emit(self.id.clone(), AgentEvent::UserInput {
            text: expanded_input_text.clone(),
        });

    // ... rest of loop unchanged ...
}
```

**Rationale:**
- `impl Into<UserInput>` allows call sites to pass `&str`, `String`, or `UserInput` directly
- Existing `session.process_input("text")` calls work unchanged via `From<&str>` impl
- Skill expansion operates on text only (skills are invoked via text, not images)
- Message construction branches on input type

### 2. Message Enum Extension

#### 2a. Add UserWithImages Variant

**File:** `lib/crates/fabro-agent/src/types.rs` (lines 38-72)

Extend the `Message` enum to support user messages with images:

```rust
#[derive(Debug, Clone)]
pub enum Message {
    User {
        content:   String,
        timestamp: SystemTime,
    },
    /// User message with leading image attachments.
    /// Images are prepended before text when converting to LlmMessage.
    UserWithImages {
        images:    Vec<ImageData>,
        content:   String,
        timestamp: SystemTime,
    },
    Assistant {
        content:        String,
        tool_calls:     Vec<ToolCall>,
        provider_parts: Vec<ContentPart>,
        usage:          Box<TokenCounts>,
        response_id:    String,
        timestamp:      SystemTime,
    },
    ToolResults {
        results:   Vec<ToolResult>,
        timestamp: SystemTime,
    },
    System {
        content:   String,
        timestamp: SystemTime,
    },
    Steering {
        content:   String,
        timestamp: SystemTime,
    },
}
```

**Rationale:**
- Separate `UserWithImages` variant preserves existing `User` variant for backward compatibility
- `ImageData` is already imported from `fabro_llm::types` (line 4 in types.rs)
- Structure mirrors `User` variant with added `images` field
- Timestamp and content remain consistent with existing user messages

#### 2b. Update SessionMessage Conversion

**File:** `lib/crates/fabro-agent/src/types.rs` (lines 93-128)

Extend `to_session_message()` and `from_session_message()` to handle the new variant:

```rust
// In to_session_message(), after the User arm (line 95-98):
Self::UserWithImages { images, content, timestamp } => SessionMessage::UserWithImages {
    images:    values_or_empty(images),
    content:   content.clone(),
    timestamp: system_time_to_utc(*timestamp),
},

// In from_session_message(), after the User arm (line 132-135):
SessionMessage::UserWithImages { images, content, timestamp } => Self::UserWithImages {
    images:    values_from_json(images)?,
    content:   content.clone(),
    timestamp: utc_to_system_time(*timestamp),
},
```

**Note:** This assumes `SessionMessage` enum in `lib/crates/fabro-types/src/session.rs` will also be extended with a `UserWithImages` variant following the same pattern. If `SessionMessage` is not used for image-bearing turns, this conversion can serialize images as JSON in the existing `User` variant's content field or omit images entirely from the serialized session state (images are transient and only needed during the session lifecycle).

**Alternative if SessionMessage extension is deferred:** Serialize `UserWithImages` as `SessionMessage::User` with a JSON-encoded content string that includes image metadata placeholders. This is a degradation but maintains compatibility.

#### 2c. Update History Conversion to LlmMessage

**File:** `lib/crates/fabro-agent/src/history.rs` (lines 96-146)

Extend `convert_to_messages()` to handle `UserWithImages`:

```rust
// In convert_to_messages(), after the existing User arm (line 100):
Message::User { content, .. } => LlmMessage::user(content),
Message::UserWithImages { images, content, .. } => {
    let mut parts: Vec<ContentPart> = Vec::new();
    // Prepend images before text
    for image in images {
        parts.push(ContentPart::Image(image.clone()));
    }
    if !content.is_empty() {
        parts.push(ContentPart::text(content));
    }
    LlmMessage {
        role:         Role::User,
        content:      parts,
        name:         None,
        tool_call_id: None,
    }
}
```

**Rationale:**
- Images are prepended before text, ensuring LLM sees visual context first
- Empty text is supported (some use cases may send images alone), but workflow layer enforces non-empty text for human answers
- Matches the pattern used for `Assistant` messages with provider_parts + text + tool calls (lines 101-124)

### 3. Workflow Integration: CodergenRunRequest Extension

**File:** `lib/crates/fabro-workflow/src/handler/agent.rs` (lines 39-50)

Extend `CodergenRunRequest` to carry image attachments:

```rust
pub struct CodergenRunRequest<'a> {
    pub node:               &'a Node,
    pub prompt:             &'a str,
    pub context:            &'a Context,
    pub thread_id:          Option<&'a str>,
    pub emitter:            &'a Arc<Emitter>,
    pub sandbox:            &'a Arc<dyn Sandbox>,
    pub tool_hooks:         Option<Arc<dyn fabro_agent::ToolHookCallback>>,
    pub cancel_token:       CancellationToken,
    pub agent_tool_runtime: fabro_agent::AgentToolRuntime,
    /// Optional image attachments to prepend to the initial user message.
    /// Populated by the workflow layer when the prior stage provided images via human-in-the-loop.
    pub initial_images:     Option<Vec<ImageData>>,
}
```

**File:** `lib/crates/fabro-workflow/src/handler/llm/api.rs` (lines 1271-1280)

Update `AgentApiBackend::run()` to construct multimodal input when images are present:

```rust
// Current (line 1271-1280):
let process_result = session
    .process_input_with_runtime(prompt, agent_tool_runtime.clone())
    .await;

// Updated:
let user_input = if let Some(images) = request.initial_images {
    if images.is_empty() {
        // Empty images vec: fall back to plain text
        UserInput::from(prompt)
    } else {
        UserInput::with_images(images, prompt)
    }
} else {
    UserInput::from(prompt)
};

let process_result = session
    .process_input_with_runtime(user_input, agent_tool_runtime.clone())
    .await;
```

**Rationale:**
- `CodergenRunRequest.initial_images` provides the path for human-stage images to reach the session
- Backend constructs `UserInput::WithImages` when images are present
- Prompt text is combined with images in the same user message (not separate messages)
- Empty or missing `initial_images` preserves existing text-only behavior

### 4. Workflow Layer: Populating initial_images

**File:** `lib/crates/fabro-workflow/src/handler/agent.rs` (executor or handler that builds CodergenRunRequest)

The workflow layer is responsible for detecting when the prior stage was a human stage with images, reading the image files from the artifact directory (paths stored in context under `fabro.human_answer_images.<stage_id>`), and populating `CodergenRunRequest.initial_images`.

**Example pseudocode (actual implementation in Slice 5 of the plan):**

```rust
// In AgentHandler::execute() or equivalent, before calling backend.run():
let initial_images = if let Some(prior_stage_id) = prior_human_stage_id(context) {
    if let Some(image_paths) = context.get(&format!("fabro.human_answer_images.{}", prior_stage_id)) {
        let paths: Vec<String> = serde_json::from_value(image_paths.clone())?;
        Some(
            paths.into_iter()
                .map(|path| ImageData {
                    url: Some(path),
                    data: None,
                    media_type: None,
                    detail: None,
                })
                .collect()
        )
    } else {
        None
    }
} else {
    None
};

let result = backend
    .run(CodergenRunRequest {
        // ... existing fields ...
        initial_images,
    })
    .await;
```

**Note:** This pseudocode is illustrative. The actual implementation will be in Slice 5 of the plan and must handle:
- Detecting the prior stage from context
- Checking if it was a human stage
- Reading the `fabro.human_answer_images.<stage_id>` context key
- Constructing `ImageData` with file URLs
- Passing to `CodergenRunRequest`

The fabro-llm attachment resolution layer (lib/crates/fabro-llm/src/attachments.rs) already handles converting `ImageData { url: Some(path), ... }` to inline base64 data before encoding for LLM providers.

### 5. Type Dependencies and Imports

**Required imports/dependencies:**

- `fabro-agent` already imports `ContentPart` and `ImageData` from `fabro_llm::types` (which re-exports from `fabro-types`)
- No new crate dependencies required
- `UserInput` type must be exported from `fabro-agent` public API for workflow layer to use

**Public API exports:**

**File:** `lib/crates/fabro-agent/src/lib.rs`

Ensure `UserInput` is re-exported:

```rust
pub use types::{
    AgentEvent, Error as AgentError, MemoryFileSummary, Message, MessageHistory, SessionState,
    SkillActivationSource, SkillSummary, UserInput,  // ← Add UserInput
};
```

### 6. Backward Compatibility

**Call sites that continue to work unchanged:**

All existing call sites that pass `&str` or `String` to `process_input()` or `process_input_with_runtime()` will compile and run unchanged because:

1. `impl Into<UserInput>` accepts `&str` and `String` via the `From` impls
2. `UserInput::Text(...)` is constructed automatically
3. `Message::User` variant is preserved and continues to convert to single-text LlmMessage

**Examples:**

```rust
// Existing code (unchanged):
session.process_input("Hello, agent").await?;
session.process_input_with_runtime("Run tests", runtime).await?;

// New code (explicit UserInput):
session.process_input(UserInput::with_images(images, "Fix the bug")).await?;

// New code (workflow layer passing CodergenRunRequest.initial_images):
// Backend constructs UserInput internally, no change to workflow call site beyond adding initial_images field
```

**Migration path for tests:**

- Unit tests that construct `Message::User` directly: no change required
- Integration tests that call `process_input()`: no change required
- Tests that inspect session history: must handle both `Message::User` and `Message::UserWithImages` in match arms

## Acceptance Criteria

1. **Session API accepts text-only input unchanged**:
   - Calling `session.process_input("text")` with a string slice works without modification
   - Calling `session.process_input_with_runtime(prompt, runtime)` with a string works unchanged
   - The resulting `Message::User` variant is pushed to history
   - The session converts `Message::User` to a single-text `LlmMessage::user(content)`

2. **Session API accepts multimodal input**:
   - Calling `session.process_input(UserInput::with_images(images, "text"))` succeeds
   - The resulting `Message::UserWithImages` variant is pushed to history with both images and text
   - Images are stored in the `images` field as `Vec<ImageData>`
   - The session converts `Message::UserWithImages` to an `LlmMessage` with `content: [Image(...), Image(...), Text(...)]`

3. **Images are prepended before text in LlmMessage**:
   - When `convert_to_messages()` processes a `Message::UserWithImages` with 2 images and text "Fix this", the resulting `LlmMessage.content` is `[Image(img1), Image(img2), Text("Fix this")]`
   - The order is deterministic: images in array order, followed by text if non-empty

4. **Workflow backend constructs multimodal input from CodergenRunRequest**:
   - When `CodergenRunRequest.initial_images` is `Some(vec![img])`, the backend constructs `UserInput::with_images(vec![img], prompt)`
   - When `CodergenRunRequest.initial_images` is `None` or `Some(vec![])`, the backend constructs `UserInput::Text(prompt)`
   - The backend passes the constructed `UserInput` to `session.process_input_with_runtime()`

5. **LLM providers receive images inline**:
   - When the session sends a request to fabro-llm with `LlmMessage { role: User, content: [Image(url: Some(path)), Text("...")] }`, fabro-llm's attachment resolution reads the file at `path`, encodes it as base64, and rewrites the `ImageData` to `{ data: Some(bytes), media_type: Some(detected), url: None }`
   - The LLM provider (Anthropic/OpenAI/Gemini) receives the inline image in the request body
   - (This behavior already exists in fabro-llm; this AC verifies the end-to-end flow)

6. **Backward compatibility preserved**:
   - All existing unit tests in `lib/crates/fabro-agent/src/session.rs` that call `process_input("text")` pass without modification
   - All existing integration tests in `lib/crates/fabro-workflow` that invoke agent stages pass without modification
   - The `Message::User` variant remains unchanged in structure and serialization

7. **SessionMessage serialization handles UserWithImages**:
   - When `Message::UserWithImages` is converted via `to_session_message()`, it produces a `SessionMessage::UserWithImages` with images serialized as JSON (OR it gracefully degrades to `SessionMessage::User` if the SessionMessage enum is not extended)
   - Round-trip serialization preserves image data and metadata
   - (If SessionMessage extension is deferred, this AC is replaced with: "UserWithImages messages are serialized to SessionMessage::User with image metadata in a documented format")

## Ambiguity Log

| Decision | Classification | Resolved By | Rationale / Answer |
|----------|---------------|-------------|-------------------|
| Should `UserInput` be an enum or a struct with `Option<Vec<ImageData>>`? | inferable | Spec author | Enum is more type-safe and explicit. `UserInput::Text` vs `UserInput::WithImages` clearly distinguishes the two cases. A struct with `Option<images>` would require runtime validation to ensure non-empty images when present, whereas the enum makes invalid states unrepresentable. |
| Should the new variant be `Message::UserWithImages` or extend `Message::User` to `User { content: Vec<ContentPart>, ... }`? | inferable | Spec author | Separate `UserWithImages` variant is less invasive. Changing `Message::User.content` from `String` to `Vec<ContentPart>` would break every match arm on `Message::User` in the codebase (session.rs, history.rs, tests). A new variant isolates the change and preserves backward compatibility. |
| Should images be prepended or appended to the text in `LlmMessage.content`? | inferable | Spec author | Prepended. LLM models (Anthropic, OpenAI, Gemini) conventionally expect visual context before text in multimodal prompts. This matches the pattern in Claude Code and user mental models ("here are screenshots, now here's my question"). |
| Should `UserInput::with_images()` accept empty text? | inferable | Spec author | Yes, the API allows it (empty text is semantically valid for image-only prompts), but the workflow layer enforces non-empty text for human answers per the freeform-image-attachments spec. This separation of concerns allows the Session API to be general-purpose while the workflow layer applies business rules. |
| Should `CodergenRunRequest.initial_images` be `Option<Vec<ImageData>>` or `Vec<ImageData>`? | inferable | Spec author | `Option<Vec<ImageData>>` distinguishes "no images provided" (None) from "images field is empty" (Some(vec![])). Simpler to check `if let Some(images) = request.initial_images` than to check `if !request.initial_images.is_empty()`. Allows future extension where an empty vec might have distinct semantics. |
| How should skill expansion interact with multimodal input? | inferable | Spec author | Skill expansion operates only on the text portion of `UserInput`. Skills are invoked via text commands (e.g., "/commit"), not images. The `expand_skill()` function receives `input.text()` and returns expanded text, which is combined with the original images in `Message::UserWithImages`. |
| Should `Message::UserWithImages` be serialized to a new `SessionMessage::UserWithImages` variant or degraded to `SessionMessage::User`? | requires-stakeholder-input | human | Preferred: extend SessionMessage enum. Fallback: serialize images as JSON metadata in User content. |
| Should `from_session_message()` for UserWithImages fail on missing/invalid image data or silently drop images? | inferable | Spec author | Fail with serde_json::Error. Images are critical to the message semantics; silently dropping them would cause agent stages to lose visual context, leading to incorrect behavior. Better to surface deserialization errors early via `values_from_json(images)?`. |
| Should the Session API validate that `UserInput::WithImages.images` is non-empty? | inferable | Spec author | No. The API accepts empty `images` vecs (degrades to text-only), but logs a warning if this occurs. The workflow layer is responsible for ensuring images are non-empty before constructing `UserInput::WithImages`. Runtime validation in the Session API would add overhead and redundant checks. |
| Should followup messages (auto-injected by session after first turn) support images? | inferable | Spec author | No. Followup messages are internal steering or continuation prompts generated by the session itself (e.g., loop detection, compaction prompts). They are always text-only. Only the initial `process_input()` call can carry images. Followups use the existing `String` type and are processed via `UserInput::Text`. |

## Open Questions

1. Should `Message::UserWithImages` be serialized to a new `SessionMessage::UserWithImages` variant or degraded to `SessionMessage::User`?
