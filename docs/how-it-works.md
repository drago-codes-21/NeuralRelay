# How NeuralRelay Works

## The Problem

When you run multiple AI agents on a task, each agent starts from scratch. The test agent doesn't know what the developer already tested. The reviewer doesn't know what the tester found suspicious. Every agent spends tokens reading the codebase and re-discovering context that a previous agent already established.

This isn't theoretical. Nicholas Carlini documented building a C compiler with 16 Claude agents for ~$20K, with significant costs attributed to agent re-orientation and coordination overhead. CodeRabbit's analysis found AI-generated code contains 1.7x more bugs than human code, partly because there's no structured quality feedback loop between generation and verification.

NeuralRelay doesn't make individual agents smarter. It makes the communication between agents structured and precise.

## The Pipeline

When you run `/neuralrelay start "add OAuth2 login"`, here's what happens:

### Step 1: Process Manager starts the pipeline

The process manager reads `neuralrelay.yaml`, creates a timestamped chain directory (e.g., `.neuralrelay/handoffs/20260322-143052/`), and begins invoking agents in sequence.

### Step 2: BA Agent writes structured requirements

The BA agent explores your codebase — reads CLAUDE.md, examines directory structure, checks existing code patterns. It produces a requirements handoff with:

- **Testable acceptance criteria** — not "auth should work" but "OAuth callback at /api/auth/callback/google returns 302 redirect to /dashboard on valid authorization code"
- **Affected modules** — actual file paths it found in your codebase
- **Out-of-scope boundaries** — what specifically should NOT be built (prevents dev agent from gold-plating)
- **Domain glossary** — terms that have codebase-specific meanings

This handoff is written to `01-ba-agent.json`.

### Step 3: Dev Agent implements and writes an honest assessment

The dev agent reads the BA's handoff — not the raw task description, but structured requirements with clear acceptance criteria and constraints. It implements the feature, writes unit tests, then produces the most important handoff in the protocol:

- **coverage_gaps** — things it knows it didn't test. "Race condition when two concurrent refresh calls hit the same token." "Error handling for network timeout in OAuth callback." Minimum 3 entries enforced by schema.
- **properties_believed** — things it thinks are true, with confidence levels and evidence. "validateOAuthCallback never throws for any input (high confidence, 12 unit tests)." "refreshAccessToken is safe for concurrent calls with different users (medium confidence, no concurrent test)."
- **known_risks** — edge cases, race conditions, and concerns it identified during implementation.

This honesty is the entire point of the protocol. Every gap the dev agent admits saves the test agent from discovering it independently.

### Step 4: Test Agent targets the gaps

Instead of spending 8,000 tokens re-reading every file and guessing where bugs might hide, the test agent reads the dev handoff and immediately knows:

1. **What to test first**: coverage gaps (known unknowns — highest value)
2. **What to try to break**: medium/low confidence properties (falsification targets)
3. **What to skip**: the 18 unit tests the dev already wrote and verified

It writes targeted tests, classifies each believed property as confirmed/falsified/inconclusive, and reports bugs with minimal counterexamples.

### Step 5: Review Agent cross-references the chain

The review agent runs in a fresh context — it has no implementation bias. It reads the entire handoff chain and the actual code, then:

- Cross-references acceptance criteria against implementation against test results
- Checks for issues other agents missed (security, performance, architecture)
- Scores both the code quality AND the handoff chain quality (were the handoffs useful?)
- Issues a verdict: approved, changes_requested, or needs_rework

If the verdict is changes_requested or needs_rework, the process manager can trigger a rework loop — sending the review's issues back to the dev agent for targeted fixes.

## Token Usage: With and Without Handoffs

Here's an illustrative comparison for a medium-complexity feature:

**Without structured handoffs:**

| Agent | Tokens | Activity |
|---|---|---|
| Dev | ~10,000 | Explores codebase, implements, writes tests |
| Test | ~8,000 | Re-explores the same codebase, re-discovers what was built, writes tests |
| Review | ~6,000 | Re-reads everything a third time |
| **Total** | **~24,000** | Significant redundancy in exploration |

**With NeuralRelay handoffs:**

| Agent | Tokens | Activity |
|---|---|---|
| Dev | ~11,000 | Same as above + writes structured handoff (~1,000 extra) |
| Test | ~4,000 | Reads handoff, goes straight to gaps — no codebase re-exploration |
| Review | ~4,000 | Reads chain, does targeted review — full context from handoffs |
| **Total** | **~19,000** | Minimal redundancy, higher quality output |

The dev agent spends slightly more tokens writing the handoff. But the test and review agents save significantly more by not re-discovering context.

**Caveats:** These numbers are illustrative. Actual savings depend on task complexity (complex tasks save more because there's more context to transfer), codebase size (large codebases have more to re-explore), and how honestly agents fill out handoffs (vague handoffs save less).

## The Key Insight

The dev-to-test handoff transforms testing from "explore everything and hope you find bugs" into "here are the specific weak spots — go attack them." This is the same approach human developers use in code review — you tell the reviewer "I'm not confident about the concurrent access path" and they focus there.

The difference is that NeuralRelay enforces this with typed schemas. The dev agent can't skip coverage_gaps (minimum 3 required). It can't claim everything is high confidence without evidence. The schema makes honesty structural, not optional.

## Rework Loops

If the review agent requests changes, the process manager creates a `rework-{N}.json` file containing the specific issues to address. The dev agent reads this file and makes targeted fixes — not a full rewrite. The test agent then re-tests only the changed code. This can happen up to 2 times (configurable) before the pipeline stops and asks you to intervene.

## Quick Mode

Quick mode (`/neuralrelay quick`) skips the BA and Review stages. The dev agent extracts requirements directly from your task description and writes them in an `inline_requirements` field. The test agent reads only the dev handoff. No rework loops.

This is 30-50% cheaper but produces less thorough results — no independent requirements analysis, no formal review verdict, no quality scoring.

## What NeuralRelay Does NOT Do

- It does not run your application or verify UI behavior
- It does not guarantee finding all bugs — it finds more bugs than undirected testing, not all bugs
- It does not replace your CI/CD pipeline, linter, or type checker
- It does not work without Claude Code (it's a Claude Code plugin, not a standalone tool)
- It does not make a single agent faster — it makes multi-agent communication more efficient
