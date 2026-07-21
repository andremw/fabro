I'll execute the next slice of the plan. Let me first read the plan to see what's been completed and what's next.I can see that Slice 4 is the next unchecked slice. Let me create a todo list and begin implementing it.Now let me start implementing Slice 4. First, I'll read the agent.rs file to locate CodergenRunRequest:Good, I can see CodergenRunRequest at line 39. Let me mark the first task as in_progress and implement it:Now let me change the CodergenRunRequest structure:Now let me test that this compiles:The build is in progress. Let me wait for it to complete and see the errors:Perfect, I can see the compilation errors. As expected from the plan, step 2 says to expect errors at backend call sites which will be fixed in later steps. The errors show:

1. Two call sites constructing CodergenRunRequest with `prompt` field (fan_in.rs:275, agent.rs:292)
2. Three call sites in api.rs and acp.rs trying to access `request.prompt`
3. Three call sites in api.rs calling `process_input_with_runtime`

Let me proceed to step 3 - updating AgentApiBackend::run. First, let me read the api.rs file:Let me see the line where it calls process_input_with_runtime:Now I need to update line 1140 to use `initial_content` and line 1272 to call `process_message` instead:Now let me check for the other occurrences in api.rs - the error log showed line 1406 and 1474:Now let me check line 1474:This repair_message is a string, so I need to wrap it in ContentPart::Text:Now let me test if AgentApiBackend compiles:Good, the api.rs errors are gone. Now I need to update the acp.rs file. Let me move to the next task:I need to extract text from initial_content for acp.rs. Let me update line 340:Now I need to add the extract_text_from_content helper function. Let me check the top of the file to add it in the right place:Let me add the helper function near the top of the file, before the impl blocks:Now let me add the extract_text_from_content helper function just before the AgentAcpBackend struct:Now let me test if AcpBackend compiles:Good, the acp.rs errors are gone. Now I have only two remaining errors - both are in call sites that construct CodergenRunRequest. Let me update PromptHandler first. Let me read prompt.rs:I can see that agent.rs:292 and fan_in.rs:275 have the construction errors. Let me read those sections:I need to update this to use `initial_content`. However, this is in the agent handler execute method, and according to the plan, I'm not supposed to do the image propagation logic yet (that's Slice 5). For now, I just need to wrap the prompt in a Vec<ContentPart>. Let me update it:Now let me read the fan_in.rs file:I also need to add the ContentPart import to fan_in.rs. Let me check the imports:The import is already done inline. Now let me test if everything compiles:Excellent! The workspace compiles. Now let me mark the task as complete and run the tests:Let me wait for the tests to complete:The tests are still running. Let me check the output:Let me stop the task and try running a smaller test set:I see there are test cases that also construct CodergenRunRequest with `prompt`. I need to update those as well. Let me find and fix them:

1. Two in agent.rs (lines 1254, 1315) - these are accessing `request.prompt`
2. Many in acp.rs tests (lines 481, 529, 599, 647, 686, 736, 790, 832, 885)
3. One in api.rs test (line 2869)

Let me start with the agent.rs ones:This test needs to extract text from initial_content. Let me update it:There are two occurrences. Let me use replace_all:Now let me update the test cases in acp.rs. Let me read one to understand the pattern:I need to replace all `prompt: "..."` with `initial_content: vec![ContentPart::Text("...".to_string())]`. Let me do this systematically:Let me do a replace-all for the common case:Now the cancel one:Now let me update the api.rs test:Now let me try compiling the tests again:The ContentPart is not in scope in the test module. I need to add a use statement. Let me check where the tests module starts:I need to add the ContentPart import to the test module:Now let me try building again:Good progress, the lib tests compile. Now there are errors in the integration tests. Let me fix those:Now let me find the other error at line 6697:I need to extract text from initial_content here as well:Now let me build the tests again:Excellent! The tests compile. Now let me run them:Perfect! All lib tests pass. Now let me commit this work and then check if I need to do the refactor step. First, let me commit:According to the plan step 10, I should extract the `extract_text` helper to a shared utility. Let me check if this makes sense. I have the same logic in several places:
1. acp.rs - has `extract_text_from_content` function
2. agent.rs test - inline extraction in PromptCapturingBackend
3. integration.rs test - inline extraction in MockCodergenBackend and LlmCodergenBackend

Let me create a shared helper. I'll add it to the agent.rs module since that's where CodergenRunRequest is defined:Let me add the helper function right after the structs:Now let me update acp.rs to use this shared helper:Now let me update the test code in agent.rs to use the helper:Now let me update the integration.rs test file:Now let me build and run the tests again to make sure everything still works:Good, it compiles. I need to remove the unused ContentPart import from acp.rs since we're now using the shared helper:Now let me run the tests to verify everything still works:Perfect! All tests still pass. Let me mark the refactor as complete and commit the work:Now let me commit this work. First, let me run the full suite one more time:Excellent! All 1182 tests pass. Now let me update the plan to mark this slice as complete and commit:Now let me commit this slice:Perfect! Slice 4 is complete. All tasks are done, tests are green, and the work is committed. Let me provide a summary of what was accomplished:

## Slice 4 Summary

**Completed Steps:**
1. ✅ Changed `CodergenRunRequest.prompt: &'a str` to `initial_content: Vec<ContentPart>`
2. ✅ Updated AgentApiBackend::run to use `session.process_message(initial_content, ...)`
3. ✅ Updated AcpBackend::run to extract text from initial_content before passing to run_turn
4. ✅ Updated all CodergenRunRequest construction sites in agent.rs and fan_in.rs
5. ✅ Updated all test cases in acp.rs, api.rs, agent.rs, and integration.rs
6. ✅ Added `extract_text_from_content` helper function in agent.rs
7. ✅ Refactored to use the shared helper across all locations

**Test Results:**
- All 1182 fabro-workflow tests pass (988 lib tests + 194 integration tests)
- No compilation warnings except for an unrelated unused import in fabro-agent

**Files Modified:**
- `lib/crates/fabro-workflow/src/handler/agent.rs` - Changed CodergenRunRequest struct, added helper, updated tests
- `lib/crates/fabro-workflow/src/handler/llm/api.rs` - Updated all process_message calls, updated test
- `lib/crates/fabro-workflow/src/handler/llm/acp.rs` - Updated to extract text, updated tests
- `lib/crates/fabro-workflow/src/handler/fan_in.rs` - Updated construction site
- `lib/crates/fabro-workflow/tests/it/integration.rs` - Updated test mocks
- `docs/plans/2026-07-21-agent-session-image-input-plan.md` - Marked slice complete

**Next Slice:** Slice 5 (Agent handler image propagation) is ready to begin.

{"preferred_next_label": "Next slice"}