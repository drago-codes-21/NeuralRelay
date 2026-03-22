---
name: neuralrelay status
description: Show current pipeline status and token usage
tools:
  - Read
  - Glob
  - Bash
---

Show the status of the active or most recent NeuralRelay pipeline run.

## Behavior

1. **Find handoff chains:**
   - Look in `.neuralrelay/handoffs/` for chain directories
   - Sort by name (which sorts by date since chains use YYYYMMDD-HHmmss format)
   - Select the most recent chain

2. **If no chains exist:**
   ```
   No NeuralRelay pipeline runs found.
   Run /neuralrelay start "your task description" to begin.
   ```

3. **For the most recent chain, read available handoffs and display:**

   ```
   [NeuralRelay] Pipeline Status — Chain: {chain-id}

   Stage            Status      Tokens    Summary
   ─────────────────────────────────────────────────
   Requirements     ✓ Done      —         {task_summary}
   Implementation   ✓ Done      {n}       {files_changed count} files changed
   Testing          ✓ Done      {n}       {passing}/{total} passing, {bugs} bugs
   Review           ✓ Done      —         {verdict} ({quality_score}/10)
   ─────────────────────────────────────────────────
   Total Tokens: {total}
   Verdict: {verdict}
   Quality Score: {quality_score}/10
   ```

   - Read all `.json` files in the chain directory sorted by sequence prefix (01-, 02-, etc.), excluding `rework-*.json`
   - Identify each stage by the `handoff_type` field in the JSON
   - Mark stages as "✓ Done" if their handoff file exists, "● Active" if in progress, "○ Pending" if not started
   - Pull summary data from each handoff JSON

4. **If a rework cycle is in progress:**
   - Show rework cycle count: `Rework Cycle: {n}/{max}`
   - Show which stage is being re-run

5. **Show chain quality if review is complete:**
   ```
   Chain Quality: requirements={n}/10  dev-handoff={n}/10  test-coverage={n}/10
   ```
