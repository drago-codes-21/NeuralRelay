# Pipeline Configuration

## The Default Pipeline

NeuralRelay uses `neuralrelay.yaml` in your project root to define the pipeline. Here's the default configuration created by `/neuralrelay init`:

```yaml
# NeuralRelay Pipeline Configuration
# Customize stages, models, and settings for your workflow.

pipeline:
  - name: requirements       # Stage name (used in status output)
    agent: ba-agent           # Agent file in agents/ (without .md extension)
    model_preference: sonnet  # Preferred Claude model for this stage
  - name: implementation
    agent: dev-agent
    model_preference: opus    # Opus for implementation — better code quality
  - name: testing
    agent: test-agent
    model_preference: sonnet  # Sonnet for testing — good enough, saves cost
  - name: review
    agent: review-agent
    model_preference: opus    # Opus for review — deeper analysis

settings:
  max_rework_cycles: 2        # Max retry loops when reviewer requests changes
  handoff_validation: strict   # Validate handoffs against JSON schemas
  handoff_dir: .neuralrelay/handoffs  # Where handoff chains are stored
  token_tracking: true         # Track token usage per agent
```

**model_preference** is a suggestion, not a guarantee — Claude Code uses the model configured in your session. But the process manager passes this preference when spawning each agent.

## Customizing Your Pipeline

### Adding a Security Review Stage

To add a security audit between implementation and testing:

```yaml
pipeline:
  - name: requirements
    agent: ba-agent
    model_preference: sonnet
  - name: implementation
    agent: dev-agent
    model_preference: opus
  - name: security-review        # Added stage
    agent: security-agent         # Uses agents/security-agent.md
    model_preference: opus
  - name: testing
    agent: test-agent
    model_preference: sonnet
  - name: review
    agent: review-agent
    model_preference: opus
```

The security agent reads the dev handoff, audits the code for vulnerabilities (injection, auth bypass, secrets exposure, dependency CVEs), and produces its own handoff. The test agent and review agent both read it as part of the chain.

Place the security agent after dev (so it has code to audit) and before test (so the test agent can write security-focused tests based on its findings).

### Removing the BA Stage

If your team writes its own requirements or you always know exactly what to build:

```yaml
pipeline:
  - name: implementation
    agent: dev-agent
    model_preference: opus
  - name: testing
    agent: test-agent
    model_preference: sonnet
  - name: review
    agent: review-agent
    model_preference: opus
```

Without the BA stage, the dev agent extracts requirements directly from your task description (same behavior as quick mode). It writes an `inline_requirements` field in its handoff. The test agent skips Priority 3 (uncovered acceptance criteria) since there's no BA handoff to cross-reference.

This saves tokens but means you lose independent requirements analysis. Your task description needs to be specific enough to serve as requirements.

### TDD: Tests Before Code

Reorder stages to write test specifications before implementation:

```yaml
pipeline:
  - name: requirements
    agent: ba-agent
    model_preference: sonnet
  - name: test-specification     # Test agent runs FIRST
    agent: test-agent
    model_preference: sonnet
  - name: implementation         # Dev implements to pass the specs
    agent: dev-agent
    model_preference: opus
  - name: test-verification      # Test agent runs AGAIN to verify
    agent: test-agent
    model_preference: sonnet
  - name: review
    agent: review-agent
    model_preference: opus
```

The first test stage writes test specifications from requirements (expected behaviors, edge cases). The dev agent implements code to pass them. The second test stage verifies the implementation and runs the full gap/property analysis.

## Pipeline Templates

NeuralRelay includes 3 pre-built configurations in `examples/templates/`. To use one, copy it to your project root:

```bash
cp examples/templates/enterprise-secure.yaml neuralrelay.yaml
```

### Startup Fast (`startup-fast.yaml`)

```
Dev (sonnet) -> Test (sonnet)
```

2 stages, both Sonnet, 1 rework cycle. For solo developers and startups who want speed over ceremony. No BA analysis, no review verdict. About 50% cheaper than the full pipeline.

**When to use:** Bug fixes, small features, rapid prototyping, tasks where you can describe requirements yourself.

**Trade-off:** No independent requirements analysis, no formal review, no quality scoring.

### Enterprise Secure (`enterprise-secure.yaml`)

```
BA -> Dev -> Security Review (opus) -> Test -> Review
```

5 stages with a dedicated security agent between dev and test. The security agent audits for input validation, auth bypass, secrets exposure, injection vulnerabilities, and dependency CVEs.

**When to use:** Authentication or authorization changes, payment processing, API endpoints exposed to the internet, code handling PII or sensitive data.

**Trade-off:** More tokens and time. Worth it for code where a security bug means an incident.

### TDD Strict (`tdd-strict.yaml`)

```
BA -> Test (specs) -> Dev -> Test (verify) -> Review
```

5 stages — tests are written before code. The first test stage writes test specifications from requirements. The dev agent implements to pass them. The second test stage verifies.

**When to use:** High-reliability features, complex logic with clear input/output contracts, teams that follow TDD methodology.

**Trade-off:** Two test stages means more tokens spent on testing. Catches requirement ambiguities earlier.

## Creating Custom Agents

You can add your own agents to the pipeline. Here's how:

### Step 1: Create the agent file

Create `agents/your-agent.md` with YAML frontmatter and markdown instructions:

```markdown
---
model: sonnet
tools:
  - Read
  - Glob
  - Grep
  - Bash
maxTurns: 20
skills:
  - handoff-protocol
---

You are the [Role] agent in the NeuralRelay pipeline. [One sentence describing what this agent does.]

## Responsibilities
- [What this agent does]
- [What it reads]
- [What it produces]

## Input
- Read prior handoffs from the chain directory by handoff_type
- [Specific handoffs this agent needs]

## Output
- Schema: [which schema, or describe custom output format]
- Write to: `.neuralrelay/handoffs/{chain-id}/{NN}-your-agent.json`

## Workflow
1. [Step-by-step instructions]

## Edge Cases
- [How to handle failures]
```

Use the existing agents (`agents/ba-agent.md`, `agents/security-agent.md`, etc.) as templates. Keep the file under 200 lines.

### Step 2: Define the handoff contract

Your agent must produce a valid handoff. You can either:
- **Reuse an existing schema** — if your agent produces output similar to an existing type (e.g., a security agent uses `test-to-review.schema.json`)
- **Create a new schema** — add a JSON Schema file to `schemas/` if your agent needs a unique output format

### Step 3: Add to the pipeline

Add a stage to `neuralrelay.yaml`:

```yaml
pipeline:
  # ... existing stages ...
  - name: your-stage-name
    agent: your-agent        # Matches agents/your-agent.md (without .md)
    model_preference: sonnet
```

Position matters — place your agent where it makes sense in the information flow. Agents read all prior handoffs in the chain, so an agent at position 3 has access to handoffs from positions 1 and 2.

### Step 4: Test it

Run `/neuralrelay start` on a small task and check the handoff chain with `/neuralrelay chain` to verify your agent reads and writes correctly.

## Settings Reference

| Setting | Default | Description |
|---|---|---|
| `max_rework_cycles` | 2 | Maximum times the pipeline retries after reviewer requests changes. After this limit, the pipeline stops and reports the current state. |
| `handoff_validation` | strict | Set to `strict` to validate handoffs against JSON schemas. Any other value skips validation (not recommended). |
| `handoff_dir` | .neuralrelay/handoffs | Directory where handoff chains are stored. |
| `token_tracking` | true | Whether to track and report per-agent token usage. |
