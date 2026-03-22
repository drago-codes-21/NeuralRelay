---
name: handoff-protocol
description: Structured Handoff Protocol (SHP) knowledge — teaches agents how to write and consume typed handoffs between pipeline stages
autoLoad: true
---

# Structured Handoff Protocol (SHP)

The Structured Handoff Protocol is NeuralRelay's core mechanism for eliminating token waste in multi-agent pipelines. Instead of each agent re-discovering context from scratch, typed JSON contracts pass precise information between stages. This is designed to reduce redundant token spend — estimated at 30-50% for medium-to-complex tasks — that would otherwise go toward redundant codebase exploration and context reconstruction. Actual savings depend on task complexity and handoff quality.

## Handoff Types

**1. Requirements** (`requirements.schema.json`)
BA agent → All downstream agents. Contains the task summary, testable acceptance criteria, technical constraints, out-of-scope boundaries, affected modules, and domain glossary. Every downstream agent reads this.

**2. Dev-to-Test** (`dev-to-test.schema.json`)
Developer → Test Engineer. The most critical handoff. Contains files changed, functions added, unit test summary, coverage gaps, properties believed (with confidence + evidence), known risks, and integration points. The coverage_gaps and properties_believed fields are what make this protocol novel.

**3. Test-to-Review** (`test-to-review.schema.json`)
Test Engineer → Reviewer. Contains test results, property verification (confirmed/falsified/inconclusive), bugs found with minimal counterexamples, and the recommended focus area for the reviewer.

**4. Review Final** (`review-final.schema.json`)
Reviewer → Pipeline completion. Contains the verdict, quality score, issues found, chain quality meta-assessment, total tokens, and process improvement recommendations.

## Rules for Writing Good Handoffs

### Coverage Gaps Must Be Specific
- Bad: "needs more testing"
- Bad: "edge cases not covered"
- Good: "race condition when two concurrent token refresh requests hit the cache simultaneously"
- Good: "error handling for malformed JWT with valid header but corrupted payload not tested"

### Properties Must Be Testable with Evidence
- Bad: "the code works correctly"
- Bad: "authentication is secure"
- Good: `{ "property": "validateToken returns false for expired tokens", "confidence": "high", "evidence": "3 unit tests with tokens at -1s, 0s, +1s relative to expiry" }`
- Good: `{ "property": "refreshAccessToken is idempotent within a 5-second window", "confidence": "low", "evidence": "implemented a cache key but no concurrent test" }`

### Bug Counterexamples Must Be Minimal
- Bad: "sending a large request causes a crash"
- Good: "calling validateOAuthCallback({ state: '', code: 'valid' }) throws TypeError instead of returning false — empty string state bypasses the null check but fails at .split()"

### Out-of-Scope Prevents Waste
Every item in out_of_scope saves downstream agents from exploring and potentially implementing excluded work. Be specific:
- Bad: "other features"
- Good: "mobile-responsive layout for the OAuth login page"
- Good: "OAuth providers other than Google and GitHub"

## Anti-Patterns

1. **Empty or thin coverage_gaps**: There are ALWAYS gaps. If coverage_gaps has fewer than 3 entries, the developer isn't looking hard enough. Minimum 3 entries.

2. **All-high confidence properties**: If every property is "high" confidence, the developer is either overconfident or only listing trivial properties. Include medium/low confidence items — those are where bugs hide.

3. **Missing counterexamples in bug reports**: A bug without a counterexample forces the developer to rediscover the failure condition. Always include the minimal triggering input.

4. **Untestable acceptance criteria**: "System should be reliable" cannot be tested. Every criterion must map to a pass/fail test.

5. **Vague recommended_focus_for_reviewer**: "Check the auth code" wastes the reviewer's time. Specify: "the token refresh race condition in src/auth/tokens.ts:45-67 — test is inconclusive."

## Handoff File Location

All handoffs are written to: `.neuralrelay/handoffs/{chain-id}/{NN}-{agent-name}.json`

- `{chain-id}`: YYYYMMDD-HHmmss format timestamp
- `{NN}`: Two-digit sequence number based on the agent's position in the pipeline (01, 02, 03, ...)
- `{agent-name}`: The agent that produced the handoff (ba-agent, dev-agent, test-agent, review-agent, security-agent, etc.)
- Default pipeline example: `01-ba-agent.json`, `02-dev-agent.json`, `03-test-agent.json`, `04-review-agent.json`
- Example path: `.neuralrelay/handoffs/20260322-143052/02-dev-agent.json`
- Agents discover prior handoffs by reading all .json files in the chain directory, sorted by sequence prefix
