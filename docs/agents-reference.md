# Agents Reference

## BA Agent (Business Analyst)

**Role:** Translates raw task descriptions into structured, testable requirements that downstream agents can act on without ambiguity.

**Reads:** Raw task description from `/neuralrelay start` command.

**Produces:** Requirements handoff (`requirements.schema.json`) — acceptance criteria, technical constraints, out-of-scope boundaries, affected modules, domain glossary.

**Model preference:** Sonnet — requirements analysis doesn't need Opus-level reasoning for most tasks.

**Key behaviors:**
- Explores the actual codebase (reads CLAUDE.md, directory structure, existing code) before writing requirements
- Every acceptance criterion must be mechanically testable — "auth works" is rejected in favor of "callback returns 302 on valid code"
- Identifies affected modules by reading real files, not guessing from the task description
- Estimates complexity based on file counts and integration points (simple/medium/complex)
- Defines out-of-scope items to prevent the dev agent from gold-plating

**Limitations:**
- Cannot verify technical feasibility. If the task requires technology the codebase doesn't use, the BA agent may not flag this.
- Requirements quality depends on how well the codebase is documented. Projects without CLAUDE.md or README produce weaker requirements.
- Cannot ask clarifying questions interactively during the pipeline — it either works with what it has or flags ambiguity in technical_constraints.
- For vague task descriptions ("make it better"), it will attempt to ask the user for clarification, but this only works if the user is actively monitoring the session.

---

## Dev Agent (Senior Developer)

**Role:** Implements features based on structured requirements and produces an honest dev-to-test handoff documenting what was tested, what wasn't, and what it believes but hasn't proven.

**Reads:** Requirements handoff (full mode) or raw task description (quick mode). Rework feedback file if in a rework cycle.

**Produces:** Dev-to-test handoff (`dev-to-test.schema.json`) — files changed, functions added, unit tests, coverage gaps, believed properties, known risks.

**Model preference:** Opus — implementation quality and coverage gap honesty are significantly better with Opus than Sonnet.

**Key behaviors:**
- Reads the requirements handoff before writing any code (respects acceptance criteria, constraints, and scope)
- Follows existing codebase conventions for naming, file organization, and patterns
- Writes unit tests alongside implementation
- Produces an honest assessment of coverage gaps (minimum 3) and believed properties with confidence levels
- In quick mode, extracts inline requirements from the task description
- In rework cycles, makes targeted fixes based on the rework file, not a full rewrite

**Limitations:**
- Quality of coverage_gaps depends on the model's self-awareness. Opus produces more honest, specific gaps than Sonnet. With Sonnet, gaps tend to be vaguer.
- May not catch its own architectural mistakes — it implements what the requirements say, even if the approach has fundamental design issues.
- Has a 40-turn limit. Very large tasks may exhaust this limit before completing implementation and handoff writing.
- Cannot run integration tests or verify behavior that requires external services.

---

## Test Agent (Test Engineer)

**Role:** Writes targeted tests that maximize new information per token spent, following a strict priority order: coverage gaps, then low-confidence properties, then uncovered criteria, then known risks.

**Reads:** All prior handoffs in the chain directory (requirements + dev-to-test in full mode, dev-to-test only in quick mode).

**Produces:** Test-to-review handoff (`test-to-review.schema.json`) — test results, property verification (confirmed/falsified/inconclusive), bugs found with counterexamples.

**Model preference:** Sonnet — test writing is structured enough that Sonnet handles it well while saving cost.

**Key behaviors:**
- Tests in strict priority order: coverage gaps (highest value) > low/medium confidence properties > uncovered acceptance criteria > known risks
- Actively tries to falsify developer-believed properties — finding a counterexample is a real bug
- Reports bugs with minimal counterexamples (the smallest input that triggers the failure)
- Does not re-test functionality the dev already covered with passing tests
- Provides a specific recommended_focus_for_reviewer (file path and line range, not vague directions)

**Limitations:**
- Cannot test things that require external services (databases, APIs, third-party auth) unless mocks or test infrastructure are already set up in the project.
- If the dev handoff has empty or vague coverage_gaps, the test agent falls back to self-directed testing, which is less efficient.
- Has a 30-turn limit. Complex tasks with many coverage gaps may require prioritization.
- Cannot run the application or verify UI behavior — testing is limited to unit and integration tests that can run in the development environment.
- Property falsification depends on test design creativity. Some subtle bugs require adversarial test design that the model may not generate.

---

## Review Agent (Tech Lead Reviewer)

**Role:** Reviews code and the handoff chain in a fresh context (no implementation bias), cross-references requirements against implementation against tests, and scores both code quality and handoff chain quality.

**Reads:** ALL handoff files in the chain directory plus the actual code changes. Runs in a fresh context intentionally — it has no memory of implementation decisions.

**Produces:** Review-final handoff (`review-final.schema.json`) — verdict, quality score, issues, chain quality scores, recommendations.

**Model preference:** Opus — deeper review analysis and more nuanced quality scoring.

**Key behaviors:**
- Cross-references acceptance criteria against implementation against test results — flags gaps in any direction
- Looks for issues other agents missed: security, performance, architecture, correctness
- Scores the handoff chain quality itself (requirements clarity, dev handoff completeness, test coverage adequacy) — this meta-feedback improves the system over time
- Issues a calibrated verdict: approved (score 7+), changes_requested (score 4-6), needs_rework (score 1-3)
- During rework cycles, focuses specifically on whether previously-identified issues are resolved

**Limitations:**
- Reviews the handoff chain and code, but cannot run the application or verify runtime behavior.
- Quality scoring is subjective — different runs may produce slightly different scores for similar code.
- Cannot verify claims about external service behavior (e.g., "the API returns 403 when permissions are revoked").
- 20-turn limit means very large changesets may not get fully reviewed.
- Fresh context is both a strength (no bias) and a weakness (no memory of why certain implementation decisions were made).

---

## Security Agent

**Role:** Performs a focused security audit of code changes, looking exclusively for vulnerabilities. Not a general code reviewer — focuses only on security.

**Reads:** All prior handoffs in the chain directory, with focus on the dev handoff's integration_points and files_changed.

**Produces:** Test-to-review handoff using `test-to-review.schema.json` — security-specific findings with exploit scenarios and suggested fixes.

**Model preference:** Opus — security analysis benefits from deeper reasoning about attack vectors.

**Key behaviors:**
- Checks every changed file for: input validation, injection (SQL, XSS, command), auth bypass, secrets exposure, insecure defaults
- Checks new dependencies for known CVEs
- Every finding includes a concrete exploit scenario, not just "this could be insecure"
- Severity reflects actual exploitability: critical (RCE/data breach), high (auth bypass), medium (info leak), low (theoretical)
- Does not review code style or general correctness — leaves that to the review agent

**Limitations:**
- Only audits code that was changed in this pipeline run — does not perform a full codebase security audit.
- Cannot test for runtime vulnerabilities (e.g., timing attacks, memory corruption) — analysis is static.
- Cannot verify that external services are configured securely (e.g., CORS headers on the server).
- Dependency vulnerability checks are limited to what the model knows about CVEs — it doesn't query a live vulnerability database.
- Not included in the default pipeline — must be explicitly added to `neuralrelay.yaml` (see the enterprise-secure template).

---

## Process Manager

**Role:** Orchestrates the pipeline — invokes agents in sequence, validates handoffs against schemas, tracks token usage, and manages rework loops. Not a coding agent.

**Note:** The process manager is not a pipeline stage that produces handoffs. It is the coordinator that runs the pipeline defined in `neuralrelay.yaml`.

**Key behaviors:**
- Reads `neuralrelay.yaml` and executes stages in order
- Assigns handoff output paths to each agent (sequence-numbered by pipeline position)
- Validates handoff JSON against schemas before allowing the next stage to proceed
- Manages rework loops: creates `rework-{N}.json` files with issues from the review, re-invokes dev and test agents
- Enforces `max_rework_cycles` — escalates to the user if the limit is reached
- Tracks and reports token usage per agent per run

**Limitations:**
- Cannot recover from agent crashes mid-turn — if an agent fails, partial handoffs may be incomplete.
- Rework loops re-run the full dev-test(-review) sequence, not individual stages.
- Does not dynamically adjust the pipeline based on results — it follows the configured sequence exactly.
