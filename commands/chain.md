---
name: neuralrelay chain
description: Visualize a completed handoff chain in human-readable format
args: chain_id
tools:
  - Read
  - Glob
  - Bash
---

Visualize a NeuralRelay handoff chain as a human-readable summary. Shows each pipeline stage with key metrics, bugs found, and total cost estimate.

## Usage

```
/neuralrelay chain                    # Show most recent chain
/neuralrelay chain 20260322-143052    # Show specific chain
```

## Behavior

1. **Find the chain:**
   - If chain_id is provided: look for `.neuralrelay/handoffs/{chain_id}/`
   - If chain_id is omitted: find the most recent chain by sorting `.neuralrelay/handoffs/` directories reverse-alphabetically and selecting the first
   - If no chains exist: print "No pipeline runs found. Run /neuralrelay start to create one."

2. **Read all available handoffs** in the chain directory:
   - Read all `.json` files sorted by sequence prefix (01-, 02-, 03-, etc.), excluding `rework-*.json` files
   - Identify each handoff by its `handoff_type` field: "requirements", "dev_to_test", "test_to_review", "review_final"
   - Any stage may be missing (quick mode skips BA and review; incomplete runs may have gaps)

3. **Display the visualization:**

   ```
   Pipeline Run: {chain-id}
   Task: "{task_summary}"
   Status: {verdict_icon} {verdict or "Incomplete"}

   ┌─ Requirements (BA Agent) ─────────────────────────
   │  Complexity: {estimated_complexity}
   │  Criteria: {count} acceptance criteria defined
   │  Scope: {first 2 out_of_scope items, comma-separated}
   │  Modules: {count} affected
   │
   ├─ Implementation (Dev Agent) ──────────────────────
   │  Files: {created} created, {modified} modified
   │  Tests: {count} written, {passing} passing
   │  Coverage Gaps: {count} identified
   │  Properties: {count} stated ({high}H {medium}M {low}L)
   │  Tokens: {tokens_used:,}
   │
   ├─ Testing (Test Agent) ────────────────────────────
   │  Tests: {total} total, {passing} passing, {failing} failing
   │  Properties: {confirmed} confirmed, {falsified} FALSIFIED, {inconclusive} inconclusive
   │  Bugs: {bug_counts_by_severity}
   │  Focus: {recommended_focus_for_reviewer (truncated to 80 chars)}
   │  Tokens: {tokens_used:,}
   │
   ├─ Review (Review Agent) ───────────────────────────
   │  Verdict: {verdict}
   │  Quality: {quality_score}/10
   │  Issues: {count} ({by_severity})
   │  Chain Quality: req {n}/10, dev {n}/10, test {n}/10
   │
   └─ Total: {total_tokens:,} tokens | ~${estimated_cost} estimated cost
   ```

4. **Verdict icons:**
   - Approved: check mark
   - Changes Requested: warning
   - Needs Rework: cross
   - Incomplete: question mark

5. **Cost estimation:**
   - Estimate cost based on total tokens using approximate Claude pricing
   - Display as a rough estimate: "~$X.XX"

6. **Handle missing stages gracefully:**
   - If a handoff file is missing, show the stage header with "(not completed)" and skip the detail lines
   - If in quick mode (no requirements or review): show only the stages that ran
   - Always show whatever data is available — partial information is better than none
