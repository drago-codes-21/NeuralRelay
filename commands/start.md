---
name: neuralrelay start
description: Start a full NeuralRelay pipeline run with a task description
args: task_description
tools:
  - Read
  - Write
  - Bash
  - Glob
  - Agent
---

Start a NeuralRelay pipeline run. Takes a natural language task description and orchestrates the full agent pipeline from requirements through review.

## Usage

```
/neuralrelay start "Add OAuth2 login with Google and GitHub providers"
```

## Behavior

1. **Validate prerequisites:**
   - Check that `neuralrelay.yaml` exists in the project root
   - If missing: print "NeuralRelay is not initialized. Run /neuralrelay init first." and stop
   - Read and parse `neuralrelay.yaml` to get pipeline stages and settings

2. **Generate chain ID:**
   - Format: `YYYYMMDD-HHmmss` (e.g., `20260322-143052`)
   - This ID uniquely identifies the pipeline run

3. **Print pipeline start:**
   ```
   [NeuralRelay] Starting pipeline run: {chain-id}
   [NeuralRelay] Task: {task_description}
   [NeuralRelay] Stages: {stage1} → {stage2} → {stage3} → {stage4}
   ```

4. **Invoke the process manager agent:**
   - Spawn the `process-manager` agent with:
     - The task description
     - The chain ID
     - The handoff directory path: `.neuralrelay/handoffs/{chain-id}/`
     - The parsed pipeline configuration
   - The process manager handles all agent orchestration, handoff validation, rework loops, and summary reporting

5. **On completion:**
   - The process manager will print its own summary
   - Print: `[NeuralRelay] Handoff chain saved to: .neuralrelay/handoffs/{chain-id}/`
   - Print: `[NeuralRelay] Run /neuralrelay status for details or /neuralrelay report for analytics.`
