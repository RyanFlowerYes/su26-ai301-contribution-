# Contribution [1]: [Integration Test for Score Parsing Pipeline]

**Contribution Number:** [2]  
**Student:** [Ryan Ouardaoui]  
**Issue:** [https://github.com/truera/trulens/issues/2496]  
**Status:** [Phase IV] [Complete]

---

## Why I Chose This Issue

The existing tests only check the regex parser by feeding it strings directly. They never run a score through the actual feedback pipeline. That's a real gap, because LLMs don't return clean numbers : they return JSON, markdown code fences, numbers buried in sentences, extra whitespace. The parser might handle a string fine in isolation and still break when the same string comes through generate_score. I wanted to close that gap.
It also fits what I can actually do right now. I don't need to understand all of TruLens, just the feedback pipeline and what the parser should output. The scope is fixed: six input formats, three assertions each. I know what "done" looks like, which is why I picked it over issues where the goal was vague.


---

## Understanding the Issue

### Problem Description

The issue is that TruLens has parser tests that directly pass raw strings into the score parser, but it does not have enough integration coverage showing that score parsing works when the output comes through the real feedback pipeline. This matters because LLM responses are often messy: they may include JSON, markdown code fences, explanations, whitespace, or numbers embedded in text. A parser can pass isolated unit tests while still failing when connected to generate_score, generate_score_and_reasons, or a feedback method.

### Expected Behavior

The feedback pipeline should consistently extract the intended numeric score from supported LLM output formats. Whether the model returns a plain number, JSON, fenced JSON, or a sentence containing the score, the pipeline should normalize the result into the expected score type/range.

### Current Behavior

The current test coverage does not fully prove that these formats work through the actual pipeline. Existing tests mostly validate the parser in isolation. During integration testing, I also found a structured JSON return-shape inconsistency in generate_score, which I captured as a strict expected failure instead of hiding it.

### Affected Components

Feedback provider score parsing
generate_score
generate_score_and_reasons
Feedback method execution path
context_relevance
Existing parser/normalization utilities
---

## Reproduction Process

### Environment Setup

I set up the TruLens repository locally from my fork, created a working branch for issue #2496, and installed the project test dependencies. The main challenge was narrowing the scope of the codebase, since TruLens is large and the issue only requires understanding the feedback parsing path. I solved this by tracing the score flow from provider output into generate_score, then checking how existing feedback tests mock or simulate provider responses.
Useful cmds:
git clone https://github.com/RyanFlowerYes/trulens.git
cd trulens
git checkout -b score-parsing-integration-tests
uv sync

To run the relevant tests:

uv run pytest tests/unit/feedback/ -q
### Steps to Reproduce

Open the existing score parser tests and observe that they mainly test parser behavior by passing strings directly.
Create mocked provider outputs that resemble realistic LLM responses, including:
plain numeric score
score inside a sentence
JSON response
fenced JSON response
output with extra whitespace
structured output edge case
Route these outputs through the real feedback pipeline using generate_score, generate_score_and_reasons, and a full feedback method such as context_relevance.
Assert that the parsed score is correctly normalized.
Observe that most cases pass, but the structured JSON return shape in generate_score exposes an inconsistency.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/truera/trulens/pull/2554
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

The root cause has two layers. First, the "issue" itself is a coverage gap, not a code defect , the parser is tested only with hardcoded strings, so nothing proves the end-to-end path survives realistic LLM output. Second, once I built the integration tests, they surfaced an actual defect: generate_score's structured-JSON branch returns the same (score, reason) tuple that generate_score_and_reasons builds, rather than extracting just the normalized float. The math is fine; the return shape is wrong, and it silently violates the -> float contract.

### Proposed Solution

Add a new integration test file that drives realistic LLM-style outputs through the real provider parsing and feedback execution path, asserting normalization to a float in [0.0, 1.0] at each entry point. Capture the structured-JSON return-shape inconsistency as a strict xfail, paired with a companion test that proves generate_score_and_reasons normalizes the identical payload correctly — so the failing test isolates the bug to generate_score's return shape and auto-flips to green the moment the JSON branch is fixed.



### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The problem is not that the regex parser has no tests. The problem is that parser tests alone do not prove the real feedback pipeline handles messy LLM outputs correctly. The fix is to add integration tests that send realistic LLM-style outputs through the actual provider parsing and feedback execution path.

**Match:** I looked for existing feedback/provider tests that already mock provider responses and call feedback methods. The closest pattern is the existing test structure around provider methods, score generation, and feedback functions. I reused that style instead of creating a separate fake pipeline.

**Plan:**Add integration test coverage for score parsing through the real feedback pipeline.
Mock realistic LLM outputs in several formats:
raw number
markdown fenced JSON
regular JSON
score in natural language
whitespace-padded score
structured JSON edge case
Test the outputs through:
generate_score
generate_score_and_reasons
one full feedback method path such as context_relevance
Assert that the final score is parsed and normalized correctly.
Mark the known structured JSON inconsistency as a strict xfail so the test documents the bug without breaking the suite unexpectedly.
Run the relevant test files locally.
Clean up formatting/lint issues before opening the PR.

**Implement:** Implementation will happen in Phase III.

Branch: https://github.com/truera/trulens/pull/2554

**Review:** Before submitting the PR, I will check the project’s CONTRIBUTING.md and make sure:

test names are clear
tests are focused and not over-broad
the PR does not change production behavior unnecessarily
the expected failure is documented clearly
formatting/linting passes
commit messages and PR description follow project conventions

**Evaluate:** The fix will be verified with automated tests. The important success condition is that realistic LLM score outputs are tested through the real feedback pipeline, not just through isolated parser calls.

Planned test checks:

parsed score equals the expected numeric value
score normalization works across supported formats
generate_score_and_reasons still preserves expected behavior
context_relevance exercises the full feedback-method execution path
the structured JSON inconsistency is captured with strict xfail

---

## Testing Strategy

### Unit Tests

-  [ x] Test case 1: generate_score normalizes every text shape (plain number, number-in-sentence, markdown-fenced JSON, whitespace-padded, prose-only) to the expected 0.2.
-  [x ]  Test case 2: generate_score accepts a float-format response ("2.5") and a custom min/max range, returning a float in [0.0, 1.0].
-  [ x] Test case 3: generate_score parses a structured BaseFeedbackResponse object correctly (a branch the text cases never exercise).



### Integration Tests

- [ ] Integration scenario 1: generate_score_and_reasons returns a (float, dict) tuple for every text shape and parses the structured ChainOfThoughtResponse object, keeping the reason text.
- [ ] Integration scenario 2: Integration scenario 2: context_relevance runs end to end across all response shapes — building the real system prompt from templates internally — and returns a normalized float. A strict xfail documents the generate_score structured-JSON tuple bug, with a companion test proving generate_score_and_reasons normalizes the same JSON to 0.2.




### Manual Testing

Ran python -m pytest test_score_parsing_pipeline.py -v against the real installed trulens-feedback package (not a stub). Result: 30 passed, 1 xfailed. The strict xfail behaved exactly as designed — it fails today because of the tuple return and would flip to passing the instant the JSON branch is fixed, so it can't silently rot.

---

## Implementation Notes

### Week [1] Progress

Pulled the actual TruLens source (generated.py, llm_provider.py, output_schemas.py) and traced the score flow from provider output into generate_score. Worked out the two interception points needed for a clean mock: _create_chat_completion for the canned response, and a pass-through endpoint.run_in_pace. Confirmed the existing tests only cover hardcoded strings, which validated the gap the issue describes.

### Week [2] Progress

Built MockLLMProvider, parametrized the six response shapes, and wired them through generate_score, generate_score_and_reasons, and context_relevance. Hit the structured-JSON return-shape inconsistency, then added the companion test to prove it was a return-shape bug rather than a parsing bug, and pinned it with a strict xfail. Tightened the file (removed dead helpers, fixed import sorting for ruff/isort), reran the full suite, and opened the PR with a bug-first description.

### Code Changes

- **Files modified:** Files modified: Added one new file, tests/integration/test_score_parsing_pipeline.py. No production code changed — this is test-only coverage plus a documented xfail.
- **Key commits:** [[Links to important commits]](https://github.com/truera/trulens/pull/2554)
- **Approach decisions:** Mock at the two narrowest seams (_create_chat_completion and run_in_pace) so the real parsing/normalization code runs unmocked. Isolate the JSON bug with a paired passing/xfailing test instead of hiding it or loosening an assertion. Route shapes through context_relevance rather than only generate_score, since that exercises more of the real production path.

---

## Pull Request

**PR Link:** [[GitHub PR URL when submitted]](https://github.com/truera/trulens/pull/2554)

**PR Description:** Added integration coverage for the score-parsing pipeline and, in doing so, surfaced an inconsistency: generate_score returns a tuple for structured-JSON responses while generate_score_and_reasons normalizes the same payload correctly (#2496); captured as a strict xfail. The PR adds a MockLLMProvider driving six realistic LLM response shapes through generate_score, generate_score_and_reasons, and context_relevance, asserting normalization to a float in [0.0, 1.0] at each entry point.

**Maintainer Feedback:**
- [06/22/26]: The PR was not merged. The issue drew several competing contributions (around five in total), and the last one submitted was the one the maintainers accepted.
I received a thanks-for-the-contribution acknowledgment from the maintainers.


**Status:** Closed - not merged

---

## Learnings & Reflections

### Technical Skills Gained

I learned how to mock an LLM provider at the right level, intercepting only _create_chat_completion and the endpoint's run_in_pace so the actual parsing and normalization code still runs, rather than mocking so much that the test no longer proves anything. I also got comfortable with pytest.mark.parametrize for response-shape matrices and with strict=True xfail as a way to record a known bug as an executable, self-resetting assertion instead of a comment.



### Challenges Overcome

The hardest part was scoping a large codebase down to just the feedback-parsing path, and then correctly diagnosing the structured-JSON failure. My first instinct was "the parser is broken," but the companion test through generate_score_and_reasons showed the same payload normalizing fine, which meant the bug was the return shape of generate_score, not the parsing math. Getting that distinction right changed the test from a vague "this fails" into a precise bug report.



### What I'd Do Differently Next Time

The issue was completely unclaimed when I picked it up, no assignee, no open PRs. The competition showed up afterward, simply because TruLens is a popular, high-visibility library and a well-scoped issue like this one draws a crowd fast. In hindsight I'd move faster once I commit to an issue like that, since being early doesn't guarantee staying ahead on a repo this active. I'd also engage the maintainer earlier in a comment to confirm the approach (test-only + documented xfail) was what they wanted, rather than presenting it all at once at PR time.


---

## Resources Used

TruLens repository source: src/feedback/trulens/feedback/generated.py, llm_provider.py, output_schemas.py
Existing tests as patterns: tests/unit/test_feedback_score_generation.py
TruLens CONTRIBUTING.md (style, pre-commit, PR conventions)
pytest docs on parametrize and xfail(strict=True)
Issue #2496 discussion thread: https://github.com/truera/trulens/issues/2496
