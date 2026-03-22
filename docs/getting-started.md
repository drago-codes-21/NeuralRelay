# Getting Started

## Prerequisites

- **Claude Code** installed (version 1.0.33 or later)
- A project directory you want to use NeuralRelay in (any language, any framework)

That's it. NeuralRelay has no external services, no API keys, no database, and no build step. Everything runs locally through Claude Code.

## Installation

**From GitHub:**

Clone or add the plugin repository, then reference it from your Claude Code configuration. Refer to the [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code) for the latest instructions on installing third-party plugins, as the mechanism may change between versions.

Repository: `https://github.com/drago-codes-21/NeuralRelay`

## First Run — Full Pipeline

### 1. Navigate to your project

```bash
cd my-project
```

### 2. Initialize NeuralRelay

```
/neuralrelay init
```

This creates two things:
- `.neuralrelay/handoffs/` — directory where handoff chain data is stored (gitignored by default)
- `neuralrelay.yaml` — pipeline configuration file (committed to your repo so your team shares the same config)

### 3. Run your first pipeline

```
/neuralrelay start "add input validation to the user registration endpoint"
```

Four agents run in sequence:

| Stage | Agent | What it does |
|---|---|---|
| Requirements | BA Agent | Explores your codebase, writes testable acceptance criteria |
| Implementation | Dev Agent | Writes the code and tests, documents coverage gaps |
| Testing | Test Agent | Tests the gaps the dev flagged, tries to break believed properties |
| Review | Review Agent | Cross-references everything, scores quality, issues verdict |

This typically takes 10-25 minutes depending on task complexity and codebase size.

### 4. Check results

```
/neuralrelay status     # Quick summary: verdict, score, token usage
/neuralrelay chain      # Visual breakdown of each stage
```

## First Run — Quick Mode

For simpler tasks where you already know the requirements:

```
/neuralrelay quick "fix the null pointer exception in UserService.getProfile when email is null"
```

Quick mode runs 2 agents (Dev + Test), takes 5-12 minutes, and costs less. The dev agent extracts its own requirements from your task description instead of waiting for a BA stage. There is no review stage — the test results are the final output.

Use quick mode for bug fixes, small features, and refactors. Use the full pipeline for new features, complex changes, or anything security-sensitive.

## What You'll See

After a full pipeline run, `/neuralrelay chain` produces output like this:

```
Pipeline Run: 20260322-143052
Task: "Add OAuth2 login with Google and GitHub providers"
Status: changes_requested

-- Requirements (BA Agent) -------------------
   Complexity: complex
   Criteria: 7 acceptance criteria defined
   Scope: mobile-responsive layout, OAuth providers other than Google/GitHub
   Modules: 5 affected

-- Implementation (Dev Agent) ----------------
   Files: 5 created, 4 modified
   Tests: 18 written, 18 passing
   Coverage Gaps: 5 identified
   Properties: 5 stated (2H 2M 1L)
   Tokens: 45,200

-- Testing (Test Agent) ----------------------
   Tests: 9 total, 7 passing, 2 failing
   Properties: 2 confirmed, 1 FALSIFIED, 1 inconclusive
   Bugs: 1 medium, 1 low
   Focus: token refresh race condition in src/lib/tokens.ts:45-67
   Tokens: 9,200

-- Review (Review Agent) ---------------------
   Verdict: changes_requested
   Quality: 7/10
   Issues: 3 (1 medium, 2 low)
   Chain Quality: req 9/10, dev 8/10, test 8/10

   Total: 27,500 tokens | ~$4.12 estimated cost
```

## Cost Expectations

Be realistic about costs:

| Mode | Typical Range | Best For |
|---|---|---|
| Full pipeline (4 stages) | $2-8 per run | New features, complex changes |
| Quick mode (2 stages) | $1-3 per run | Bug fixes, small features |

These vary with task complexity and codebase size. A simple bug fix in a small project costs less. A complex feature in a large monorepo costs more.

NeuralRelay adds overhead for writing and reading structured handoffs — each agent spends tokens producing JSON. But it is designed to reduce the larger overhead of agents re-discovering context. For medium-to-complex tasks, the estimated reduction in redundant token spend is 30-50% compared to running the same agents without handoffs. This estimate is based on architectural analysis of multi-agent coordination overhead, not independent benchmarking. Actual savings depend on task complexity, codebase size, and how well agents follow the handoff protocol.

**When the overhead isn't worth it:** For trivial tasks (typo fixes, config changes, renaming a variable), the pipeline overhead exceeds the benefit. Just use Claude Code directly.

## Next Steps

- [How It Works](how-it-works.md) — understand the pipeline and why handoffs matter
- [Project Setup](project-setup.md) — how to make NeuralRelay work well with your project (CLAUDE.md tips, good task descriptions)
- [When to Use](when-to-use.md) — honest guidance on when NeuralRelay helps and when it doesn't
- [Pipeline Configuration](pipeline-configuration.md) — customize stages, models, and templates
- [Commands Reference](commands-reference.md) — all 6 commands with examples
