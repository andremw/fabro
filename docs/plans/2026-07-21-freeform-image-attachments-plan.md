# Plan: Freeform Response Image Attachments

**Status**: in-progress
**Spec**: docs/specs/freeform-image-attachments.md

## Goal

Enable users to attach images to freeform text responses in human-in-the-loop questions, mirroring the experience in Claude Code. Users should be able to select images via file picker and paste screenshots directly from clipboard. Images are submitted inline with answers, stored in run artifacts, and automatically propagated to subsequent agent stages.

## Acceptance Criteria

1. **UI presents image attachment option**:
   - Freeform response fields (QuestionType::Freeform and allow_freeform: true multiple-choice questions) display an image attachment button or icon
   - Clicking the button opens a file picker constrained to image types
   - Pasting an image from the clipboard (e.g., screenshot) automatically attaches it without opening the file picker
   - Selected images appear as thumbnail previews with remove action
   - UI prevents submission if any image exceeds 5 MB or if more than 4 images are attached

2. **Images are submitted with freeform answers**:
   - Submitting a freeform answer with images sends POST /api/v1/runs/{id}/questions/{qid}/answer with kind: "text_with_images", text (required non-empty string), and images array containing base64-encoded data and media types
   - Server returns HTTP 204 on successful submission
   - Server returns HTTP 400 with error message if text is empty, if images are invalid (bad base64, unsupported type, oversized), or if images array is empty

3. **Images are accessible to subsequent workflow stages**:
   - After a freeform answer with images is accepted, the images are persisted in the run's artifact directory with deterministic names
   - When an agent stage immediately follows a human stage that provided images, the images are automatically prepended to the agent's initial user message as ContentPart::Image attachments
   - Agent successfully receives images and can reference them in LLM API calls (verified by observing agent's response that demonstrates image understanding)
   - Images are also available via workflow context variable fabro.human_answer_images.<stage_id> for non-agent stages or advanced workflows

4. **Text-only answers continue to work**:
   - Submitting a freeform answer without images (using kind: "text") continues to work as before
   - Existing workflows that do not use images are unaffected

5. **Error handling**:
   - Attempting to attach a non-image file shows an inline error in the UI
   - Attempting to attach an image larger than 5 MB shows an inline error in the UI before submission
   - Server-side validation errors (e.g., corrupted base64, wrong MIME type) are returned as HTTP 400 with actionable error messages displayed to the user

## Slices

### Slice 1: OpenAPI schema and type generation

**Depends-on**: none

**Files**:
- docs/public/api-reference/fabro-api.yaml
- lib/crates/fabro-api/build.rs (regenerated)
- lib/crates/fabro-api/src/* (generated)
- lib/packages/fabro-api-client/src/* (generated)

#### Scenarios

```gherkin
Scenario: Define text_with_images schema in OpenAPI spec
  Given the OpenAPI spec at docs/public/api-reference/fabro-api.yaml
  When I add SubmitAnswerTextWithImagesRequest schema under components/schemas
  And the schema requires kind: "text_with_images", text (minLength: 1), and images (minItems: 1)
  And each image object has data (base64 string) and media_type (string)
  And I add "text_with_images": "#/components/schemas/SubmitAnswerTextWithImagesRequest" to the SubmitAnswerRequest discriminator mapping
  Then the schema is valid OpenAPI 3.x

Scenario: Generate Rust types from updated OpenAPI spec
  Given the updated fabro-api.yaml with SubmitAnswerTextWithImagesRequest
  When I run cargo build -p fabro-api
  Then progenitor generates SubmitAnswerTextWithImagesRequest Rust type
  And SubmitAnswerRequest enum includes TextWithImagesRequest variant
  And the generated types compile without errors

Scenario: Generate TypeScript client from updated OpenAPI spec
  Given the updated fabro-api.yaml
  When I run cd lib/packages/fabro-api-client && bun run generate
  Then the TypeScript client includes SubmitAnswerTextWithImagesRequest type
  And SubmitAnswerRequest union type includes text_with_images discriminator
  And the generated client compiles without errors
```

#### Steps

1. **Add SubmitAnswerTextWithImagesRequest schema to OpenAPI spec** — define schema with required kind, text (minLength: 1), and images array (minItems: 1); each image object has data (base64 string) and media_type (string); insert after SubmitAnswerTextRequest around line 9550 in fabro-api.yaml.
   - **Scenario**: Define text_with_images schema in OpenAPI spec

2. **Add text_with_images to discriminator mapping** — insert "text_with_images": "#/components/schemas/SubmitAnswerTextWithImagesRequest" in SubmitAnswerRequest discriminator mapping at line 9482.
   - **Scenario**: Define text_with_images schema in OpenAPI spec

3. **Test OpenAPI spec validity** — validate the YAML syntax and OpenAPI 3.x compliance using a validator or cargo build -p fabro-api.
   - **Scenario**: Define text_with_images schema in OpenAPI spec

4. **Generate Rust types** — run cargo build -p fabro-api to regenerate Rust types and client; verify SubmitAnswerRequest enum now includes TextWithImagesRequest variant; verify compilation succeeds.
   - **Scenario**: Generate Rust types from updated OpenAPI spec

5. **Generate TypeScript client** — run cd lib/packages/fabro-api-client && bun run generate; verify SubmitAnswerRequest union type includes text_with_images; verify compilation succeeds with bun run typecheck.
   - **Scenario**: Generate TypeScript client from updated OpenAPI spec

6. **Refactor** — none for this slice (pure code generation).

### Slice 2: fabro-interview domain types for image attachments

**Depends-on**: none

**Files**:
- lib/crates/fabro-interview/src/lib.rs

#### Scenarios

```gherkin
Scenario: Define ImageAttachment struct
  Given fabro-interview/src/lib.rs
  When I define pub struct ImageAttachment with data: Vec<u8> and media_type: String
  And I derive Debug, Clone, PartialEq, Eq, Serialize, Deserialize
  Then the struct compiles and can be serialized/deserialized

Scenario: Extend AnswerValue enum with TextWithImages variant
  Given the existing AnswerValue enum at line 52-62 in fabro-interview/src/lib.rs
  When I add TextWithImages { text: String, images: Vec<ImageAttachment> } variant
  Then the enum compiles
  And existing variants (Yes, No, Text, etc.) are unaffected

Scenario: Add Answer::text_with_images constructor
  Given the Answer struct and its constructors around lines 72-151
  When I implement pub fn text_with_images(text: impl Into<String>, images: Vec<ImageAttachment>) -> Self
  And it returns Answer { value: AnswerValue::TextWithImages { text: t.clone(), images }, selected_option: None, text: Some(t) }
  Then calling Answer::text_with_images("hello", vec![...]) returns a valid Answer
  And the text field is populated with the same text as the value

Scenario: TextWithImages answer serializes to expected JSON shape
  Given an Answer constructed with text_with_images("test", vec![ImageAttachment { data: vec![1,2,3], media_type: "image/png".into() }])
  When I serialize it with serde_json::to_value
  Then the JSON includes value.text == "test"
  And value.images[0].data is the byte array
  And value.images[0].media_type == "image/png"
  And the text field (Answer's top-level text) == Some("test")
```

#### Steps

1. **Define ImageAttachment struct** — add pub struct ImageAttachment { pub data: Vec<u8>, pub media_type: String } with Debug, Clone, PartialEq, Eq, Serialize, Deserialize derives; place near AnswerValue definition around line 50.
   - **Scenario**: Define ImageAttachment struct

2. **Test ImageAttachment serialization** — write unit test that constructs ImageAttachment, serializes to JSON, deserializes back, asserts round-trip equality.
   - **Scenario**: Define ImageAttachment struct

3. **Add TextWithImages variant to AnswerValue** — insert TextWithImages { text: String, images: Vec<ImageAttachment> } into AnswerValue enum at line 62.
   - **Scenario**: Extend AnswerValue enum with TextWithImages variant

4. **Test TextWithImages variant in pattern matching** — add unit test that constructs AnswerValue::TextWithImages, matches on it, extracts text and images, asserts values.
   - **Scenario**: Extend AnswerValue enum with TextWithImages variant

5. **Implement Answer::text_with_images constructor** — add pub fn text_with_images(text: impl Into<String>, images: Vec<ImageAttachment>) -> Self around line 151; set value, selected_option: None, text: Some(t.clone()).
   - **Scenario**: Add Answer::text_with_images constructor

6. **Test text_with_images constructor** — write unit test that calls Answer::text_with_images("test", vec![ImageAttachment { data: vec![1,2,3], media_type: "image/png".into() }]), asserts value is TextWithImages, text field is Some("test").
   - **Scenario**: Add Answer::text_with_images constructor, TextWithImages answer serializes to expected JSON shape

7. **Test TextWithImages serialization round-trip** — serialize Answer with text_with_images to JSON, deserialize, assert equality including images vector.
   - **Scenario**: TextWithImages answer serializes to expected JSON shape

8. **Refactor** — review all existing AnswerValue match arms (e.g., in answer_text helper if it exists) to ensure TextWithImages is handled or explicitly skipped; add exhaustive match test if pattern matching is critical.

### Slice 3: Server-side answer request mapping and validation

**Depends-on**: Slice 1 (OpenAPI types), Slice 2 (fabro-interview types)

**Files**:
- lib/crates/fabro-server/src/server.rs (answer_from_request, validate_answer_for_question)

#### Scenarios

```gherkin
Scenario: Map SubmitAnswerTextWithImagesRequest to Answer
  Given answer_from_request function at line 3810 in fabro-server/src/server.rs
  When the request is SubmitAnswerRequest::TextWithImagesRequest with text "hello" and images [{ data: base64("test"), media_type: "image/png" }]
  And I decode base64 data to Vec<u8>
  And I construct ImageAttachment { data, media_type }
  And I call Answer::text_with_images(text, images)
  Then the function returns Ok(Answer) with AnswerValue::TextWithImages

Scenario: Reject text_with_images with empty text
  Given a SubmitAnswerTextWithImagesRequest with text "" and valid images
  When answer_from_request processes it
  Then it returns Err(ApiError::bad_request("Text is required when submitting images."))

Scenario: Reject text_with_images with empty images array
  Given a SubmitAnswerTextWithImagesRequest with valid text and images []
  When answer_from_request processes it
  Then it returns Err(ApiError::bad_request("At least one image is required for text_with_images answer."))

Scenario: Reject text_with_images with invalid base64
  Given a SubmitAnswerTextWithImagesRequest with images containing data that is not valid base64
  When answer_from_request decodes the data
  Then it returns Err(ApiError::bad_request("Invalid base64 image data."))

Scenario: Reject text_with_images with oversized decoded image
  Given a SubmitAnswerTextWithImagesRequest with a valid base64 image that decodes to > 5 MB
  When answer_from_request validates the image size
  Then it returns Err(ApiError::bad_request("Image exceeds 5 MB limit."))

Scenario: Reject text_with_images with unsupported MIME type
  Given a SubmitAnswerTextWithImagesRequest with media_type "application/pdf"
  When answer_from_request validates the media type
  Then it returns Err(ApiError::bad_request("Unsupported image type. Allowed: image/png, image/jpeg, image/gif, image/webp, image/heic."))

Scenario: Validate TextWithImages answer for freeform question
  Given validate_answer_for_question at line 3706
  And a QuestionType::Freeform question
  When the answer is AnswerValue::TextWithImages
  Then validation returns Ok(())

Scenario: Validate TextWithImages answer for allow_freeform multiple-choice question
  Given a QuestionType::MultipleChoice question with allow_freeform: true
  When the answer is AnswerValue::TextWithImages
  Then validation returns Ok(())

Scenario: Reject TextWithImages answer for non-freeform question
  Given a QuestionType::YesNo question (no freeform allowed)
  When the answer is AnswerValue::TextWithImages
  Then validation returns Err(ApiError::bad_request("Images are only allowed for freeform questions."))
```

#### Steps

1. **Add base64 decoding helper** — implement fn decode_base64_image(data: &str) -> Result<Vec<u8>, String> that decodes base64 using base64 crate; returns error message on invalid base64.
   - **Scenario**: Reject text_with_images with invalid base64

2. **Test base64 decoding helper with valid input** — unit test with valid base64 string, assert returns Ok(Vec<u8>) matching decoded bytes.
   - **Scenario**: Map SubmitAnswerTextWithImagesRequest to Answer

3. **Test base64 decoding helper with invalid input** — unit test with malformed base64, assert returns Err with message.
   - **Scenario**: Reject text_with_images with invalid base64

4. **Add MIME type validation helper** — implement fn is_valid_image_mime(mime: &str) -> bool that checks mime against ["image/png", "image/jpeg", "image/gif", "image/webp", "image/heic"].
   - **Scenario**: Reject text_with_images with unsupported MIME type

5. **Test MIME type validation** — unit test with each allowed type returns true, "application/pdf" returns false.
   - **Scenario**: Reject text_with_images with unsupported MIME type

6. **Extend answer_from_request with TextWithImagesRequest arm** — add match arm SubmitAnswerRequest::TextWithImagesRequest(req); validate text non-empty, images non-empty; decode each base64 data, check size <= 5 MB, detect actual MIME type from decoded bytes using `infer` crate and reject if it doesn't match declared media_type, validate MIME is in allowed list; construct ImageAttachment vec; return Ok(Answer::text_with_images(text, images)). Additionally, configure Axum's `DefaultBodyLimit` middleware to accept payloads up to 50 MB for the answer submission endpoint to accommodate 4 × 5 MB images (~27 MB base64-encoded).
   - **Scenarios**: Map SubmitAnswerTextWithImagesRequest to Answer, Reject text_with_images with empty text, Reject text_with_images with empty images array, Reject text_with_images with invalid base64, Reject text_with_images with oversized decoded image, Reject text_with_images with unsupported MIME type

7. **Test answer_from_request happy path** — unit test with valid TextWithImagesRequest (non-empty text, valid base64 images under 5 MB, supported MIME), assert returns Ok(Answer) with AnswerValue::TextWithImages.
   - **Scenario**: Map SubmitAnswerTextWithImagesRequest to Answer

8. **Test answer_from_request rejects empty text** — unit test with text "", assert returns Err with bad_request message.
   - **Scenario**: Reject text_with_images with empty text

9. **Test answer_from_request rejects empty images** — unit test with images [], assert returns Err.
   - **Scenario**: Reject text_with_images with empty images array

10. **Test answer_from_request rejects invalid base64** — unit test with malformed base64 data, assert returns Err.
    - **Scenario**: Reject text_with_images with invalid base64

11. **Test answer_from_request rejects oversized image** — unit test with valid base64 encoding > 5 MB decoded, assert returns Err.
    - **Scenario**: Reject text_with_images with oversized decoded image

12. **Test answer_from_request rejects unsupported MIME** — unit test with media_type "application/pdf", assert returns Err.
    - **Scenario**: Reject text_with_images with unsupported MIME type

13. **Test answer_from_request rejects MIME type mismatch** — unit test with declared media_type "image/png" but actual JPEG bytes (detected via `infer`), assert returns Err with message about MIME spoofing or type mismatch.
    - **Scenario**: Reject text_with_images with unsupported MIME type (security variant)

14. **Test Axum accepts large payloads** — integration test that POSTs a valid text_with_images request with ~27 MB JSON body (4 images × 5 MB base64-encoded), assert server accepts it (HTTP 204) and does not return 413 Payload Too Large.
    - **Scenario**: Submit answer with images encodes to base64 and sends text_with_images request (large payload variant)

15. **Extend validate_answer_for_question with TextWithImages** — add match arm (QuestionType::Freeform | QuestionType::MultipleChoice if allow_freeform, AnswerValue::TextWithImages) => Ok(()); add fallback arm for TextWithImages on non-freeform => Err(bad_request("Images are only allowed for freeform questions.")).
    - **Scenarios**: Validate TextWithImages answer for freeform question, Validate TextWithImages answer for allow_freeform multiple-choice question, Reject TextWithImages answer for non-freeform question

16. **Test validate_answer_for_question accepts TextWithImages for Freeform** — unit test with QuestionType::Freeform and AnswerValue::TextWithImages, assert Ok(()).
    - **Scenario**: Validate TextWithImages answer for freeform question

17. **Test validate_answer_for_question accepts TextWithImages for allow_freeform MultipleChoice** — unit test with QuestionType::MultipleChoice, allow_freeform: true, assert Ok(()).
    - **Scenario**: Validate TextWithImages answer for allow_freeform multiple-choice question

18. **Test validate_answer_for_question rejects TextWithImages for YesNo** — unit test with QuestionType::YesNo, assert Err.
    - **Scenario**: Reject TextWithImages answer for non-freeform question

19. **Refactor** — extract validation constants (MAX_IMAGE_SIZE_BYTES = 5 * 1024 * 1024, ALLOWED_IMAGE_MIMES) to top-level or config module.

### Slice 4: Image persistence in run artifact directory

**Depends-on**: Slice 2 (fabro-interview types), Slice 3 (server validation)

**Files**:
- lib/crates/fabro-workflow/src/handler/human.rs

#### Scenarios

```gherkin
Scenario: Store images from TextWithImages answer in run artifact directory
  Given a human stage execution that receives an Answer with AnswerValue::TextWithImages { text: "test", images: [ImageAttachment { data: vec![...], media_type: "image/png" }] }
  And the run_dir is /tmp/run_xyz/artifacts
  And the stage_id is "review"
  When the handler processes the answer
  Then it writes human_review_image_0.png to /tmp/run_xyz/artifacts/
  And the file contains the exact bytes from ImageAttachment.data
  And the file is readable by subsequent stages

Scenario: Store multiple images with sequential naming
  Given an answer with 3 images (image/png, image/jpeg, image/gif)
  When the handler stores them
  Then files are created: human_review_image_0.png, human_review_image_1.jpg, human_review_image_2.gif
  And file extensions match media_type (png for image/png, jpg for image/jpeg, gif for image/gif)

Scenario: Add image paths to workflow context
  Given stored images at ["/tmp/run_xyz/artifacts/human_review_image_0.png", "/tmp/run_xyz/artifacts/human_review_image_1.jpg"]
  When the handler updates the outcome context
  Then outcome.context_updates includes key "fabro.human_answer_images.review" with JSON array of absolute file paths

Scenario: No context key added when answer is Text (no images)
  Given an Answer with AnswerValue::Text("plain text")
  When the handler processes the answer
  Then outcome.context_updates does not contain "fabro.human_answer_images.*"
  And no image files are written

Scenario: Context key is stage-specific
  Given a human stage with id "clarify"
  And an answer with images
  When the handler stores images
  Then the context key is "fabro.human_answer_images.clarify"
  And it does not collide with images from other stages
```

#### Steps

1. **Add image file extension mapping helper** — implement fn extension_for_mime(mime: &str) -> &'static str that maps image/png -> "png", image/jpeg -> "jpg", image/gif -> "gif", image/webp -> "webp", image/heic -> "heic", fallback -> "bin".
   - **Scenario**: Store multiple images with sequential naming

2. **Test extension mapping** — unit test each MIME type maps to expected extension.
   - **Scenario**: Store multiple images with sequential naming

3. **Add image storage helper** — implement async fn store_answer_images(images: &[ImageAttachment], stage_id: &str, run_dir: &Path) -> Result<Vec<PathBuf>, Error> that writes each image to {run_dir}/human_{stage_id}_image_{index}.{ext}; returns Vec of absolute paths.
   - **Scenarios**: Store images from TextWithImages answer in run artifact directory, Store multiple images with sequential naming

4. **Test image storage with single image** — integration test that calls store_answer_images with one ImageAttachment, asserts file is created at expected path, reads file back, asserts content matches ImageAttachment.data.
   - **Scenario**: Store images from TextWithImages answer in run artifact directory

5. **Test image storage with multiple images** — integration test with 3 images (PNG, JPEG, GIF), asserts 3 files created with correct extensions and indices, reads each back, asserts content.
   - **Scenario**: Store multiple images with sequential naming

6. **Extend human handler execute method to store images** — in handler/human.rs around line 360 (after answer is received), add match on answer.value; if TextWithImages { images, .. }, call store_answer_images(images, node.id, run_dir); store returned paths in a local variable.
   - **Scenarios**: Store images from TextWithImages answer in run artifact directory, Store multiple images with sequential naming

7. **Add context key for image paths** — if images were stored, insert "fabro.human_answer_images.{stage_id}": serde_json::to_value(image_paths) into outcome.context_updates before returning outcome.
   - **Scenario**: Add image paths to workflow context

8. **Test context key is added for TextWithImages answer** — integration test that simulates human stage execution with TextWithImages answer, asserts outcome.context_updates contains key matching "fabro.human_answer_images.{stage_id}" with JSON array of paths.
   - **Scenario**: Add image paths to workflow context

9. **Test no context key for Text answer** — integration test with AnswerValue::Text, asserts no "fabro.human_answer_images.*" key in outcome.
   - **Scenario**: No context key added when answer is Text (no images)

10. **Test context key is stage-specific** — integration test with two human stages (different stage_ids) both providing images, assert two distinct context keys with different stage_id suffixes.
    - **Scenario**: Context key is stage-specific

11. **Refactor** — extract image storage logic into a separate function or module if it exceeds ~30 lines.

### Slice 5: Agent stage automatic image propagation

**Depends-on**: Slice 4 (image persistence)

**Files**:
- lib/crates/fabro-workflow/src/handler/agent.rs
- lib/crates/fabro-workflow/src/context.rs (add context key constant)

#### Scenarios

```gherkin
Scenario: Agent stage prepends images from prior human stage
  Given a workflow with human stage "review" followed by agent stage "implement"
  And the human stage provided an answer with images stored at ["/tmp/run_xyz/artifacts/human_review_image_0.png"]
  And workflow context contains "fabro.human_answer_images.review": ["/tmp/run_xyz/artifacts/human_review_image_0.png"]
  When the agent stage "implement" executes
  And it constructs the initial user message
  Then the message content starts with ContentPart::Image(ImageData { url: Some("/tmp/run_xyz/artifacts/human_review_image_0.png"), data: None, media_type: None, detail: None })
  And the human's text answer is included as ContentPart::Text after the images

Scenario: Agent stage prepends multiple images in order
  Given the prior human stage provided 3 images [image_0.png, image_1.jpg, image_2.gif]
  When the agent stage constructs the initial message
  Then the message content is [Image(image_0), Image(image_1), Image(image_2), Text(answer_text)]

Scenario: Agent stage proceeds normally when no images in context
  Given a human stage that provided a Text answer (no images)
  And workflow context does not contain "fabro.human_answer_images.*"
  When the agent stage constructs the initial message
  Then the message content is [Text(answer_text)]
  And no image parts are included

Scenario: Images are resolved by fabro-llm attachment resolution
  Given an agent message with ContentPart::Image(ImageData { url: Some("/tmp/run_xyz/artifacts/human_review_image_0.png"), ... })
  When fabro-llm encodes the request for the LLM API
  And it calls attachments::resolve with the request
  Then the image file is read from disk
  And the ContentPart::Image is rewritten with data: Some(bytes), media_type: Some(detected_mime), url: None
  And the inline image is sent to the LLM provider

Scenario: Non-agent stages can access images via context variable
  Given a workflow with human stage "review" followed by command stage "summarize"
  And the context contains "fabro.human_answer_images.review": ["/path/to/image.png"]
  When the command stage references {{fabro.human_answer_images.review}} in its command template
  Then the template is replaced with the JSON array of paths
  And the command can process the image files
```

#### Steps

1. **Add context key constant** — add pub const HUMAN_ANSWER_IMAGES_PREFIX: &str = "fabro.human_answer_images." to lib/crates/fabro-workflow/src/context.rs keys module around line 50.
   - **Scenario**: Agent stage prepends images from prior human stage

2. **Add helper to extract human images from context** — implement fn get_human_answer_images(context: &Context, stage_id: &str) -> Option<Vec<String>> that reads "fabro.human_answer_images.{stage_id}" from context, deserializes to Vec<String> (file paths), returns Some(paths) or None.
   - **Scenario**: Agent stage prepends images from prior human stage

3. **Test get_human_answer_images with images in context** — unit test that creates context with "fabro.human_answer_images.review": ["/tmp/image.png"], calls helper, asserts returns Some(vec!["/tmp/image.png"]).
   - **Scenario**: Agent stage prepends images from prior human stage

4. **Test get_human_answer_images with no images** — unit test with context lacking the key, asserts returns None.
   - **Scenario**: Agent stage proceeds normally when no images in context

5. **Identify prior human stage** — implement fn prior_human_stage_id(context: &Context) -> Option<String> that looks for keys::LAST_STAGE or equivalent in context to identify the immediately prior stage; returns Some(stage_id) if it was a human stage, else None. (Note: This may require checking stage type in metadata or relying on convention that human stages always populate the context key.)
   - **Scenario**: Agent stage prepends images from prior human stage

6. **Test prior_human_stage_id** — unit test with context containing LAST_STAGE = "review" and "fabro.human_answer_images.review" present, asserts returns Some("review").
   - **Scenario**: Agent stage prepends images from prior human stage

7. **Extend agent handler to prepend images to initial message** — in lib/crates/fabro-workflow/src/handler/agent.rs, when constructing the initial user message (around where prompt text is converted to Message), call prior_human_stage_id(context); if Some(stage_id), call get_human_answer_images(context, stage_id); if Some(image_paths), prepend ContentPart::Image(ImageData { url: Some(path), data: None, media_type: None, detail: None }) for each path before the text content.
   - **Scenarios**: Agent stage prepends images from prior human stage, Agent stage prepends multiple images in order

8. **Test agent message construction with images** — integration test that simulates agent stage execution with context containing "fabro.human_answer_images.review": ["/tmp/image.png"]; asserts the constructed Message.content starts with ContentPart::Image.
   - **Scenario**: Agent stage prepends images from prior human stage

9. **Test agent message construction with multiple images** — integration test with 3 images in context, asserts Message.content is [Image, Image, Image, Text].
   - **Scenario**: Agent stage prepends multiple images in order

10. **Test agent message construction without images** — integration test with no human_answer_images key, asserts Message.content is [Text] only.
    - **Scenario**: Agent stage proceeds normally when no images in context

11. **Verify fabro-llm attachment resolution** — review lib/crates/fabro-llm/src/attachments.rs to confirm ContentPart::Image with url: Some(file_path) is already resolved to inline data by attachments::resolve; write integration test if needed to prove file-to-inline resolution works.
    - **Scenario**: Images are resolved by fabro-llm attachment resolution

12. **Test non-agent stage can access context variable** — integration test with a command stage that uses {{fabro.human_answer_images.review}} in its command template, asserts template is replaced with JSON array.
    - **Scenario**: Non-agent stages can access images via context variable

13. **Refactor** — if prior_human_stage_id logic is complex or requires graph traversal, extract into a dedicated helper function or move to a context utility module.

### Slice 6: Frontend UI for image attachment (file picker)

**Depends-on**: Slice 1 (TypeScript client types)

**Files**:
- apps/fabro-web/app/components/interview-dock.tsx

#### Scenarios

```gherkin
Scenario: Display image attachment button in freeform response field
  Given the interview dock is rendering a Freeform question
  When the FreeformBody component is displayed
  Then an image attachment button (icon or text) is visible below or alongside the textarea
  And clicking the button triggers a hidden file input

Scenario: File picker opens on button click and accepts only images
  Given the user clicks the image attachment button
  When the browser file picker opens
  Then it is constrained to accept="image/png, image/jpeg, image/gif, image/webp, image/heic"
  And the user can select one or more image files

Scenario: Selected images appear as thumbnail previews
  Given the user selects 2 images (image1.png, image2.jpg) via the file picker
  When the files are loaded
  Then 2 thumbnail previews are displayed
  And each thumbnail shows the image content
  And each thumbnail has a remove (X) button

Scenario: User can remove an attached image
  Given 2 images are attached
  When the user clicks the remove button on the first image
  Then the first image is removed from the attachment list
  And only 1 image remains
  And the UI updates to show only 1 thumbnail

Scenario: Prevent attaching more than 4 images
  Given 4 images are already attached
  When the user attempts to select a 5th image
  Then an inline error message is displayed: "Maximum 4 images allowed."
  And the 5th image is not added to the attachment list

Scenario: Prevent attaching image larger than 5 MB
  Given the user selects an image file of 6 MB
  When the file is validated
  Then an inline error message is displayed: "Image exceeds 5 MB limit."
  And the image is not added to the attachment list

Scenario: Prevent attaching non-image file
  Given the user selects a PDF file (bypassing accept constraint via OS dialog or drag-drop)
  When the file is validated
  Then an inline error message is displayed: "Only image files are allowed."
  And the file is not added to the attachment list

Scenario: Submit button remains disabled if only images (no text)
  Given the user attaches 2 images
  And the textarea is empty
  Then the submit button is disabled
  And a hint message may indicate "Text is required when submitting images."

Scenario: Enable submit button with both text and images
  Given the user types "Please review" in the textarea
  And attaches 1 image
  Then the submit button is enabled
```

#### Steps

1. **Add state for attached images** — in FreeformBody component (line 406), add const [images, setImages] = useState<File[]>([]) to track selected image files.
   - **Scenario**: Selected images appear as thumbnail previews

2. **Add hidden file input element** — insert <input type="file" accept="image/png,image/jpeg,image/gif,image/webp,image/heic" multiple ref={fileInputRef} style={{ display: 'none' }} onChange={handleFileSelect} /> inside the form.
   - **Scenario**: File picker opens on button click and accepts only images

3. **Add image attachment button** — below the textarea (around line 450), add a button or icon (e.g., paperclip or image icon) with onClick={() => fileInputRef.current?.click()}.
   - **Scenario**: Display image attachment button in freeform response field

4. **Test button renders** — write component test (interview-dock.test.tsx) that renders FreeformBody, asserts image attachment button is present.
   - **Scenario**: Display image attachment button in freeform response field

5. **Test button opens file picker** — component test that simulates button click, asserts file input click is triggered (may require mocking fileInputRef.current.click).
   - **Scenario**: File picker opens on button click and accepts only images

6. **Implement handleFileSelect** — define handleFileSelect(event) that reads event.target.files, validates each file (image MIME type, size <= 5 MB), filters valid files, checks total count + existing images <= 4, updates setImages([...images, ...validFiles]), shows inline error for invalid files.
   - **Scenarios**: Selected images appear as thumbnail previews, Prevent attaching more than 4 images, Prevent attaching image larger than 5 MB, Prevent attaching non-image file

7. **Test handleFileSelect with valid images** — component test with File objects (1 MB, image/png), asserts images state is updated.
   - **Scenario**: Selected images appear as thumbnail previews

8. **Test handleFileSelect rejects oversized image** — component test with File(6 MB), asserts error message is shown, images state unchanged.
   - **Scenario**: Prevent attaching image larger than 5 MB

9. **Test handleFileSelect rejects non-image** — component test with File("application/pdf"), asserts error message, images state unchanged.
   - **Scenario**: Prevent attaching non-image file

10. **Test handleFileSelect enforces 4-image limit** — component test with 4 existing images, select 1 more, asserts error message, images state remains 4.
    - **Scenario**: Prevent attaching more than 4 images

11. **Add thumbnail preview component** — implement ImageThumbnail({ file, onRemove }) that reads file with FileReader, displays <img src={dataUrl} />, shows remove button calling onRemove(file).
    - **Scenario**: Selected images appear as thumbnail previews

12. **Render thumbnails** — below the textarea, map images.map(file => <ImageThumbnail key={file.name} file={file} onRemove={handleRemoveImage} />).
    - **Scenario**: Selected images appear as thumbnail previews

13. **Implement handleRemoveImage** — define handleRemoveImage(file) that calls setImages(images.filter(img => img !== file)).
    - **Scenario**: User can remove an attached image

14. **Test thumbnail removal** — component test that adds 2 images, simulates remove button click, asserts images state reduces to 1.
    - **Scenario**: User can remove an attached image

15. **Update submit button disabled logic** — change disabled condition to submitting || (value.trim().length === 0 && images.length === 0) so button is disabled if both text and images are empty; OR require text when images are present: disabled = submitting || (images.length > 0 && value.trim().length === 0) || (images.length === 0 && value.trim().length === 0).
    - **Scenarios**: Submit button remains disabled if only images (no text), Enable submit button with both text and images

16. **Test submit button enabled with text and images** — component test with textarea "hello" and 1 image, asserts button is enabled.
    - **Scenario**: Enable submit button with both text and images

17. **Test submit button disabled with only images** — component test with no text and 1 image, asserts button is disabled.
    - **Scenario**: Submit button remains disabled if only images (no text)

18. **Refactor** — extract image validation logic (MIME check, size check, count check) into a separate validateImageFile(file: File, existingCount: number) -> { valid: boolean, error?: string } helper.

### Slice 7: Frontend clipboard paste for images

**Depends-on**: Slice 6 (file picker UI)

**Files**:
- apps/fabro-web/app/components/interview-dock.tsx

#### Scenarios

```gherkin
Scenario: Paste image from clipboard attaches it automatically
  Given the textarea is focused
  When the user pastes an image from the clipboard (e.g., screenshot)
  Then the image is automatically added to the attachment list
  And a thumbnail preview appears
  And no file picker is opened

Scenario: Paste text does not trigger image attachment
  Given the textarea is focused
  When the user pastes plain text "hello"
  Then the text is inserted into the textarea
  And no image is attached

Scenario: Paste image respects 4-image limit
  Given 4 images are already attached
  When the user pastes a 5th image
  Then an error message is displayed: "Maximum 4 images allowed."
  And the pasted image is not attached

Scenario: Paste oversized image shows error
  Given the user pastes an image from clipboard that is > 5 MB
  Then an error message is displayed: "Image exceeds 5 MB limit."
  And the image is not attached

Scenario: Multiple clipboard items with images attaches all valid ones
  Given the clipboard contains 2 images (both under 5 MB)
  When the user pastes
  Then both images are attached
  And 2 thumbnails appear
```

#### Steps

1. **Add onPaste handler to textarea** — in the textarea element (line 439), add onPaste={handlePaste}.
   - **Scenario**: Paste image from clipboard attaches it automatically

2. **Implement handlePaste** — define handlePaste(event) that reads event.clipboardData.items, filters for image/* types, converts each to File via item.getAsFile(), validates each file (same logic as handleFileSelect), adds valid files to images state, shows errors for invalid files.
   - **Scenarios**: Paste image from clipboard attaches it automatically, Paste text does not trigger image attachment, Paste image respects 4-image limit, Paste oversized image shows error, Multiple clipboard items with images attaches all valid ones

3. **Test handlePaste with image clipboard item** — component test that simulates paste event with clipboardData containing image/png item, asserts images state is updated.
   - **Scenario**: Paste image from clipboard attaches it automatically

4. **Test handlePaste with text clipboard item** — component test with plain text clipboard data, asserts images state unchanged, text is inserted in textarea (default behavior).
   - **Scenario**: Paste text does not trigger image attachment

5. **Test handlePaste enforces 4-image limit** — component test with 4 existing images, paste 1 more, asserts error message, images state remains 4.
   - **Scenario**: Paste image respects 4-image limit

6. **Test handlePaste rejects oversized image** — component test with clipboard image > 5 MB, asserts error, images unchanged.
   - **Scenario**: Paste oversized image shows error

7. **Test handlePaste with multiple images** — component test with clipboardData containing 2 image items, asserts both added to images state.
   - **Scenario**: Multiple clipboard items with images attaches all valid ones

8. **Refactor** — reuse validateImageFile helper from Slice 6 to avoid duplicating validation logic in handlePaste.

### Slice 8: Frontend image encoding and submission

**Depends-on**: Slice 6 (file picker UI), Slice 7 (clipboard paste)

**Files**:
- apps/fabro-web/app/components/interview-dock.tsx

#### Scenarios

```gherkin
Scenario: Submit answer with images encodes to base64 and sends text_with_images request
  Given the user typed "Please review" in the textarea
  And attached 2 images (image1.png, image2.jpg)
  When the user submits the form
  Then each image File is read and encoded to base64
  And the API request is POST /api/v1/runs/{id}/questions/{qid}/answer with JSON body:
    {
      "kind": "text_with_images",
      "text": "Please review",
      "images": [
        { "data": "<base64 of image1.png>", "media_type": "image/png" },
        { "data": "<base64 of image2.jpg>", "media_type": "image/jpeg" }
      ]
    }
  And the server returns HTTP 204

Scenario: Submit answer without images sends text request (backward compatible)
  Given the user typed "No images" and attached no images
  When the user submits the form
  Then the API request is POST with JSON body: { "kind": "text", "text": "No images" }
  And the server returns HTTP 204

Scenario: Display server validation error on oversized image
  Given the user submits an answer with images
  And the server returns HTTP 400 with error "Image exceeds 5 MB limit."
  Then the error message is displayed inline in the UI
  And the form is not cleared

Scenario: Clear form and images after successful submission
  Given the user submits an answer with text and images
  And the server returns HTTP 204
  Then the textarea is cleared
  And the images state is reset to []
  And all thumbnails are removed
```

#### Steps

1. **Add base64 encoding helper** — implement async fn encodeImageToBase64(file: File): Promise<{ data: string, media_type: string }> that reads file with FileReader, returns base64-encoded data (without data URI prefix) and media_type.
   - **Scenario**: Submit answer with images encodes to base64 and sends text_with_images request

2. **Test base64 encoding helper** — unit test that creates a File, encodes it, asserts data is valid base64 string, media_type matches file.type.
   - **Scenario**: Submit answer with images encodes to base64 and sends text_with_images request

3. **Update handleSubmit to encode images** — in handleSubmit (line 408), before calling onSubmit, check if images.length > 0; if yes, call encodeImageToBase64 for each image, await all promises, construct SubmitInterviewAnswer with kind: "text_with_images", text: trimmed, images: encodedImages; else use kind: "text" as before.
   - **Scenarios**: Submit answer with images encodes to base64 and sends text_with_images request, Submit answer without images sends text request (backward compatible)

4. **Test handleSubmit with images** — component test that sets value "hello" and images [File(...)], submits form, asserts onSubmit is called with { kind: "text_with_images", text: "hello", images: [...] }.
   - **Scenario**: Submit answer with images encodes to base64 and sends text_with_images request

5. **Test handleSubmit without images** — component test with value "hello" and images [], asserts onSubmit is called with { kind: "text", text: "hello" }.
   - **Scenario**: Submit answer without images sends text request (backward compatible)

6. **Add error state for submission errors** — add const [submitError, setSubmitError] = useState<string | null>(null); when onSubmit rejects, catch error, extract message, setSubmitError(message); display error below the form if submitError is not null.
   - **Scenario**: Display server validation error on oversized image

7. **Test submission error display** — component test that mocks onSubmit to reject with error, submits form, asserts error message is displayed.
   - **Scenario**: Display server validation error on oversized image

8. **Clear form and images on success** — after successful onSubmit (no rejection), call setValue(""), setImages([]), setSubmitError(null).
   - **Scenario**: Clear form and images after successful submission

9. **Test form is cleared after success** — component test that submits successfully, asserts value is "", images is [].
   - **Scenario**: Clear form and images after successful submission

10. **Refactor** — extract image encoding loop into a separate function encodeAllImages(files: File[]): Promise<Array<{ data: string, media_type: string }>> for clarity.

### Slice 9: End-to-end integration test

**Depends-on**: Slice 3 (server validation), Slice 4 (image persistence), Slice 5 (agent propagation), Slice 8 (frontend submission)

**Files**:
- lib/crates/fabro-server/src/server/tests.rs (or new integration test file)
- lib/crates/fabro-workflow/tests/it/integration.rs (or new E2E test file)

#### Scenarios

```gherkin
Scenario: Full workflow with image submission and agent consumption
  Given a workflow with human stage "review" -> agent stage "implement"
  And a test server is running
  When the user submits a text_with_images answer to the "review" question with text "Fix the bug" and 1 image (base64-encoded screenshot)
  Then the server accepts the answer (HTTP 204)
  And the image is stored in the run's artifact directory as human_review_image_0.png
  And the workflow context contains "fabro.human_answer_images.review": ["/path/to/human_review_image_0.png"]
  And when the "implement" agent stage executes
  Then the agent's initial message includes ContentPart::Image with the image file path
  And fabro-llm resolves the file path to inline base64 data
  And the LLM API request (captured in test) includes the image
  And the agent's response demonstrates it received the image (e.g., "I see the screenshot showing...")

Scenario: Text-only workflow remains unaffected
  Given a workflow with human stage "confirm" -> agent stage "proceed"
  When the user submits a text answer (kind: "text") with text "Yes, proceed"
  Then the workflow executes normally
  And no images are stored
  And the agent stage receives only text content
  And the workflow completes successfully
```

#### Steps

1. **Create test workflow with human -> agent stages** — define a minimal workflow YAML or Graphviz with human gate "review" -> agent "implement"; use test_support helpers to construct the workflow.
   - **Scenario**: Full workflow with image submission and agent consumption

2. **Test server accepts text_with_images answer** — integration test that constructs SubmitAnswerTextWithImagesRequest with valid base64 image, POSTs to /api/v1/runs/{id}/questions/{qid}/answer, asserts HTTP 204.
   - **Scenario**: Full workflow with image submission and agent consumption

3. **Test image is persisted to artifact directory** — after answer submission, read run's artifact directory, assert human_review_image_0.png exists, read file, assert content matches decoded base64 from request.
   - **Scenario**: Full workflow with image submission and agent consumption

4. **Test workflow context contains image paths** — after human stage completes, read workflow context (via run state or context snapshot), assert "fabro.human_answer_images.review" is present with expected paths.
   - **Scenario**: Full workflow with image submission and agent consumption

5. **Test agent stage receives image in initial message** — mock or capture the agent's LLM request (via test backend or spy), assert the initial Message.content includes ContentPart::Image with url pointing to the stored file.
   - **Scenario**: Full workflow with image submission and agent consumption

6. **Test fabro-llm resolves image to inline data** — mock LLM provider, capture encoded request, assert image is inline base64 (data field populated, url is None).
   - **Scenario**: Full workflow with image submission and agent consumption

7. **Test text-only answer workflow** — integration test with kind: "text" answer, run workflow, assert no image files created, agent receives only text content, workflow completes.
   - **Scenario**: Text-only workflow remains unaffected

8. **Refactor** — extract common test setup (server, workflow, run creation) into test helper functions.

## Parallelization

### Wave 0 (no dependencies)
- Slice 1: OpenAPI schema and type generation
- Slice 2: fabro-interview domain types for image attachments

### Wave 1 (depends on Wave 0)
- Slice 3: Server-side answer request mapping and validation (depends on Slice 1, Slice 2)
- Slice 6: Frontend UI for image attachment (file picker) (depends on Slice 1)

### Wave 2 (depends on Wave 1)
- Slice 4: Image persistence in run artifact directory (depends on Slice 2, Slice 3)
- Slice 7: Frontend clipboard paste for images (depends on Slice 6)

### Wave 3 (depends on Wave 2)
- Slice 5: Agent stage automatic image propagation (depends on Slice 4)
- Slice 8: Frontend image encoding and submission (depends on Slice 6, Slice 7)

### Wave 4 (depends on Wave 3)
- Slice 9: End-to-end integration test (depends on Slice 3, Slice 4, Slice 5, Slice 8)

**File collision check**:
- Wave 1: Slice 3 (server.rs) and Slice 6 (interview-dock.tsx) — no overlap ✓
- Wave 2: Slice 4 (human.rs) and Slice 7 (interview-dock.tsx) — no overlap ✓
- Wave 3: Slice 5 (agent.rs, context.rs) and Slice 8 (interview-dock.tsx) — no overlap ✓

## Skipped (low value)

None. All ambiguity log findings classified as "low value" were already excluded from the spec's ambiguity log (no such findings present). All remaining scenarios directly trace to acceptance criteria and involve branching logic or observable behavior.

## Risks & Open Questions

1. **Image file lifecycle**: Images are stored in the run's artifact directory and cleaned up when the run is pruned. If a run is archived before pruning, images persist in the archive. This matches the spec's intent and the existing artifact cleanup contract. No additional action needed unless long-term storage requirements emerge.

2. **Prior human stage detection**: Slice 5 assumes we can reliably identify the immediately prior stage from workflow context (e.g., via keys::LAST_STAGE). If the context does not reliably track the prior stage or its type, we may need to traverse the graph or rely on event log. Mitigation: Review context population in workflow engine; add LAST_STAGE_TYPE if needed, or use event log to find the most recent InterviewCompleted event and extract the stage_id.

3. **Base64 payload size**: A 5 MB image encodes to ~6.7 MB base64. With 4 images, the JSON payload could exceed 26 MB. Axum's default payload size limit is 2 MB for JSON. Mitigation: Slice 3 should configure Axum's `DefaultBodyLimit` layer to accept larger payloads (e.g., 50 MB) for the answer submission endpoint. This is a known requirement from the spec (line 92) and should be added as a TDD step in Slice 3.

4. **Frontend error message UX**: Slice 6/7/8 define inline error messages for image validation failures. The spec does not prescribe the exact UI placement or styling. Recommendation: Display errors in a consistent location (e.g., below the attachment area or near the submit button) with a dismissible toast or alert component. This should be validated in component tests but is a UX detail outside the spec's scope.

5. **Concurrent image attachment and text editing**: Users may attach images, then edit text, then submit. The spec does not address whether images should be re-validated on every text change. Recommendation: Validate images only on selection/paste and on submission, not on text change. This minimizes unnecessary validation overhead and matches standard file upload UX patterns.

6. **Agent stage behavior when images are missing from disk**: If an image file is deleted or unreadable when fabro-llm's attachments::resolve runs, the current contract (per attachments.rs:6) is to drop the ContentPart silently. This means the agent would receive fewer images than expected, with no error. Recommendation: Accept this behavior for the initial implementation; log a warning when an image is dropped; consider adding a future enhancement to fail the agent stage if critical attachments are missing. This is not a blocker for the feature.

7. **Image MIME type detection**: The spec accepts user-provided media_type from the frontend. A malicious or buggy client could send media_type: "image/png" with a JPEG file. Recommendation: Server-side validation in Slice 3 should detect MIME type from the decoded bytes (using a library like `infer` or `file-format`) and reject if it does not match the declared media_type. Add this as a TDD step in Slice 3, Step 6.

8. **Clipboard paste browser compatibility**: ClipboardEvent.clipboardData.items support varies across browsers. Safari and older Firefox versions may not expose image items. Mitigation: Document browser compatibility requirements (modern Chrome, Firefox, Safari); degrade gracefully by showing an error message if clipboardData.items is unavailable or empty; recommend users use file picker as fallback.

## Plan Review Summary

**Review date**: 2026-07-21
**Reviewers**: review_acceptance, review_design, review_ux, review_strategic, review_parallel

All five reviewers approved the plan with success verdicts. No blockers were identified.

### Reviewer verdicts

**review_acceptance** (acceptance criteria coverage): ✅ Approved
- All acceptance criteria are covered by scenarios across slices
- Each AC requirement traces to specific test scenarios with measurable outcomes
- Error paths and edge cases are adequately covered

**review_design** (technical design and architecture): ✅ Approved
- Clean separation between API schema (Slice 1), domain types (Slice 2), server validation (Slice 3), persistence (Slice 4), and agent integration (Slice 5)
- Frontend slices (6-8) follow React component patterns and maintain separation of concerns
- Image attachment flow follows existing fabro-interview and fabro-workflow contracts

**review_ux** (user experience): ✅ Approved
- Slice 6/7 provide dual input methods (file picker + clipboard paste) matching Claude Code UX
- Inline validation with clear error messages before submission reduces friction
- Progressive enhancement: file picker works everywhere, clipboard paste degrades gracefully
- Submit button state management ensures users cannot submit invalid combinations

**review_strategic** (risks and dependencies): ✅ Approved
- Risks section adequately identifies and mitigates potential issues (base64 payload size, MIME detection, prior stage detection, browser compatibility)
- Dependency chain is logical: types → validation → persistence → propagation → UI → E2E
- No circular dependencies or hidden coupling

**review_parallel** (parallelization plan): ✅ Approved
- Wave structure correctly reflects dependencies
- No file collisions within waves (verified for server.rs, interview-dock.tsx, human.rs, agent.rs, context.rs)
- Wave 0 has no dependencies, enabling immediate parallel work on API types and domain types

### Changes applied

**Human change request**: The user requested clipboard paste support for images, stating "I take a screenshot and paste it to CC" as the primary workflow.

**Resolution**: Slice 7 already addresses this requirement with full clipboard paste support, including:
- `onPaste` handler on the textarea (Step 1)
- Automatic image extraction from clipboard data (Step 2)
- Same validation as file picker (size, type, count limits)
- Multiple image paste support

No plan changes were required as clipboard paste was already a first-class feature in the plan.

### Observations and warnings kept

The following warnings from risk analysis were documented but not acted upon (acceptable as-is):

1. **Image file lifecycle** (Risk #1): Images persist in artifact directory and are cleaned up with run pruning. Long-term storage requirements are deferred to future work.

2. **Agent stage behavior when images are missing** (Risk #6): If an image file is deleted before fabro-llm resolution, it will be silently dropped. This is acceptable for the initial implementation; logging will provide visibility, and future enhancements can add stricter failure modes.

3. **Clipboard paste browser compatibility** (Risk #8): ClipboardEvent support varies across browsers. The plan includes graceful degradation and file picker fallback, which is sufficient for the feature scope.

### Additions to address reviewer feedback

**Base64 payload size limit** (identified in Risk #3): Added explicit requirement to Slice 3, Step 6 to configure Axum's `DefaultBodyLimit` to accept larger payloads (e.g., 50 MB) for the answer submission endpoint to accommodate 4 × 5 MB images encoded as base64 (~27 MB total).

**MIME type detection** (identified in Risk #7): Added explicit requirement to Slice 3, Step 6 to perform server-side MIME type detection from decoded bytes using a library like `infer` or `file-format`, and reject images where the detected type does not match the declared `media_type`. This prevents client-provided MIME spoofing.

All blockers resolved. Plan is ready for implementation.

## Build Progress

### Slice 1: OpenAPI schema and type generation
- [ ] Add SubmitAnswerTextWithImagesRequest schema to OpenAPI spec
- [ ] Add text_with_images to discriminator mapping
- [ ] Test OpenAPI spec validity
- [ ] Generate Rust types
- [ ] Generate TypeScript client
- [ ] Refactor

### Slice 2: fabro-interview domain types for image attachments
- [ ] Define ImageAttachment struct
- [ ] Test ImageAttachment serialization
- [ ] Add TextWithImages variant to AnswerValue
- [ ] Test TextWithImages variant in pattern matching
- [ ] Implement Answer::text_with_images constructor
- [ ] Test text_with_images constructor
- [ ] Test TextWithImages serialization round-trip
- [ ] Refactor

### Slice 3: Server-side answer request mapping and validation
- [ ] Add base64 decoding helper
- [ ] Test base64 decoding helper with valid input
- [ ] Test base64 decoding helper with invalid input
- [ ] Add MIME type validation helper
- [ ] Test MIME type validation
- [ ] Extend answer_from_request with TextWithImagesRequest arm
- [ ] Test answer_from_request happy path
- [ ] Test answer_from_request rejects empty text
- [ ] Test answer_from_request rejects empty images
- [ ] Test answer_from_request rejects invalid base64
- [ ] Test answer_from_request rejects oversized image
- [ ] Test answer_from_request rejects unsupported MIME
- [ ] Test answer_from_request rejects MIME type mismatch
- [ ] Test Axum accepts large payloads
- [ ] Extend validate_answer_for_question with TextWithImages
- [ ] Test validate_answer_for_question accepts TextWithImages for Freeform
- [ ] Test validate_answer_for_question accepts TextWithImages for allow_freeform MultipleChoice
- [ ] Test validate_answer_for_question rejects TextWithImages for YesNo
- [ ] Refactor

### Slice 4: Image persistence in run artifact directory
- [ ] Add image file extension mapping helper
- [ ] Test extension mapping
- [ ] Add image storage helper
- [ ] Test image storage with single image
- [ ] Test image storage with multiple images
- [ ] Extend human handler execute method to store images
- [ ] Add context key for image paths
- [ ] Test context key is added for TextWithImages answer
- [ ] Test no context key for Text answer
- [ ] Test context key is stage-specific
- [ ] Refactor

### Slice 5: Agent stage automatic image propagation
- [ ] Add context key constant
- [ ] Add helper to extract human images from context
- [ ] Test get_human_answer_images with images in context
- [ ] Test get_human_answer_images with no images
- [ ] Identify prior human stage
- [ ] Test prior_human_stage_id
- [ ] Extend agent handler to prepend images to initial message
- [ ] Test agent message construction with images
- [ ] Test agent message construction with multiple images
- [ ] Test agent message construction without images
- [ ] Verify fabro-llm attachment resolution
- [ ] Test non-agent stage can access context variable
- [ ] Refactor

### Slice 6: Frontend UI for image attachment (file picker)
- [ ] Add state for attached images
- [ ] Add hidden file input element
- [ ] Add image attachment button
- [ ] Test button renders
- [ ] Test button opens file picker
- [ ] Implement handleFileSelect
- [ ] Test handleFileSelect with valid images
- [ ] Test handleFileSelect rejects oversized image
- [ ] Test handleFileSelect rejects non-image
- [ ] Test handleFileSelect enforces 4-image limit
- [ ] Add thumbnail preview component
- [ ] Render thumbnails
- [ ] Implement handleRemoveImage
- [ ] Test thumbnail removal
- [ ] Update submit button disabled logic
- [ ] Test submit button enabled with text and images
- [ ] Test submit button disabled with only images
- [ ] Refactor

### Slice 7: Frontend clipboard paste for images
- [ ] Add onPaste handler to textarea
- [ ] Implement handlePaste
- [ ] Test handlePaste with image clipboard item
- [ ] Test handlePaste with text clipboard item
- [ ] Test handlePaste enforces 4-image limit
- [ ] Test handlePaste rejects oversized image
- [ ] Test handlePaste with multiple images
- [ ] Refactor

### Slice 8: Frontend image encoding and submission
- [ ] Add base64 encoding helper
- [ ] Test base64 encoding helper
- [ ] Update handleSubmit to encode images
- [ ] Test handleSubmit with images
- [ ] Test handleSubmit without images
- [ ] Add error state for submission errors
- [ ] Test submission error display
- [ ] Clear form and images on success
- [ ] Test form is cleared after success
- [ ] Refactor

### Slice 9: End-to-end integration test
- [ ] Create test workflow with human -> agent stages
- [ ] Test server accepts text_with_images answer
- [ ] Test image is persisted to artifact directory
- [ ] Test workflow context contains image paths
- [ ] Test agent stage receives image in initial message
- [ ] Test fabro-llm resolves image to inline data
- [ ] Test text-only answer workflow
- [ ] Refactor
