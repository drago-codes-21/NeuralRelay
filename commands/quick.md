---
name: neuralrelay quick
description: Run a lightweight 2-stage pipeline (dev + test) for simple tasks
args: task_description
tools:
  - Read
  - Write
  - Bash
  - Glob
  - Agent
---

Run a lightweight NeuralRelay pipeline with only 2 stages: Dev Agent and Test Agent. Skips the BA requirements stage and the review stage for faster, cheaper execution on straightforward tasks.

## When to Use

- Bug fixes where you already know the requirements
- Small features with clear scope
- Refactoring with well-defined boundaries
- Tasks where you can describe the requirements yourself in the task description

Use `/neuralrelay start` instead for: new features, complex changes, unfamiliar codebase areas, anything touching security or payments.

## Usage

```
/neuralrelay quick "Fix the off-by-one in history command directory listing"
```

## Behavior

1. **Validate prerequisites:**
   - Check that `neuralrelay.yaml` exists (suggest `/neuralrelay init` if not)
   - Read settings from neuralrelay.yaml (max_rework_cycles, handoff_dir, etc.)

2. **Generate chain ID:**
   - Format: `YYYYMMDD-HHmmss` (same as full pipeline)

3. **Print pipeline start:**
   ```
   [NeuralRelay] Quick mode — 2-stage pipeline
   [NeuralRelay] Chain: {chain-id}
   [NeuralRelay] Task: {task_description}
   [NeuralRelay] Stages: implementation → testing
   [NeuralRelay] Estimated savings: 30-50% fewer tokens vs full pipeline
   ```

4. **Invoke the process manager agent** with mode set to "quick":
   - Quick mode passes the task description directly to the dev agent. No separate BA stage runs.
   - The dev agent extracts its own inline requirements from the task description and records them in the `inline_requirements` field of its handoff
   - The test agent reads only the dev-to-test handoff (no requirements handoff to cross-reference — skips Priority 3)
   - No review stage — the test-to-review handoff is the final artifact

5. **On completion:**
   - Print summary (simplified — no review verdict or quality score):
   ```
   [NeuralRelay] Quick Pipeline Complete
   ├── Task: {task_description}
   ├── Tests: {passing}/{total} passing
   ├── Bugs Found: {count}
   ├── Total Tokens: {total}
   ├── Stages:
   │   ├── Implementation: {tokens} tokens
   │   └── Testing: {tokens} tokens
   └── Saved: ~{pct}% vs full pipeline estimate
   ```
   - Print: `[NeuralRelay] For full review, run: /neuralrelay start "{task_description}"`
