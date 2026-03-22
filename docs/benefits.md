# What NeuralRelay Does For You

NeuralRelay is a Claude Code plugin that adds structured communication between agents in multi-agent development pipelines. It is a new tool that has not been independently benchmarked. The benefits below describe what the architecture is designed to deliver based on how the handoff protocol works. Your actual results will depend on task complexity, codebase characteristics, and how well the agents follow handoff instructions.

## Agents Stop Re-Discovering What Previous Agents Already Know

**How it works:** Without handoffs, every agent reads the full codebase from scratch to understand what happened before it. The test agent doesn't know what the developer built, which files changed, or what was already tested. It spends 3,000-8,000 tokens reading files to reconstruct this context.

With NeuralRelay, downstream agents read a structured JSON handoff (~500 tokens) that tells them exactly what changed, what was tested, and where to focus. The test agent reads the dev-to-test handoff and immediately knows which files were modified, what functions were added, and what the developer already tested. It skips the exploration phase entirely.

**Honest caveat:** The actual token savings depend on codebase size (larger codebases have more to re-explore, so savings are greater) and task complexity (simple tasks have less context to transfer, so the handoff overhead may not pay for itself). The estimated savings of 30-50% on redundant token spend is an architectural projection, not independently measured data.

## Testing Targets Gaps Instead of Re-Testing Known-Good Code

**How it works:** The dev-to-test handoff includes three fields that direct the test agent's work:

- **coverage_gaps** — things the developer explicitly knows are untested
- **properties_believed** — claims about code behavior with confidence levels
- **known_risks** — edge cases identified but not addressed

The test agent prioritizes these in order: gaps first (highest value), then low-confidence properties, then uncovered acceptance criteria, then risks. This means every test token targets new information instead of re-confirming what 18 passing unit tests already proved.

**Concrete example:** If the developer writes "I didn't test concurrent token refresh — two simultaneous calls may both try to use the same refresh token" in coverage_gaps, the test agent writes a concurrent test for exactly that scenario. Without this handoff, the test agent might spend its token budget testing the happy path that already has 12 passing unit tests, and never get to the concurrency issue.

**Honest caveat:** This works well when the dev agent writes specific, honest gaps. Vague gaps ("needs more testing") don't help the test agent focus. The schema enforces a minimum of 3 coverage_gaps, but it enforces presence, not quality. Opus-tier models tend to produce more specific, actionable gaps than Sonnet.

## Property Falsification Catches Bugs That Random Testing Misses

**How it works:** The developer states what they believe is true about the code in the `properties_believed` field, with confidence levels and evidence. The test agent treats these as falsification targets — it specifically tries to disprove them. When a believed property is disproven with a counterexample, that's a documented bug.

**Concrete example:** Developer believes "refresh_session is idempotent — calling it twice returns the same session" with medium confidence and notes "no concurrent test exists." Test agent calls refresh_session twice simultaneously and discovers it creates duplicate sessions because the upsert has a race window. This is a real bug that generic test generation would likely miss because the function looks correct on single-threaded execution.

**Honest caveat:** Not all properties can be tested automatically. Properties involving external services (OAuth provider behavior), timing-dependent behavior (race conditions that require precise timing), or UI rendering cannot be falsified without infrastructure beyond what the test agent has access to. The test agent marks these as "inconclusive" rather than falsely confirming them.

## Fresh-Context Review Prevents Self-Grading Bias

**How it works:** The review agent runs in an independent context that never participated in implementation. It reads the handoff chain cold — requirements, implementation decisions, test results — and evaluates both the code and the quality of the handoff chain itself.

This addresses a documented problem: when the same context that generates code also reviews it, the review inherits the same blind spots and assumptions. Anthropic's own guidance recommends separate sessions for writing and reviewing code because self-grading tends to confirm rather than challenge.

**Honest caveat:** The review agent reads handoffs and source code, but does not execute the code. It can catch logic errors visible in the source (off-by-one, null handling, type mismatches) but not runtime bugs that only surface under execution (memory leaks, performance degradation under load, UI rendering issues).

## Every Decision Is Documented in the Handoff Chain

**How it works:** The handoff chain in `.neuralrelay/handoffs/` is a structured audit trail of the entire pipeline run:

- What requirements were defined (acceptance criteria, constraints, scope)
- What was built (files changed, functions added, design decisions)
- What was and wasn't tested (coverage gaps, property verification results)
- What bugs were found (with minimal counterexamples)
- What the reviewer concluded (verdict, quality score, issues)

This persists on disk as JSON files after the session ends. The `/neuralrelay chain` command renders it as a human-readable summary.

**Practical value:** Useful for understanding past decisions ("why did we implement it this way?"), onboarding new developers to recent changes, post-mortem analysis when bugs reach production, and regulated environments that need documentation of AI-assisted development.

**Honest caveat:** Handoff files are structured JSON — informative but not pretty without the chain visualization command. The quality of the audit trail depends on the quality of the handoffs themselves. Vague handoffs produce a vague audit trail.

## Forced Honesty About What Isn't Done

**How it works:** The JSON schema for the dev-to-test handoff requires a minimum of 3 entries in `coverage_gaps`. The agent literally cannot produce a valid handoff without admitting at least 3 things it didn't test. The process manager validates the handoff against the schema before the pipeline continues.

This counters the tendency for AI agents to declare work "done" without acknowledging gaps. By making gap disclosure structural (schema-enforced) rather than optional (prompt-requested), the protocol ensures downstream agents always have specific targets.

**Honest caveat:** Schema enforcement guarantees the presence of gaps, not their quality. An agent can write three vague gaps ("edge cases not fully tested", "error handling could be improved", "more testing needed") to pass validation. In practice, Opus-tier models produce specific, actionable gaps ("race condition in concurrent token refresh not tested") while Sonnet sometimes produces vaguer ones. The protocol is designed to work best with Opus for the dev stage.

## Pipeline Configuration Lets Teams Define Their Own Process

**How it works:** The pipeline is defined in `neuralrelay.yaml` — a YAML file committed to the project repo. Teams can:

- **Add agents:** Insert a security review stage, a compliance check, or a custom domain-specific agent
- **Remove agents:** Skip the BA stage for experienced teams who write their own requirements
- **Reorder agents:** The TDD template puts testing before development
- **Set model tiers:** Use Opus for critical stages (dev, review) and Sonnet for cheaper stages (BA, test)

Three pre-built templates cover common workflows: startup-fast (2 stages, all Sonnet), enterprise-secure (5 stages with security agent), and tdd-strict (tests written before code).

**Practical value:** An enterprise team can enforce security review as a mandatory pipeline stage. A solo developer can run a lightweight 2-stage pipeline. The same handoff protocol works regardless of topology because agents discover prior handoffs by reading the chain directory, not by expecting hardcoded filenames.

**Honest caveat:** Custom agents need to be written as markdown files following the existing agent templates. There is no drag-and-drop pipeline builder or visual editor — it's YAML configuration and markdown agent definitions.

## Process Improves Over Time Through Meta-Feedback

**How it works:** The review agent doesn't just rate the code — it rates the handoff chain quality on three dimensions:

- **requirements_clarity** — Were the BA's acceptance criteria testable and specific?
- **dev_handoff_completeness** — Were coverage gaps honest and specific? Were believed properties useful?
- **test_coverage_adequacy** — Did the tester follow priority order? Were counterexamples minimal?

The `/neuralrelay report` command aggregates these scores across multiple runs to identify patterns: "dev_handoff_completeness averaging 4/10 — coverage_gaps may be too vague."

**Honest caveat:** This requires multiple pipeline runs to generate useful patterns. A single run provides a quality score but not enough data for trend analysis. The scoring itself is subjective — the review agent's assessment may vary between runs for similar quality levels.

## What NeuralRelay Does NOT Do

This section is as important as the benefits above:

- **Does not make individual agents smarter.** It structures communication between agents, not the agents themselves. A poorly-prompted agent produces poor handoffs regardless of the protocol.
- **Does not guarantee finding all bugs.** It improves the probability of finding bugs by directing test effort toward known gaps, but cannot prove correctness or catch every issue.
- **Does not reduce costs for trivial tasks.** The pipeline overhead (spawning 2-4 agents, writing handoff JSON, validating schemas) exceeds the benefit for simple changes. Use Claude Code directly for small tasks.
- **Does not replace human judgment.** It provides better structured information for humans to review. The verdict is advisory, not binding.
- **Does not work without Claude Code.** It is a plugin, not a standalone tool. It cannot run from CI/CD, command line scripts, or other AI platforms.
- **Has not been independently benchmarked.** The 30-50% savings estimate is based on architectural analysis of how handoffs reduce redundant exploration, not on measured data from controlled experiments. Real-world performance across diverse codebases, team sizes, and task types is not yet proven.
- **Does not guarantee agents follow the protocol perfectly.** Model behavior is non-deterministic. Agents occasionally produce vague coverage gaps, ignore priority ordering, or write handoffs that technically validate but aren't useful. This is a known limitation of prompt-based systems.

## Closing

NeuralRelay is a v1 tool. It introduces structured handoff protocols for multi-agent development — a concept we believe is valuable based on documented problems in multi-agent coordination. The protocol is designed carefully and tested against example tasks, but real-world performance across diverse codebases, team sizes, and task types is not yet proven. We're releasing it to the community to validate the concept, gather feedback, and improve. If you try it and it helps, tell us what worked. If it doesn't, tell us what broke. Both are equally valuable.
