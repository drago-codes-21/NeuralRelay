---
name: neuralrelay report
description: Generate cost and quality analytics across pipeline runs
tools:
  - Read
  - Glob
  - Bash
---

Analyze all NeuralRelay pipeline runs and generate cost and quality analytics.

## Behavior

1. **Find all handoff chains:**
   - Scan `.neuralrelay/handoffs/` for all chain directories
   - Read the review-final handoff from each completed chain

2. **If no completed chains exist:**
   ```
   No completed NeuralRelay pipeline runs found.

   The report command analyzes historical pipeline data to identify:
   - Token usage patterns per agent stage
   - Average quality scores over time
   - Most common bug categories
   - Cost optimization opportunities

   Run /neuralrelay start "your task" to create your first pipeline run.
   ```

3. **Generate the analytics report:**

   ```
   [NeuralRelay] Pipeline Analytics — {n} completed runs

   Token Usage by Stage
   ─────────────────────────────────────────
   Requirements:    avg {n} tokens  ({pct}%)
   Implementation:  avg {n} tokens  ({pct}%)
   Testing:         avg {n} tokens  ({pct}%)
   Review:          avg {n} tokens  ({pct}%)
   ─────────────────────────────────────────
   Average Total:   {n} tokens/run

   Quality Scores
   ─────────────────────────────────────────
   Average Quality:      {n}/10
   Approval Rate:        {n}% (approved on first pass)
   Avg Rework Cycles:    {n}

   Chain Quality Trends
   ─────────────────────────────────────────
   Requirements Clarity:     avg {n}/10
   Dev Handoff Completeness: avg {n}/10
   Test Coverage Adequacy:   avg {n}/10

   Most Common Bug Categories
   ─────────────────────────────────────────
   1. {category}: {count} occurrences
   2. {category}: {count} occurrences
   3. {category}: {count} occurrences

   Cost Hotspots & Recommendations
   ─────────────────────────────────────────
   ```

4. **Generate recommendations based on data:**
   - If implementation stage uses >50% of tokens: "Dev agent spends {pct}% of tokens — consider adding module boundaries and key patterns to CLAUDE.md to reduce codebase exploration time."
   - If rework rate is >30%: "High rework rate ({pct}%) — check if BA requirements are specific enough (avg clarity: {n}/10)."
   - If test coverage adequacy is <6: "Test coverage scores are low — dev agent may not be providing detailed enough coverage_gaps."
   - If a bug category dominates: "{category} bugs account for {pct}% of issues — consider adding a dedicated {category} check to the pipeline."
