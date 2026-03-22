# The Structured Handoff Protocol

## What Is a Handoff?

A handoff is a JSON file that one agent writes when it finishes work and the next agent reads before starting. It contains structured information about what was done, what wasn't done, and what the next agent should focus on.

Handoffs are not logs or transcripts. They are contracts — structured, typed, schema-validated agreements about the state of work. Each handoff conforms to a JSON Schema (draft-07) with required fields, type constraints, and `additionalProperties: false` to prevent drift.

## The Four Handoff Types

### 1. Requirements Handoff

**Produced by:** BA Agent | **Consumed by:** All downstream agents | **Schema:** `schemas/requirements.schema.json`

The requirements handoff translates a raw task description into structured, testable specifications. Its purpose is to eliminate ambiguity before any code is written.

**Key fields and why they matter:**

- **acceptance_criteria** (array of strings) — Each criterion must be testable by an automated test. This is the contract between "what we're building" and "how we know it's done." Bad: "Authentication should work properly." Good: "OAuth callback at /api/auth/callback/google returns 302 redirect to /dashboard on valid authorization code."
- **out_of_scope** (array of strings) — Prevents the dev agent from building things that weren't asked for. Every item here saves downstream agents from exploring and potentially implementing excluded work.
- **affected_modules** (array of strings) — Real file paths discovered through codebase exploration, not guesses. Tells the dev agent exactly where to look.
- **domain_glossary** (object) — Terms that have codebase-specific meanings. Prevents downstream agents from misinterpreting terminology.

**Trimmed example:**

```json
{
  "handoff_type": "requirements",
  "task_summary": "Add OAuth2 login with Google and GitHub providers",
  "acceptance_criteria": [
    "OAuth callback at /api/auth/callback/google returns 302 redirect to /dashboard on valid authorization code",
    "Sign-out at /api/auth/signout destroys the session and redirects to /",
    "Existing users linking a new OAuth provider retain their original account data"
  ],
  "out_of_scope": [
    "Mobile-responsive layout for the OAuth login page",
    "OAuth providers other than Google and GitHub"
  ],
  "affected_modules": ["src/lib/oauth.ts", "src/lib/session.ts", "prisma/schema.prisma"],
  "estimated_complexity": "complex"
}
```

The dev agent reads this and knows exactly what to build, what not to build, and which files to modify.

### 2. Dev-to-Test Handoff

**Produced by:** Dev Agent | **Consumed by:** Test Agent | **Schema:** `schemas/dev-to-test.schema.json`

This is the protocol's core contribution. Two fields make it novel:

**coverage_gaps** — Things the developer knows they didn't test.

Without this field, the test agent has to read every file and guess where bugs might hide. With it, the test agent goes straight to known weak spots. The schema enforces a minimum of 3 entries because there are always things you didn't test — error handling paths, concurrent access, boundary values.

```json
"coverage_gaps": [
  "Race condition when two concurrent refreshAccessToken calls for the same user execute simultaneously",
  "Token encryption key rotation — no test for reading tokens encrypted with a previous key",
  "Profile linking when Google and GitHub return different email formats (User@Gmail.com vs user@gmail.com)"
]
```

**properties_believed** — Things the developer thinks are true but hasn't proven.

These are falsification targets. The test agent actively tries to break them. When a believed property is falsified, that's a real bug discovered through structured adversarial testing.

```json
"properties_believed": [
  {
    "property": "validateOAuthCallback never throws for any input including null/undefined",
    "confidence": "high",
    "evidence": "try-catch wraps the entire body; 12 unit tests with null/undefined/empty inputs"
  },
  {
    "property": "refreshAccessToken is safe for concurrent calls with different users",
    "confidence": "medium",
    "evidence": "No shared mutable state per-user, but no concurrent test exists"
  },
  {
    "property": "State parameter is cryptographically random and single-use",
    "confidence": "low",
    "evidence": "crypto.randomBytes(32) used, but 5-minute expiry window allows theoretical replay"
  }
]
```

The test agent sees the medium-confidence property and writes a concurrent test. It sees the low-confidence property and checks whether replay is actually possible. Every test token targets new information.

**Other important fields:**
- **files_changed** — Every file modified, with change type and summary
- **functions_added** — New function signatures and purposes
- **known_risks** — Edge cases and concerns identified during implementation
- **integration_points** — External APIs, databases, and services the code touches

### 3. Test-to-Review Handoff

**Produced by:** Test Agent | **Consumed by:** Review Agent | **Schema:** `schemas/test-to-review.schema.json`

The test handoff reports what was verified, what was broken, and what couldn't be determined.

**Key fields:**

- **property_verification** — Each dev-believed property classified as confirmed, falsified, or inconclusive. Falsified properties include counterexamples — these are confirmed bugs.
- **bugs_found** — Each bug includes severity, location, a minimal counterexample (the smallest input that triggers the failure), and a suggested fix.
- **recommended_focus_for_reviewer** — The single most important thing for the reviewer to examine. Must be specific: "the token refresh race condition in src/lib/tokens.ts:45-67" not "check the auth code."

**Trimmed example:**

```json
{
  "handoff_type": "test_to_review",
  "test_summary": { "total": 9, "passing": 7, "failing": 2, "skipped": 1 },
  "property_verification": {
    "confirmed": ["validateOAuthCallback never throws — tested with 15 input combinations"],
    "falsified": ["Off-by-one in chain listing: ls -r | head -5 includes header line, shows 4 of 5 chains"],
    "inconclusive": ["Concurrent refresh safety — requires integration test environment"]
  },
  "bugs_found": [{
    "severity": "medium",
    "description": "Off-by-one: only 4 of 5 chains displayed when exactly 5 exist",
    "counterexample": "Create exactly 5 chain directories. Run the command. Expected: 5 rows. Actual: 4.",
    "suggested_fix": "Use ls -1r instead of ls -r to suppress header line"
  }],
  "recommended_focus_for_reviewer": "the off-by-one in directory listing — medium severity, silently drops data"
}
```

### 4. Review-Final Handoff

**Produced by:** Review Agent | **Consumed by:** Pipeline completion | **Schema:** `schemas/review-final.schema.json`

The final handoff contains the review verdict, quality scores, and — uniquely — meta-feedback on the handoff chain itself.

**Key fields:**

- **verdict** — `approved`, `changes_requested`, or `needs_rework`. Determines whether the pipeline completes or triggers a rework loop.
- **quality_score** (1-10) — Calibrated: 1-3 is fundamentally broken, 4-6 is functional with issues, 7-8 is good, 9 is excellent, 10 is reserved for exceptional work.
- **chain_quality** — Rates the handoffs themselves, not just the code:
  - `requirements_clarity` — Were the BA's criteria testable and specific?
  - `dev_handoff_completeness` — Were coverage gaps honest? Were properties useful?
  - `test_coverage_adequacy` — Did the tester follow priority order? Were counterexamples minimal?

This meta-feedback is what improves the system over time. If `/neuralrelay report` shows dev_handoff_completeness averaging 4/10, you know the dev agent's coverage_gaps need work.

## Schema Validation

Every handoff is validated against its JSON Schema before the pipeline continues. If an agent produces invalid JSON (missing required fields, wrong types, fewer than 3 coverage_gaps), the process manager catches the error and asks the agent to fix it.

The schemas use JSON Schema draft-07 with:
- `required` arrays listing mandatory fields
- `additionalProperties: false` preventing unstructured data
- `minItems` constraints (e.g., coverage_gaps requires at least 3)
- `enum` constraints (e.g., verdict must be one of three values)

You can find the schemas in the `schemas/` directory.

## Where Handoffs Are Stored

All handoffs are written to: `.neuralrelay/handoffs/{chain-id}/{NN}-{agent-name}.json`

- **chain-id**: Timestamp in YYYYMMDD-HHmmss format (e.g., `20260322-143052`)
- **NN**: Two-digit sequence number based on pipeline position (01, 02, 03, ...)
- **agent-name**: The agent that produced the file (ba-agent, dev-agent, test-agent, review-agent, security-agent)

Default pipeline produces: `01-ba-agent.json`, `02-dev-agent.json`, `03-test-agent.json`, `04-review-agent.json`

Agents discover prior handoffs by reading all `.json` files in the chain directory sorted by sequence prefix and identifying each by its `handoff_type` field. This makes the protocol work with any pipeline topology — adding, removing, or reordering stages doesn't break anything.

Rework feedback is stored as `rework-{N}.json` in the same directory.

## Reading Handoffs for Debugging

If something goes wrong in a pipeline run, the handoff files are your debugging tool:

```bash
# List all handoffs for the most recent run
ls .neuralrelay/handoffs/

# Read the dev's coverage gaps
cat .neuralrelay/handoffs/20260322-143052/02-dev-agent.json | jq '.coverage_gaps'

# Check what the test agent found
cat .neuralrelay/handoffs/20260322-143052/03-test-agent.json | jq '.bugs_found'

# See the review verdict
cat .neuralrelay/handoffs/20260322-143052/04-review-agent.json | jq '.verdict, .quality_score'
```

Or use `/neuralrelay chain 20260322-143052` for a formatted summary.
