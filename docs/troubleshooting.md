# Troubleshooting

## Pipeline takes too long

**Typical cause:** Large codebase. Agents spend significant time reading files to understand the project.

**Solutions:**
- Add a CLAUDE.md to your project with key architectural decisions, module boundaries, and conventions. This gives agents a head start instead of exploring everything.
- Use quick mode (`/neuralrelay quick`) for simpler tasks — 2 stages instead of 4.
- Check if your `neuralrelay.yaml` uses Opus for stages where Sonnet would suffice. The BA and test stages usually work well with Sonnet.
- If a specific stage is slow, check whether the agent's `maxTurns` limit is being approached (visible in `/neuralrelay status`).

## Dev agent produces empty or vague coverage_gaps

**Typical cause:** Using Sonnet for the dev stage. Sonnet tends to produce vaguer coverage gaps than Opus.

**Solutions:**
- Set `model_preference: opus` for the implementation stage in `neuralrelay.yaml`. Opus is notably better at self-reflection about what it didn't test.
- If you must use Sonnet, the schema still enforces a minimum of 3 coverage_gaps. The quality may be lower but the structure is maintained.
- Check the dev-to-test handoff directly: `cat .neuralrelay/handoffs/{chain-id}/02-dev-agent.json | jq '.coverage_gaps'`. If the gaps are generic ("needs more testing"), this is a model quality issue, not a NeuralRelay bug.

## Test agent re-tests everything instead of focusing on gaps

**Typical cause:** The test agent didn't read the dev handoff, or the handoff was missing.

**Solutions:**
- Run `/neuralrelay chain` to verify the dev-to-test handoff was actually written. If it's missing, the dev agent failed before producing output.
- Check the dev handoff exists: `ls .neuralrelay/handoffs/{chain-id}/02-dev-agent.json`
- If the handoff exists but the test agent ignores it, this is a model behavior issue. The handoff protocol instructs the agent to follow priority order, but models don't always follow instructions perfectly. This is a known limitation.

## Review agent gives changes_requested on everything

**Typical cause:** The quality bar may be appropriate — check the specifics.

**Solutions:**
- Check the `quality_score` in the review handoff. Scores 4-6 result in changes_requested with specific issues.
- Read the `issues` array to see what the reviewer flagged. If all issues are "style" category, consider whether those matter for your project.
- If the reviewer is too strict for your needs, you can: remove the review stage from `neuralrelay.yaml`, or use quick mode which has no review.
- A score of 7+ results in approval. If you're consistently getting 4-6, the implementation quality may genuinely need improvement.

## neuralrelay.yaml references an agent that doesn't exist

**Error:** The process manager reports "Agent {name} not found" with a list of available agents.

**Solution:** Check that the agent name in `neuralrelay.yaml` matches the filename in `agents/` (without the `.md` extension). For example, `agent: security-agent` requires `agents/security-agent.md` to exist.

## Handoff validation fails

**Typical cause:** An agent produced JSON that doesn't match the schema — missing required fields, wrong types, or fewer than 3 coverage_gaps.

**Solutions:**
- The process manager retries once when validation fails (asks the agent to fix its output).
- If it fails twice, check the agent's output for malformed JSON. This is rare but can happen with complex tasks that push the agent close to its context limit.
- Read the raw handoff file to see what's wrong: `cat .neuralrelay/handoffs/{chain-id}/{NN}-{agent}.json | python -m json.tool`
- If a specific field is consistently missing, the agent instructions may need updating. Check the agent's markdown file in `agents/`.

## Quick mode dev agent doesn't understand the task

**Typical cause:** Vague task description. In quick mode, the dev agent extracts requirements directly from your description — there's no BA to clarify.

**Solution:** Be more specific in your `/neuralrelay quick` description.

- Too vague: `"fix the login"`
- Better: `"fix the null pointer exception in UserService.getProfile when the user has no email address"`
- Best: `"fix the null pointer in UserService.getProfile (src/services/user.ts:45) — getProfile calls user.email.toLowerCase() without null check, throws when email is null"`

The more context you provide, the better the dev agent performs without a BA stage.

## Rework loop runs twice and still fails review

**Typical cause:** The issues are too complex for targeted fixes, or there's a fundamental design problem.

**Solutions:**
- `max_rework_cycles` defaults to 2. After that, the pipeline stops and reports the current state.
- Read the review handoffs from each cycle to understand what keeps failing. The issues may require a different approach, not just fixes.
- Consider breaking the task into smaller pieces. If the reviewer keeps finding new issues, the task scope may be too large for a single pipeline run.
- You can increase `max_rework_cycles` in `neuralrelay.yaml`, but more than 3 cycles rarely helps — if 2 rounds of fixes didn't resolve it, the issue is usually architectural.

## No .neuralrelay/ directory after init

**Typical cause:** File system permissions or the init command failed silently.

**Solution:** Check that you have write permissions in the project directory. Try creating the directory manually: `mkdir -p .neuralrelay/handoffs`. Then verify `neuralrelay.yaml` was created.

## Chain visualization shows (not completed) for a stage that ran

**Typical cause:** The agent ran but failed before writing its handoff file, or the file was written with invalid JSON.

**Solutions:**
- List files in the chain directory: `ls .neuralrelay/handoffs/{chain-id}/`
- If the file is missing, the agent failed before completing. Check Claude Code's output for error messages.
- If the file exists but chain shows "not completed," the JSON may be malformed. Validate it: `python -m json.tool .neuralrelay/handoffs/{chain-id}/{file}.json`

## "No pipeline history found" but I know I ran pipelines

**Typical cause:** The handoff directory is in a different location or was cleaned up.

**Solutions:**
- Check `neuralrelay.yaml` for the `handoff_dir` setting — it may point somewhere other than `.neuralrelay/handoffs`.
- The default `.gitignore` entry excludes handoff data from git. If you cloned the repo on a new machine, the handoff history isn't there.
- Handoff data is local to your machine. It is not committed to the repository.
