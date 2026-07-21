# Freeform Response Image Attachments

## Intent Description

Users want to attach images to freeform text responses in human-in-the-loop questions, mirroring the experience available in Claude Code's prompt interface. Currently, freeform responses accept only plain text through a textarea component. This feature extends freeform responses to accept one or more image attachments alongside the text answer.

The primary use case is enabling users to provide visual context when answering workflow questions — screenshots of UI states, diagrams sketched on paper, error messages captured as images, or reference materials that help the agent understand the user's intent. This brings Fabro's human-in-the-loop interaction closer to the conversational richness of Claude Code's prompt interface.

This feature applies only to freeform response fields (`QuestionType::Freeform` and `allow_freeform` multiple-choice questions). Other question types (yes/no, confirmation, multiple choice without freeform) remain unchanged.

## Architecture Specification

### Frontend Changes (apps/fabro-web)

**Interview Dock Component** (`app/components/interview-dock.tsx`):
- Extend `FreeformBody` component (lines 393–469) to include an image attachment UI
- Add file input (hidden, triggered by button/icon) constrained to image MIME types (PNG, JPEG, GIF, WebP, HEIC)
- Add paste event handler on the textarea to capture clipboard image data
- When user pastes an image from clipboard (e.g., screenshot), automatically attach it
- Display attached images as thumbnail previews with remove action before submission
- Encode selected images as base64 data URLs in browser memory
- Maximum individual image size: 5 MB (configurable constant)
- Maximum total attachments per answer: 4 images (configurable constant)
- Show inline validation errors for oversized files or unsupported formats

**Answer Submission**:
- When user submits a freeform answer with images, construct answer payload with:
  - `kind: "text_with_images"` (new discriminator variant)
  - `text: string` (existing text content, required non-empty)
  - `images: Array<{ data: string, media_type: string }>` where `data` is base64-encoded image bytes

### API Changes

**OpenAPI Schema** (`docs/public/api-reference/fabro-api.yaml`):
- Add new `SubmitAnswerTextWithImagesRequest` schema under `components/schemas`
- Required fields: `kind: "text_with_images"`, `text: string` (minLength: 1), `images: array` (min 1 item)
- Each image object: `{ data: string (base64), media_type: string }`
- Add `text_with_images` mapping to `SubmitAnswerRequest` discriminator (line 9482)
- Server validation: reject if `text` is empty, if `images` array is empty, if any `data` field is not valid base64, or if decoded image exceeds size limits

**Generated Clients**:
- Re-run `cargo build -p fabro-api` to regenerate Rust types
- Re-run `cd lib/packages/fabro-api-client && bun run generate` for TypeScript client

### Backend Changes

**Interview Answer Processing** (`lib/crates/fabro-interview/src/lib.rs`):
- Extend `AnswerValue` enum with new variant:
  ```rust
  TextWithImages {
      text: String,
      images: Vec<ImageAttachment>,
  }
  ```
- Define `ImageAttachment` struct:
  ```rust
  pub struct ImageAttachment {
      pub data: Vec<u8>,       // decoded from base64
      pub media_type: String,  // e.g., "image/png"
  }
  ```
- Update `Answer::text_with_images(...)` constructor
- Update serde serialization to match API wire format

**Server Handler** (`lib/crates/fabro-server/src/server.rs`):
- Extend `submit_run_answer` handler to accept `SubmitAnswerTextWithImagesRequest`
- Decode base64 image data to `Vec<u8>`
- Validate decoded image size (reject if any single image > 5 MB after decoding)
- Construct `Answer` with `AnswerValue::TextWithImages`
- Pass to workflow engine via existing interviewer callback

**Workflow Context Propagation** (`lib/crates/fabro-workflow/src/handler/human.rs`):
- When a `TextWithImages` answer is received, store images as temporary files in run-local artifact storage with deterministic names (e.g., `human_<stage_id>_image_0.png`, `human_<stage_id>_image_1.jpg`)
- Inject image file paths into workflow context under a new key: `fabro.human_answer_images.<stage_id>` as a JSON array of absolute file paths

**Agent Integration** (`lib/crates/fabro-agent/src/lib.rs` or workflow engine's agent stage handler):
- When invoking an agent stage immediately after a human stage that provided images, automatically prepend the images to the agent's initial user message as `ContentPart::Image` entries with `url` pointing to the stored files
- Fabro's existing `fabro-llm` attachment resolution (`lib/crates/fabro-llm/src/attachments.rs`) already handles file-path-to-inline-data conversion before encoding for Anthropic/OpenAI/Gemini APIs
- The agent stage does not need explicit configuration to receive images; propagation is automatic if the prior stage is a human stage with image attachments
- Images are prepended before the text answer, allowing the agent to reference them naturally in its response

### Storage and Cleanup

- Images submitted with answers are stored in the run's artifact directory (same location as agent outputs and checkpoints)
- Images persist for the run's lifetime and are cleaned up when the run is pruned
- No separate image upload endpoint; images are submitted inline with answer requests

### Security and Validation

- Client-side: enforce max file size (5 MB) and image MIME type before encoding to base64
- Server-side: validate base64 encoding, re-check decoded size, and verify MIME type against allow-list (image/png, image/jpeg, image/gif, image/webp, image/heic)
- Reject requests with excessively large payloads (total POST body size limit enforced by Axum defaults)
- No path traversal risk: images are written with generated UUIDs to run-scoped artifact directory

### Backward Compatibility

- Existing `SubmitAnswerTextRequest` (`kind: "text"`) remains unchanged and continues to work
- `QuestionType::Freeform` questions presented before this feature ships will still accept text-only answers
- Workflow engine must gracefully handle both `AnswerValue::Text` and `AnswerValue::TextWithImages`
- New UI gracefully degrades if API returns error for `text_with_images` (falls back to text-only submission)

## Acceptance Criteria

1. **UI presents image attachment option**:
   - Freeform response fields (`QuestionType::Freeform` and `allow_freeform: true` multiple-choice questions) display an image attachment button or icon
   - Clicking the button opens a file picker constrained to image types
   - Pasting an image from the clipboard (e.g., screenshot) automatically attaches it without opening the file picker
   - Selected images appear as thumbnail previews with remove action
   - UI prevents submission if any image exceeds 5 MB or if more than 4 images are attached

2. **Images are submitted with freeform answers**:
   - Submitting a freeform answer with images sends `POST /api/v1/runs/{id}/questions/{qid}/answer` with `kind: "text_with_images"`, `text` (required non-empty string), and `images` array containing base64-encoded data and media types
   - Server returns HTTP 204 on successful submission
   - Server returns HTTP 400 with error message if text is empty, if images are invalid (bad base64, unsupported type, oversized), or if images array is empty

3. **Images are accessible to subsequent workflow stages**:
   - After a freeform answer with images is accepted, the images are persisted in the run's artifact directory with deterministic names
   - When an agent stage immediately follows a human stage that provided images, the images are automatically prepended to the agent's initial user message as `ContentPart::Image` attachments
   - Agent successfully receives images and can reference them in LLM API calls (verified by observing agent's response that demonstrates image understanding)
   - Images are also available via workflow context variable `fabro.human_answer_images.<stage_id>` for non-agent stages or advanced workflows

4. **Text-only answers continue to work**:
   - Submitting a freeform answer without images (using `kind: "text"`) continues to work as before
   - Existing workflows that do not use images are unaffected

5. **Error handling**:
   - Attempting to attach a non-image file shows an inline error in the UI
   - Attempting to attach an image larger than 5 MB shows an inline error in the UI before submission
   - Server-side validation errors (e.g., corrupted base64, wrong MIME type) are returned as HTTP 400 with actionable error messages displayed to the user

## Ambiguity Log

| Decision | Classification | Resolved By | Rationale / Answer |
|----------|---------------|-------------|-------------------|
| Should images be submitted inline (base64 in JSON) or via multipart upload? | inferable | Spec author | Inline base64 in JSON is consistent with Fabro's existing image handling in `fabro-llm` (ContentPart::Image supports both `url` and inline `data`). Avoids separate upload endpoint and simplifies client/server implementation. 5 MB limit per image keeps payload sizes manageable. |
| Should the text field be required or optional when images are present? | requires-stakeholder-input | human | required |
| How should images be presented to subsequent agent stages? | requires-stakeholder-input | human | "whatever is better" — interpreted as automatic propagation: images are stored as files in the run artifact directory AND automatically prepended to the agent's initial message when an agent stage immediately follows a human stage. This provides both automatic convenience (no workflow configuration needed) and flexibility (context variable available for advanced use cases). |
| What is the maximum number of images per answer? | inferable | Spec author | 4 images matches Claude Code's typical attachment limit and balances user flexibility with payload size constraints. |
| What is the maximum individual image size? | inferable | Spec author | 5 MB per image is standard for web file uploads and aligns with Claude API's base64 payload limits. |
| Should images be stored permanently or cleaned up with the run? | inferable | Spec author | Store images in run artifact directory and clean up with run pruning. Matches existing artifact lifecycle. No business requirement for long-term image storage outside of run context. |
| Should non-freeform question types (yes/no, confirmation) also support image attachments? | inferable | Spec author | No. The feature request specifically references "this field that I'm typing (freeform response)" and the use case is providing context for open-ended answers. Yes/no and confirmation questions are binary decisions that do not typically require visual context. |
| How should the UI handle attachment errors (network failure, validation error)? | inferable | Spec author | Display inline error messages below the attachment area. Do not clear the textarea or remove already-attached images unless the user explicitly removes them. This matches standard form validation UX patterns. |
| Should the backend re-encode images to a standard format (e.g., always convert to PNG)? | inferable | Spec author | No. Accept images as-is and pass them through to the LLM provider. Anthropic/OpenAI/Gemini APIs already accept multiple image formats. Re-encoding is computationally expensive and may degrade quality. |
| Should the API support updating an already-submitted answer to add images? | inferable | Spec author | No. The current API does not support answer updates (HTTP 409 if question is already answered). Adding update semantics is a separate feature. |
| Should the UI support pasting images from clipboard? | requires-stakeholder-input | human | "I take a screenshot and paste it to CC" — yes, add paste event handler on the textarea to capture clipboard images. When user pastes an image, automatically attach it. This mirrors Claude Code's paste-to-attach behavior and supports the common workflow of screenshot → paste → submit. |

## Cross-Artifact Consistency Gate

- [x] Intent is unambiguous — two developers would interpret it the same way.
- [x] Every behavior/goal in the intent maps to at least one acceptance criterion.
- [x] Architecture constrains implementation without over-engineering.
- [x] Same concepts named consistently across all three artifacts.
- [x] No artifact contradicts another.
- [x] Every gap/ambiguity finding is logged — inferable with rationale, or resolved by the human.
