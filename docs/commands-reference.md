# Commands Reference

## /neuralrelay init

**Syntax:** `/neuralrelay init`

Initializes NeuralRelay in the current project. Creates the `.neuralrelay/handoffs/` directory for storing handoff chains and a default `neuralrelay.yaml` configuration file. Adds `.neuralrelay/handoffs/` to `.gitignore` (handoff data is ephemeral; the config file is committed).

**Example:**

```
/neuralrelay init
```

**Output:**

```
NeuralRelay initialized successfully.

Created:
  .neuralrelay/handoffs/   -- where handoff chains are stored
  neuralrelay.yaml         -- pipeline configuration (edit to customize)

Quick start:
  /neuralrelay start "add OAuth2 login"   -- run a full pipeline
  /neuralrelay status                      -- check pipeline progress
  /neuralrelay report                      -- view cost & quality analytics
```

**Common issues:**
- If NeuralRelay is already initialized, it prints a message and stops — it won't overwrite your existing config.
- If you want to reset, delete `.neuralrelay/` and `neuralrelay.yaml`, then run init again.

---

## /neuralrelay start

**Syntax:** `/neuralrelay start "task description"`

Runs a full NeuralRelay pipeline for the given task. Reads the pipeline configuration from `neuralrelay.yaml`, generates a timestamped chain ID, and invokes the process manager to orchestrate all agents in sequence.

**Example:**

```
/neuralrelay start "Add OAuth2 login with Google and GitHub providers"
```

**Output during execution:**

```
[NeuralRelay] Starting pipeline run: 20260322-143052
[NeuralRelay] Task: Add OAuth2 login with Google and GitHub providers
[NeuralRelay] Stages: requirements -> implementation -> testing -> review
```

Each agent runs in sequence. On completion, the process manager prints a summary with the verdict, quality score, token usage per stage, and the handoff chain location.

**Common issues:**
- Requires `neuralrelay.yaml` to exist. Run `/neuralrelay init` first if you see "NeuralRelay is not initialized."
- Long tasks (10-25 minutes) are normal. Use `/neuralrelay status` in another session to check progress.
- If an agent fails, completed handoffs are preserved. You can see partial results with `/neuralrelay chain`.

---

## /neuralrelay quick

**Syntax:** `/neuralrelay quick "task description"`

Runs a lightweight 2-stage pipeline: Dev Agent then Test Agent. Skips the BA requirements stage and the review stage. The dev agent extracts its own requirements from your task description.

**Example:**

```
/neuralrelay quick "Fix the null pointer exception in UserService.getProfile when email is null"
```

**Output during execution:**

```
[NeuralRelay] Quick mode -- 2-stage pipeline
[NeuralRelay] Chain: 20260322-150030
[NeuralRelay] Task: Fix the null pointer exception in UserService.getProfile when email is null
[NeuralRelay] Stages: implementation -> testing
[NeuralRelay] Estimated savings: 30-50% fewer tokens vs full pipeline
```

**When to use instead of start:**
- Bug fixes where you already know the requirements
- Small features with clear scope
- Refactoring with well-defined boundaries
- Tasks where your description is specific enough to serve as requirements

**When to use start instead:**
- New features, complex changes, unfamiliar codebase areas
- Anything touching security or payments
- Tasks that need independent requirements analysis or formal review

**Common issues:**
- Be specific in your task description — the dev agent extracts requirements from it directly. "Fix the login" is too vague; "fix the null pointer in UserService.getProfile when email is null" gives the agent clear direction.
- No rework loops in quick mode. If the tests reveal issues, you'll need to address them manually or run a full pipeline.

---

## /neuralrelay status

**Syntax:** `/neuralrelay status`

Shows the status of the most recent pipeline run. Displays each stage's completion status, token usage, and key metrics.

**Example output:**

```
[NeuralRelay] Pipeline Status -- Chain: 20260322-143052

Stage            Status      Tokens    Summary
-------------------------------------------------
Requirements     Done        --        Add OAuth2 login with Google and GitHub
Implementation   Done        45,200    9 files changed
Testing          Done        9,200     7/9 passing, 2 bugs
Review           Done        --        changes_requested (7/10)
-------------------------------------------------
Total Tokens: 27,500
Verdict: changes_requested
Quality Score: 7/10

Chain Quality: requirements=9/10  dev-handoff=8/10  test-coverage=8/10
```

**Stage status indicators:**
- Done — handoff file exists and is valid
- Active — agent is currently running (visible if you check from another session)
- Pending — stage hasn't started yet

**Common issues:**
- Shows only the most recent run. To see a specific run, use `/neuralrelay chain {chain-id}`.
- If no runs exist, suggests running `/neuralrelay start`.

---

## /neuralrelay chain

**Syntax:** `/neuralrelay chain [chain-id]`

Visualizes a handoff chain in human-readable format. Shows each pipeline stage with key metrics, bugs found, property verification results, and total cost estimate.

**Examples:**

```
/neuralrelay chain                    # Most recent chain
/neuralrelay chain 20260322-143052    # Specific chain by ID
```

**Output:** See the [Getting Started](getting-started.md#what-youll-see) guide for a full example of the chain visualization output.

The chain command reads all handoff JSON files in the chain directory, sorted by sequence prefix. It handles missing stages gracefully — quick mode chains show only the stages that ran, and incomplete runs show whatever data is available.

**Common issues:**
- If a handoff file contains invalid JSON, the command may show that stage as incomplete rather than crashing.
- Chain IDs are timestamps (YYYYMMDD-HHmmss). List available chains with `ls .neuralrelay/handoffs/`.

---

## /neuralrelay report

**Syntax:** `/neuralrelay report`

Generates cost and quality analytics across all completed pipeline runs. Shows token usage patterns per stage, average quality scores, common bug categories, and optimization recommendations.

**Example output (with 3+ completed runs):**

```
[NeuralRelay] Pipeline Analytics -- 5 completed runs

Token Usage by Stage
-----------------------------------------
Requirements:    avg 3,200 tokens  (8%)
Implementation:  avg 28,500 tokens (62%)
Testing:         avg 9,800 tokens  (21%)
Review:          avg 4,100 tokens  (9%)
-----------------------------------------
Average Total:   45,600 tokens/run

Quality Scores
-----------------------------------------
Average Quality:      7.2/10
Approval Rate:        60% (approved on first pass)
Avg Rework Cycles:    0.4

Most Common Bug Categories
-----------------------------------------
1. correctness: 8 occurrences
2. security: 3 occurrences
3. performance: 2 occurrences
```

The report also generates recommendations based on the data. For example, if the implementation stage uses over 50% of tokens, it suggests adding module boundaries to CLAUDE.md to reduce codebase exploration time.

**Common issues:**
- Requires at least one completed pipeline run (with a review-final handoff). Quick-mode runs without a review stage are not included in quality score analytics.
- More runs produce more useful analytics. The report is most valuable after 5+ pipeline runs.
