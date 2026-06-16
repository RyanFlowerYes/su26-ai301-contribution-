# Contribution [1]: [Integration Test for Score Parsing Pipeline]

**Contribution Number:** [2]  
**Student:** [Ryan Ouardaoui]  
**Issue:** [https://github.com/truera/trulens/issues/2496]  
**Status:** [Phase II] [Complete]

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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
