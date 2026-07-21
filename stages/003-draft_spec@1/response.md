I'll explore the repository to understand the current architecture and then draft the specification for the Session API change.Now I have a clear picture. Let me check if there's an existing spec and then draft the specification.Now I understand the spec format. Let me create the specification document for the Session API change.Perfect! I've drafted the specification. Now let me review it once more to ensure completeness and check for any gaps or conflicts.

The specification I've created covers:

## Summary

I've drafted a comprehensive specification for enabling fabro-agent Session API to accept multi-part content (images + text) in the initial user message. The spec is located at:

**`docs/superpowers/specs/2026-07-21-agent-session-image-input.md`**

### Key Design Decisions

1. **`Message::User` structure change**: Changed from `content: String` to `content: Vec<ContentPart>` to align with the LLM-facing `LlmMessage` structure

2. **New Session API**: Replaced `process_input`/`process_input_with_runtime` with:
   - `process_message(content: Vec<ContentPart>, runtime: AgentToolRuntime)` — primary entry point
   - `process_text_input(input: &str)` — convenience wrapper for backward compatibility

3. **`CodergenRunRequest` update**: Changed `prompt: &str` to `initial_content: Vec<ContentPart>` to carry multi-part messages from workflow handler to session

4. **Workflow handler integration**: The agent handler constructs `Vec<ContentPart>` by prepending images from `fabro.human_answer_images.*` context keys, then appending the text prompt

5. **Minimal surface area**: No changes needed to fabro-llm attachment resolution (already handles `ContentPart::Image` with file URLs)

### Architecture Highlights

- **Least-invasive approach**: Changes only the message construction layer, not the agent loop, LLM codecs, or provider adapters
- **Clean migration path**: Existing text-only call sites use the new `process_text_input` wrapper with minimal code changes
- **Leverages existing infrastructure**: fabro-llm's attachment resolver already loads images from disk and inlines them during request encoding

### Ambiguity Resolution

All 10 design decisions in the Ambiguity Log were classified as **inferable** based on:
- Existing codebase patterns (ContentPart already used throughout fabro-llm)
- Rust ownership idioms (owned Vec vs borrowed slice)
- API consistency (single process_message method vs split methods)
- Established contracts (attachment resolution, serialization)

**No open questions — all findings were inferable.**